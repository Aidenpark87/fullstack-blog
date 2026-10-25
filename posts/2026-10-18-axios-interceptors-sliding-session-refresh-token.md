# [React/Node.js] Axios Interceptors and Sliding Sessions for Seamless Login (Refresh Tokens): A Complete Implementation

> **In a nutshell:** I implemented silent token renewal when an access token expires, and used `refreshPromise` to prevent duplicate refresh requests when multiple requests receive `401` responses at the same time.

## Table of Contents

1. [Introduction: Why not use only an access token?](#1-introduction-why-not-use-only-an-access-token)
2. [Backend design: Defend against two security pitfalls](#2-backend-design-defend-against-two-security-pitfalls)
3. [The frontend challenge: Concurrent `401` errors and race conditions](#3-the-frontend-challenge-concurrent-401-errors-and-race-conditions)
4. [The solution: Share a single Promise with `refreshPromise`](#4-the-solution-share-a-single-promise-with-refreshpromise)
5. [Measured verification](#5-measured-verification)
6. [What I learned while building the authentication system](#6-what-i-learned-while-building-the-authentication-system)

## 1. Introduction: Why not use only an access token?

When adding JWT (JSON Web Token)-based authentication to a web service, we often run into a trade-off between **security and user experience (UX)**.

- **Set a long expiration time for the access token (for example, 14 days):** It is convenient for users, but if the token is stolen, an attacker could use the account freely for 14 days, leading to a serious security incident.
- **Set a short expiration time for the access token (for example, 15 minutes to 1 hour):** This is safer, but a user might be forced to sign in again after 15 minutes of working on a post, or even lose their work. That is a terrible experience.

A standard way to address this is a **two-token system: a short-lived access token (1 hour) + a long-lived refresh token (14 days)**.

Normally, the client calls APIs with the access token. When the token expires and the server returns `401 Unauthorized`, the frontend obtains a new token in the background with the refresh token and resumes the original request.

The idea sounds simple, but **real-world implementation has major pitfalls: concurrency (race conditions) and preventing token theft**. In this post, I share how I implemented and verified the backend token design and concurrency control in the frontend’s Axios interceptor.

## 2. Backend design: Defend against two security pitfalls

### 2-1. Prevent token misuse: Validate the payload’s `type`

One common mistake for beginners is **not distinguishing the purpose (type) of each token**.

Access tokens and refresh tokens are both JWT strings. If the payload does not distinguish them, a malicious user could put a refresh token in an `Authorization: Bearer <token>` header and call a regular API. Since the signature is valid, `jwt.verify()` would accept it. That would make the one-hour access-token limit ineffective.

To prevent this, include a `type` in the payload when issuing each token:

```js
// backend/src/utils/token.js
const ACCESS_TOKEN_EXPIRES_IN = '1h';
const REFRESH_TOKEN_EXPIRES_IN = '14d';

export function issueTokens(user) {
  return {
    accessToken: sign(user, 'access', ACCESS_TOKEN_EXPIRES_IN),
    refreshToken: sign(user, 'refresh', REFRESH_TOKEN_EXPIRES_IN),
  };
}

function sign(user, type, expiresIn) {
  return jwt.sign(
    { userId: user.id, email: user.email, type }, // Include the token type in the payload
    process.env.JWT_SECRET,
    { expiresIn }
  );
}
```

The authentication middleware must then check that `decoded.type === 'access'`:

```js
// backend/src/middlewares/auth.middleware.js
export function requireAuth(req, res, next) {
  const token = req.headers.authorization?.replace(/^Bearer\s+/i, '');
  if (!token) throw new UnauthorizedException('Please sign in.');

  try {
    const decoded = jwt.verify(token, process.env.JWT_SECRET);
    // Immediately reject a refresh token used to access a regular endpoint
    if (decoded.type !== 'access') {
      throw new UnauthorizedException('This is not a valid access token.');
    }
    req.user = decoded;
    next();
  } catch (error) {
    throw new UnauthorizedException('The token has expired or is invalid.');
  }
}
```

### 2-2. Sliding sessions: Automatically extend an active session

Instead of a fixed-expiration approach that only issues a new access token, I chose a **Sliding Session** approach that also issues a new refresh token whenever the client requests a refresh.

```js
// backend/src/routes/auth.routes.js
authRouter.post('/refresh', async (req, res, next) => {
  try {
    const { refreshToken } = req.body;
    if (!refreshToken) throw new BadRequestException('A refresh token is required.');

    const decoded = jwt.verify(refreshToken, process.env.JWT_SECRET);
    if (decoded.type !== 'refresh') {
      throw new UnauthorizedException('This is not a valid refresh token.');
    }

    const user = await userRepository.findById(decoded.userId);
    if (!user) throw new UnauthorizedException('User not found.');

    // Issue both a new access token and a new refresh token (sliding session)
    const tokens = issueTokens(user);
    res.json(tokens);
  } catch (error) {
    next(error);
  }
});
```

This way, if a user visits the service at least once within 14 days, the expiration is extended by another 14 days. Active users are not unnecessarily signed out.

## 3. The frontend challenge: Concurrent `401` errors and race conditions

Now it is time to write the Axios interceptor on the frontend. The typical flow is:

```text
API request → receive 401 → call POST /auth/refresh → save new tokens → retry original request
```

But translating this flow directly into code without coordination can cause **a serious problem**.

### The thundering herd: What happens when several APIs receive `401` at once?

Imagine a user refreshing the dashboard. To render the page, several components send API requests almost simultaneously:

1. `GET /users/me` (my profile)
2. `GET /products` (product list)
3. `GET /notifications` (notifications)

What if the access token has expired? **All three requests receive a `401` at nearly the same time.**

Without any coordination in the interceptor, each request that receives a `401` will call **`POST /auth/refresh` separately**, three times in a row. If the token exchanges happen out of order, the server may reject an old refresh token that has already been used or invalidated. A user who was signed in could then be sent to the login page.

### An everyday analogy

Imagine three coworkers go to the office kitchen at the same time to make coffee and find the water dispenser empty.

- **Bad approach:** Each person runs out to buy a water bottle, and all three try to replace the dispenser at once. That is like duplicate requests colliding with one another.
- **Good approach:** The first person says, “I’ll get the water—hold on to your cups while I’m gone!” The other two wait and make their coffee once the dispenser is refilled.

## 4. The solution: Share a single Promise with `refreshPromise`

The key is to **cache the Promise for the refresh operation that is already in progress**.

### Complete Axios interceptor

```js
// frontend/src/lib/api/httpClient.js
import axios from 'axios';
import { getToken, setTokens, clearTokens } from '@/utils/token';

const httpClient = axios.create({
  baseURL: process.env.NEXT_PUBLIC_API_BASE_URL,
});

// 1. Request interceptor: attach the latest access token
httpClient.interceptors.request.use((config) => {
  const token = getToken('accessToken');
  if (token) {
    config.headers.Authorization = `Bearer ${token}`;
  }
  return config;
});

// 2. A single Promise cache for concurrency control
let refreshPromise = null;

// 3. Response interceptor: silently renew the token when a 401 occurs
httpClient.interceptors.response.use(
  (response) => response,
  async (error) => {
    const originalRequest = error.config;

    // Return the error if it is not a 401 or this request has already been retried
    if (error.response?.status !== 401 || originalRequest._retry) {
      return Promise.reject(error);
    }

    // If the refresh endpoint itself returns 401, the refresh token has expired too
    if (originalRequest.url?.includes('/auth/refresh')) {
      clearTokens();
      window.location.href = '/signin';
      return Promise.reject(error);
    }

    // Set a retry flag to prevent an infinite loop
    originalRequest._retry = true;

    try {
      // 🌟 Reuse an in-progress refresh Promise; create one only if needed
      if (!refreshPromise) {
        refreshPromise = (async () => {
          const refreshToken = getToken('refreshToken');
          if (!refreshToken) throw new Error('No refresh token');

          // Call the backend refresh API
          const response = await axios.post(
            `${process.env.NEXT_PUBLIC_API_BASE_URL}/auth/refresh`,
            { refreshToken }
          );

          const { accessToken, refreshToken: newRefreshToken } = response.data;
          setTokens({ accessToken, refreshToken: newRefreshToken });
          return accessToken;
        })().finally(() => {
          // Always clear the cache when the operation finishes, whether it succeeds or fails
          refreshPromise = null;
        });
      }

      // All waiting requests share the result of this single refresh call
      const newAccessToken = await refreshPromise;

      // Replace the original request's header with the new token, then retry it
      originalRequest.headers.Authorization = `Bearer ${newAccessToken}`;
      return httpClient(originalRequest);
    } catch (refreshError) {
      clearTokens();
      window.location.href = '/signin';
      return Promise.reject(refreshError);
    }
  }
);

export default httpClient;
```

### How the flow works

1. Request A receives a `401`, creates `refreshPromise`, and asks the server for a new token.
2. Requests B and C receive `401` at almost the same time. They skip `if (!refreshPromise)` and `await` the **same `refreshPromise`** that A created.
3. When the refresh succeeds, A, B, and C all receive the new token and retry their own original requests.
4. The `finally` block resets `refreshPromise` to `null`, so it is ready for the next token expiration.

## 5. Measured verification

I tested the behavior against four scenarios:

| Test scenario | Expected behavior | Measured result |
|---|---|---|
| Call `/users/me` with an expired access token | Receive `401` → refresh in the background with `/auth/refresh` → original request succeeds with `200` | ✅ The profile appeared normally without a visible interruption. |
| Put a refresh token where an access token belongs | The middleware rejects it after checking `decoded.type` | ✅ Returned `401` with “This is not a valid access token.” |
| Several API requests receive `401` at once | Check the number of `/auth/refresh` calls in the browser Network tab | ✅ **Exactly one call**; all other requests recovered successfully. |
| The refresh token has also expired | Clear local tokens and safely navigate to the login page | ✅ Tokens were cleared and the browser redirected to `/signin`. |

## 6. What I learned while building the authentication system

JWT authentication does not end with issuing a token and putting it in `localStorage`.

1. **Tokens need clearly defined roles (`type`).** A valid signature does not make every token interchangeable; stating its purpose in the payload helps prevent security incidents.
2. **In asynchronous systems, always consider concurrency.** Code that looks perfect in a single-request test can hit a race condition as soon as a real browser sends several API requests at once.
3. **A Promise can be passed around and cached like a value.** Beyond simply using `async`/`await`, storing a Promise object in a variable lets multiple executions share it. The `refreshPromise` pattern is a powerful way to control network bottlenecks and duplicate requests.

I hope this helps anyone dealing with a similar problem.
