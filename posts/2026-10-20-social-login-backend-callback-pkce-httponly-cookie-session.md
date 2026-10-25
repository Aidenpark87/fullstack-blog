# Don’t Send the Authorization Code to the Frontend — Backend Callback, PKCE, and an httpOnly Cookie Session

> English translation of [the original Tistory post](https://aiden.tistory.com/57), published on October 20, 2026. This is a review draft; it has not been published to GitHub.

## Table of Contents

1. [Why shouldn’t the login code reach the frontend?](#1-why-shouldnt-the-login-code-reach-the-frontend)
2. [My first attempt: receiving `?code=` on the frontend](#2-my-first-attempt-receiving-code-on-the-frontend)
3. [OAuth 2.1: What are `state` and PKCE?](#3-oauth-21-what-are-state-and-pkce)
4. [Architecture: Let the backend handle the whole flow](#4-architecture-let-the-backend-handle-the-whole-flow)
5. [Key code 1 — Start login (`/login`)](#5-key-code-1--start-login-login)
6. [Key code 2 — Receive the callback (`/callback`)](#6-key-code-2--receive-the-callback-callback)
7. [Key code 3 — httpOnly cookie session and automatic refresh](#7-key-code-3--httponly-cookie-session-and-automatic-refresh)
8. [Account-linking policy: What if the email already exists?](#8-account-linking-policy-what-if-the-email-already-exists)
9. [Verification results](#9-verification-results)
10. [Four real-world pitfalls](#10-four-real-world-pitfalls)
11. [Summary: Three things to remember](#11-summary-three-things-to-remember)
12. [References](#12-references)

## 1. Why shouldn’t the login code reach the frontend?

When adding Google or Kakao login, the first approach many of us try is: **“Let the frontend receive the code returned by Google and send it to our server.”** That’s how I started, too—and it worked.

But when I reviewed the design just before putting it into production, I found three things that made me uneasy:

1. **The authorization code passes through the browser’s address bar and JavaScript memory.** Once exposed, an attacker may be able to try logging in with it.
2. **There was effectively no `state` validation.** We had skipped the check that asks, “Is this the login request I initiated?”
3. **The token lives in `localStorage`.** A single XSS vulnerability can expose all of it.

So I changed the architecture entirely. **The backend now handles everything from generating the authorization URL to receiving the callback, while the browser only carries cookies back and forth.**

💡 **What you’ll learn:** How to build a social-login flow handled entirely by the backend (**`/login` → callback → cookies**) and implement the two security keys that protect it: **a signed `state` and PKCE**.

## 2. My first attempt: receiving `?code=` on the frontend

This was the flow before the change. Looking at it now, you might wonder why the frontend needed to receive the code at all. I didn’t understand that at the time, either.

| Step | Previous flow (through the frontend) | Risk |
|---:|---|---|
| 1 | The frontend navigates to Google’s authorization URL. | — |
| 2 | Google redirects to the **frontend** at `/signin?code=...`. | **The code is exposed in the address bar.** |
| 3 | The frontend sends the code to `POST /api/auth/google`. | JavaScript handles the code. |
| 4 | The backend asks Google to exchange the code for tokens. | — |
| 5 | The backend returns a JWT in the response body, and the frontend stores it in `localStorage`. | **XSS can expose everything.** |
| 6 | Every subsequent request sends it in an `Authorization: Bearer` header. | The token is exposed on every request. |

We weren’t checking `state` at all. In other words, we had no way to confirm, “Did I actually initiate this login request?”

## 3. OAuth 2.1: What are `state` and PKCE?

### 3-1. `state` — A seal that prevents forged consent

`state` is a **random value** we create when starting the login process. When the callback returns, we check that the same value came back. Put simply: **“Return the seal I gave you.”**

If an attacker sends a victim through the attacker’s own link, Google returns the **attacker-created `state`**. Because it doesn’t match the seal we created, we reject it.

### 3-2. PKCE — A paired key that makes an intercepted code unusable

The name sounds difficult, but the idea is simple:

1. The backend creates a **random string (`code_verifier`)**.
2. It hashes that value with SHA-256 and sends the result (**`code_challenge`**) to Google in the authorization URL.
3. Google binds the code it issues to that challenge.
4. In the callback, the backend sends the **original `code_verifier`**. Google checks whether its hash matches before returning the tokens.

Even if someone intercepts the code, **Google won’t issue tokens without its matching verifier**.

**An analogy:** `state` is **a seal secretly placed on a consent form**, while PKCE is **a key that matches a note**. Someone might steal the note (the code), but they can’t use it without the matching key (the verifier), which never left our server.

### 3-3. Why is this becoming the standard?

The OAuth 2.1 draft recommends PKCE **for all clients**, not only SPAs and mobile apps. The authorization-code flow becomes the only recommended flow, and the implicit grant is being deprecated. In other words, the pattern where `access_token=` appears in the address bar has been out of favor in the standards for a long time.

## 4. Architecture: Let the backend handle the whole flow

Here is the final architecture. The key point is that **the arrows never pass through the frontend**.

| Route or component | Owner | Responsibility |
|---|---|---|
| `GET /api/auth/social/:provider/login` | Backend | Create signed `state` + PKCE → transaction cookies → **302 redirect to Google** |
| `GET /api/auth/social/callback/:provider` | Backend | Validate → exchange tokens → validate profile → resolve user → **set auth cookies** → 302 redirect to the frontend |
| `POST /api/auth/logout` | Backend | Clear authentication cookies |
| Global request middleware | Backend | Authenticate cookies + **automatically refresh expired access tokens** |
| `useSocialLogin()` | Frontend | **Only navigate** to the login URL (zero code handling) |

All that remains on the frontend is: **“When the user clicks the login button, navigate to the server’s login URL.”** There is no code, token, or refresh logic on the frontend.

### 4-1. The `SocialAccount` table added to the database

One user may connect both Google and Kakao accounts, so we need a table to represent those connections. The combination of **provider + provider ID** is unique worldwide, so I added a unique constraint.

```js
// src/prisma/contract.prisma
model SocialAccount {
  id         Int    @id @default(autoincrement())
  provider   String   // 'google' | 'kakao'
  providerId String   // User identifier issued by that provider
  userId     Int
  createdAt  TimestampString(3) @default(now())

  @@unique([provider, providerId])   // Guarantees uniqueness, e.g. (google, 10317…)
  @@map("SocialAccount")
}
```

**⚠️ A quick note — why use a synthetic email?**  Kakao requires separate business-app approval to obtain the `account_email` permission. So instead of an email, I use a synthetic value such as `{provider}_{hash}@social.local`. The important point is that **the provider and provider ID are always available in full**, so identification is not affected.

## 5. Key code 1 — Start login (`/login`)

This is the heart of the article. The whole idea is to **sign the `state` and include the signature**.

```js
// src/providers/oauth-state-provider.js
import { createHmac, createHash, randomBytes, timingSafeEqual } from 'node:crypto';

export class OauthStateProvider {
  // code_verifier is 32 random bytes in base64url; challenge is its SHA-256 hash
  #randomBytes(size) {
    return randomBytes(size).toString('base64url');
  }

  #sign(encoded) {
    return createHmac('sha256', process.env.OAUTH_STATE_SECRET)
      .update(encoded)
      .digest('base64url');
  }

  createState({ provider, next = '/' }) {
    const codeVerifier  = this.#randomBytes(32);
    const codeChallenge = createHash('sha256').update(codeVerifier).digest('base64url');

    const payload = {
      provider,
      next: this.normalizeNextPath(next),   // Prevents open redirects
      transactionNonce: this.#randomBytes(16),
      codeChallenge,
      issuedAt: Math.floor(Date.now() / 1000),
    };

    // Encode the payload as base64url, then append an HMAC signature
    const encoded = Buffer.from(JSON.stringify(payload)).toString('base64url');
    return {
      state: `${encoded}.${this.#sign(encoded)}`,   // ← “A sealed stamp”
      transactionNonce: payload.transactionNonce,
      codeVerifier,                                 // ← Stored in a cookie (server only)
      codeChallenge,
    };
  }

  // Prevents open redirects: ignore absolute URLs and paths beginning with //; use '/' instead
  normalizeNextPath(next) {
    if (typeof next !== 'string') return '/';
    if (!next.startsWith('/')) return '/';
    if (next.startsWith('//')) return '/';
    return next;
  }
}
```

The controller uses this to create a **transaction cookie** and send a 302 redirect to Google. I restricted the cookie’s path to the callback route so that **it is not sent with requests to other pages at all**.

```js
// src/controllers/auth/social-controller.js
async startLogin(req, res) {
  const { provider } = req.params;
  const { next } = req.query;

  const { state, transactionNonce, codeVerifier, codeChallenge } =
    this.#oauthStateProvider.createState({ provider, next });

  // A transaction cookie sent only to the callback route (expires in 10 minutes)
  this.#cookieProvider.setOAuthTransactionCookies(res, {
    state, transactionNonce, codeVerifier, provider,
  });

  const url = new URL(`https://accounts.google.com/o/oauth2/v2/auth`);
  url.searchParams.set('client_id',     this.#config.GOOGLE_CLIENT_ID);
  url.searchParams.set('redirect_uri',  this.#config.GOOGLE_REDIRECT_URI);
  url.searchParams.set('response_type', 'code');
  url.searchParams.set('scope',         'openid email profile');
  url.searchParams.set('state',         state);
  url.searchParams.set('code_challenge',        codeChallenge);
  url.searchParams.set('code_challenge_method', 'S256');
  url.searchParams.set('prompt',        'select_account');   // Force account selection

  res.redirect(302, url.toString());
}
```

## 6. Key code 2 — Receive the callback (`/callback`)

When Google returns the code and `state` to our callback, we validate them **in order**. Changing the order can be risky.

| Order | What to validate | If missing or invalid |
|---:|---|---|
| 1 | Is the transaction cookie present? | → `400 invalid_request` |
| 2 | Is the `state` **HMAC signature** valid? | → `400` (forged) |
| 3 | Does the nonce (single-use value) match? | → `400` (prevents reuse) |
| 4 | Was `issuedAt` created within the last **10 minutes**? | → `400` (expired) |
| 5 | Is the provider one we support (`google`/`kakao`)? | → `400` |
| 6 | Is **`code_verifier`** sent along when exchanging the code? | → Token rejected (PKCE) |
| 7 | Is the profile validated with a **Zod schema**? | → A clear rejection instead of a `500` |

```js
// src/controllers/auth/social-controller.js (callback — abbreviated)
async handleCallback(req, res) {
  const tx = this.#cookieProvider.readOAuthTransactionCookies(req);

  try {
    if (!tx) throw this.#exceptions.badRequest('INVALID_REQUEST');
    if (!this.#oauthStateProvider.verifySignedState(req.query.state, tx))
      throw this.#exceptions.badRequest('INVALID_REQUEST');

    // ① Exchange the code + code_verifier for tokens  ② Validate the profile with Zod
    const profile = await this.#socialAuthService.fetchProfile({
      provider: tx.provider,
      code: req.query.code,
      codeVerifier: tx.codeVerifier,
    });

    // ③ Resolve the user (create one if missing; connect the account if one exists)
    const { user } = await this.#socialAuthService.resolveUser(profile);

    // ④ httpOnly authentication cookies
    const tokens = this.#authService.issueTokens(user);
    this.#cookieProvider.setAuthCookies(res, tokens);

    this.#cookieProvider.clearOAuthTransactionCookies(res);
    res.redirect(302, `${this.#config.CLIENT_BASE_URL}${tx.next ?? '/'}`);
  } catch (error) {
    // Redirect so callback failures can be shown in a frontend modal (never return raw JSON)
    const code = this.#toClientErrorCode(error);   // 409→link_required · 401→auth_failed
    res.redirect(302, `${this.#config.CLIENT_BASE_URL}/signin?error=${code}`);
  }
}
```

**Why redirect to `/signin?error=` with a 302?**  The callback is opened directly by the browser’s address bar. If we return JSON here, the user sees a block of text such as `{"success":false,...}`. **Redirecting to the login screen and showing a modal feels much more natural.** I mapped 409 to “This account needs to be linked,” 401 to “Authentication failed,” and 400 to “Your login request has expired.”

## 7. Key code 3 — httpOnly cookie session and automatic refresh

### 7-1. Put tokens in cookies, not in the response body

`localStorage` and cookies have completely different security boundaries.

| Comparison | `localStorage` | httpOnly cookie |
|---|---|---|
| Can JavaScript read it? | ✅ Yes | ❌ No (blocked by the browser) |
| Can XSS steal it? | Stolen immediately | Safer because it cannot be read |
| Sent automatically? | ❌ No (attach it to every header) | ✅ Yes (included with requests) |

```js
// src/providers/cookie-provider.js
const ACCESS_MAX_AGE = 15 * 60;        // 15 minutes
const REFRESH_MAX_AGE = 7 * 24 * 3600; // 7 days

setAuthCookies(res, { accessToken, refreshToken }) {
  res.cookie('accessToken', accessToken, {
    httpOnly: true, secure: true, sameSite: this.#sameSite(),
    path: '/', maxAge: ACCESS_MAX_AGE,
  });
  res.cookie('refreshToken', refreshToken, {
    httpOnly: true, secure: true, sameSite: this.#sameSite(),
    path: '/', maxAge: REFRESH_MAX_AGE,
  });
}

// Supports cross-site deployments — covered in detail in Part 2
#sameSite() {
  return process.env.NODE_ENV === 'production' ? 'none' : 'lax';
}
```

### 7-2. Automatic refresh — keep the user signed in without interrupting them

An access token lasts 15 minutes, so it will eventually expire while a user is working. At that point, **the backend middleware refreshes it automatically without involving the frontend**.

```js
// src/middlewares/auth.middleware.js
async resolveUser(req, res) {
  const { accessToken, refreshToken } = req.cookies ?? {};

  // ① If the access token is valid, use it right away
  if (accessToken) {
    const payload = this.#tokenProvider.verifyAccessToken(accessToken);
    if (payload) {
      const user = await this.#userRepository.findById(Number(payload.sub));
      if (user) return user;
    }
  }

  // ② If access has expired but refresh is still valid → automatically issue new tokens
  if (!refreshToken) return null;
  const { user, tokens } = await this.#authService.refreshTokens(refreshToken);
  this.#cookieProvider.setAuthCookies(res, tokens);   // Replace them with new cookies
  return user;
}
```

**Measured behavior so far:** With the `accessToken` cookie removed, I called `GET /users/me`. The response was **200**, and a **new `accessToken` cookie** came back with it. The user stays signed in without noticing anything. A forged refresh cookie is handled with **401 + cookie deletion**.

## 8. Account-linking policy: What if the email already exists?

This was where I actually encountered a 409 error in practice. We need a policy for **when a social account is not yet linked to an existing account with the same email**.

| Policy | Behavior | Pros and cons |
|---|---|---|
| **Automatic linking** | If the email matches, log in and link immediately. | Convenient, but **an attacker who can spoof an email could take over the existing account**. |
| **409 + require linking** | Show “This email is already registered” → sign in to the existing account, then link it. | Requires one more login, but **confirms ownership**. |

I chose **409 + require linking**. The policy worked as intended: the 409 appeared because the account had been created earlier with email login and had no social-account row.

**⚠️ A real-world pitfall — accounts with random passwords**  The account I had created earlier using a different approach had a **random password**, so the “sign in to the existing account and link it” path was blocked. I ended up **deleting that test account** and creating a new one through social login. In a local development database, that is often the quickest approach. For a **production database**, however, always check the foreign-key (FK) relationships before cleaning up data. I checked the product owner, likes, and linking tables together before deleting the account.

## 9. Verification results

I verified the behavior with measured results rather than relying on the impression that it “seemed to work.”

| # | Verification | Result |
|---:|---|---|
| 1 | Start social login (Google) | 302 + `code_challenge` in the authorization URL + two cookie types |
| 2 | Unsupported provider (`naver`) | Clear `400` response |
| 3 | Forged/expired `state` | `400` → `/signin?error=invalid_request` |
| 4 | Read cookie session | `200` + user returned |
| 5 | Read after logout | `204` → `401` |
| 6 | **Automatic refresh** (only the refresh token remains) | `200` + new cookies issued |
| 7 | Forged refresh cookie | `401` + cookie deleted |
| 8 | CORS + credentials | Confirmed `Access-Control-Allow-Credentials: true` |
| 9 | Real browser login (Google) | Confirmed new user + account-link row created |
| 10 | Unit tests (`state` tampering, expiration, path normalization) | 4/4 passed |

## 10. Four real-world pitfalls

### Pitfall 1 — Register the redirect URI in the right place in the console

When registering the redirect URI in the Google console, go to the **“Credentials” tab**, not the screen you might expect. The URI must point to our callback route.

```text
// If registered incorrectly → redirect_uri_mismatch
https://your-backend-domain/api/auth/social/callback/google
```

### Pitfall 2 — nodemon may not detect dotfiles

I put the backend environment variables in `env/.env.development` and ran nodemon. Even after changing the file, **login still returned 400**. The cause was that **nodemon does not watch changes to files that begin with a dot** by default.

**How to diagnose it:** Compare the process start time with the environment file’s modification time (`mtime`). If the file is newer than the server process, **restart the server manually**.

### Pitfall 3 — Kakao’s `KOE010 bad client credentials`

This error occurs if you issue or reissue a Kakao client secret but keep using the **old value** or copy a secret from a different app. What makes it confusing is that it is completely different from an invalid authorization code.

| Error code | Meaning | What to check |
|---|---|---|
| `KOE010` | Credential error | Are the REST key and secret correct? |
| `KOE320` | Authorization code not found | Credentials are valid · **the code is the problem** |

To distinguish them, I **called the token endpoint directly with a dummy code**. If the result is `KOE010`, the key is wrong; if it is `KOE320`, the key is valid and only the code flow is the problem. That one check cut the search space in half.

### Pitfall 4 — Kakao returns the fallback nickname

The consent screen **does not appear again for an account that is already linked**. So even if you add `profile_nickname` to the requested fields later, no nickname is returned, and the code uses a fallback such as “Panda.”

The solution is to request **`scope=profile_nickname`** to prompt for consent again. However, Kakao’s policy marks this as **required consent**; there is no option to make it optional. If the user cancels consent, Kakao returns `access_denied`, so I separated that case from authentication failure with a dedicated `consent_cancelled` code.

```js
// Distinguish consent cancellation from authentication failure
const toClientErrorCode = (error) => {
  if (error.oauthError === 'access_denied') return 'consent_cancelled';
  if (error.statusCode === 409) return 'link_required';
  if (error.statusCode === 401) return 'auth_failed';
  return 'invalid_request';
};
```

## 11. Summary: Three things to remember

📌 **Three things to remember**

1. **Keep the authorization code out of the browser.** If both `/login` and `/callback` are handled by the backend, the code never has to pass through the browser.
2. **`state` is not just a “validation string”; it is a signed string.** Adding an HMAC lets us mathematically prevent forgery.
3. **Put tokens in httpOnly cookies, not in the response body.** If the frontend never needs to handle tokens, the XSS exposure area is greatly reduced.

When I first used the approach where the frontend receives the code, I thought, “It works, so it must be right.” This project taught me that security should be judged **not by whether something works, but by where the sensitive data is exposed**. I added only a few lines and moved responsibility one layer up, but the architecture changed completely.

## 12. References

- [OAuth 2.0 for Browser-Based Applications](https://datatracker.ietf.org/doc/html/draft-ietf-oauth-browser-based-apps) — Security recommendations for browser-based applications
- [RFC 7636 — Proof Key for Code Exchange (PKCE)](https://datatracker.ietf.org/doc/html/rfc7636)
- [RFC 6749 §10.12 — Cross-Site Request Forgery](https://datatracker.ietf.org/doc/html/rfc6749#section-10.12) — Basis for `state`
- [Google Identity — OAuth 2.0 for Web Server Applications](https://developers.google.com/identity/protocols/oauth2/web-server)
- [Kakao Login REST API](https://developers.kakao.com/docs/latest/ko/kakaologin/rest-api)
- [OWASP — Cross-Site Request Forgery](https://owasp.org/www-community/attacks/csrf)
