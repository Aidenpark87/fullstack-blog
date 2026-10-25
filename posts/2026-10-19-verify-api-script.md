# One Verification Script Instead of Dozens of `curl` Commands — `verify:api`

> A concise overview: I replaced manual checks with a single command that runs 56 API checks against either a local or deployed server, and counts failures instead of relying on memory.

## Table of Contents

1. [When verification depends on memory](#1-when-verification-depends-on-memory)
2. [How much changed? Start with the numbers](#2-how-much-changed-start-with-the-numbers)
3. [What I built: one command, 56 checks](#3-what-i-built-one-command-56-checks)
4. [Switch between local and deployed servers with `--base`](#4-switch-between-local-and-deployed-servers-with---base)
5. [What does it check?](#5-what-does-it-check)
6. [Why count failures? Passing checks are not enough](#6-why-count-failures-passing-checks-are-not-enough)
7. [Why document it in the README?](#7-why-document-it-in-the-readme)
8. [Measured verification results](#8-measured-verification-results)
9. [Summary: three things to remember](#9-summary-three-things-to-remember)
10. [References](#10-references)

## 1. When verification depends on memory

Whenever I changed a feature, I used to send **dozens of requests by hand** to check it: sign in, open the post list, like a post, unlike it, write a comment, delete it, and so on. The whole process existed only as a **checklist in my head**.

Here is what could happen: **I might not notice that the like toggle had broken**, because that day I only checked the post list before committing. It was also easy to miss when a feature that had worked before broke later—because I had not checked it.

💡 **What you’ll learn:** How I turned “dozens of `curl` commands” into **one command** that works against both local and deployed servers, and why a verification script should count failures explicitly.

## 2. How much changed? Start with the numbers

The biggest change is that I now check **results instead of relying on memory**.

| Aspect | Before (manual) | After (script) |
|---|---|---|
| Checks | Only the cases I happened to remember that day | **All 56 checks**, every run |
| Time | Several minutes of manual work | One command |
| Deployment verification | Change URLs and repeat the manual checks | One `--base` option |
| Reproducing a check with a teammate | Explain the steps verbally | Share `pnpm verify:api` |
| Regression detection | Learn about it when a user reports it | **Catch it automatically before committing** |

After I created the script, it caught regressions in the like and comment features **twice**. If I had checked manually, I might have missed both.

## 3. What I built: one command, 56 checks

I added a command to the project’s root `package.json`:

```json
{
  "scripts": {
    "verify": "node ./scripts/verify-api.js",
    "verify:api": "node ./scripts/verify-api.js"
  }
}
```

```bash
# With the local server running
pnpm verify:api

# Example output
# ✅ 56 passed / 0 failed
```

The core idea is simple: **each check returns whether it passed or failed, and the script totals the results at the end.**

```js
// scripts/verify-api.js (simplified structure)
const BASE = process.argv.find((arg) => arg.startsWith('--base='))?.split('=')[1]
  ?? 'http://localhost:5001';

let passed = 0;
let failed = 0;

async function check(name, expectedStatus, request) {
  const response = await request();
  if (response.status === expectedStatus) {
    passed += 1;
    console.log(`  ✅ ${name}`);
  } else {
    failed += 1;
    console.log(`  ❌ ${name} (expected ${expectedStatus} · got ${response.status})`);
  }
}

// … checks …

console.log(`\n${passed} passed / ${failed} failed`);
process.exit(failed === 0 ? 0 : 1); // ★ A failure is reported through the exit code
```

**Why set the process exit code?** If the script finishes with exit code `0` even when checks fail, later automation—such as a deployment pipeline or a pre-commit hook—will not know anything went wrong. So the script returns exit code `1` if even one check fails. **That is what turns it from a report into a gate.**

## 4. Switch between local and deployed servers with `--base`

The feature I value most is being able to change the target server by changing **one address**.

```bash
# Local server (default)
pnpm verify:api

# Deployed server
pnpm verify:api --base=https://panda-market-backend-sprint9.vercel.app

# Verify an earlier deployment with the same script
pnpm verify:api --base=https://panda-market-backend.vercel.app
```

| Situation | Before | Now |
|---|---|---|
| Check right after deployment | Run `curl` against each URL one by one | One `--base` option |
| “It works locally, but not after deployment” | Spend hours tracing the cause | Run **the same checks** against production to narrow down the difference |
| Check for a regression in an earlier deployment | Nearly impossible | Change the URL and compare immediately |

**⚠️ Be careful when running this against a deployment.** Some checks **change production data** by creating, editing, deleting, or toggling likes. Running them against a deployment changes data on that server. I run deployment verification **once right after deployment** and use the local server afterward.

## 5. What does it check?

More important than the number “56” is that I included **cases that are supposed to fail**, not just the happy path.

### 5-1. Like toggle — verify the entire flow

| Step | Action | Expected status | What it checks |
|---:|---|---:|---|
| 1 | Add a like for the first time | 200 | Normal behavior |
| 2 | Add the same like again | 409 | Is duplicate prevention working? |
| 3 | Remove the like | 200 | Delete behavior |
| 4 | Remove a like that does not exist | 404 | Is “not found” handled? |

All four steps need to work for the like toggle to be correct. When checking by hand, it is easy to stop after step 1 and say, “Looks good.” **Step 2 (the duplicate `409`)** in particular often breaks during development, but manual checks rarely catch it.

### 5-2. Tokens and authentication — cases that should fail

| Scenario | Expected status | What it confirms |
|---|---:|---|
| Expired access token | 401 | Expiration is checked |
| Use a refresh token as an access token | 401 | Token types are distinguished |
| Forge the social-login `state` | 400 | Tampering is rejected |
| Send a request after logout | 401 | The cookie was properly cleared |

**Why test cases that should fail?** Security checks cover things that are invisible when everything appears to work. Duplicate prevention and forgery detection do not show up in the UI, so a broken check can go unnoticed. That is why I make those cases explicit.

### 5-3. Starting social login — check the redirect too

It is not enough to check whether the login button responds. I also check **where it redirects and whether the required parameters are included**.

| Check | Expected result |
|---|---|
| Start-login request | 302 + authorization URL |
| Security parameters | `state` and `code_challenge` are present |
| Missing configuration | 400 + a readable message for the user |

## 6. Why count failures? Passing checks are not enough

The script reports both sides, for example, `56 passed / 0 failed`. That may look like a small detail, but it is **an important design choice**.

| Output style | Problem or benefit |
|---|---|
| Show only 56 passed | You cannot detect missing checks; the script might report 56 even if it actually ran only 50. |
| Show passed and failed counts | **56 + 0 = 56**, so you can also confirm the total number of checks. |

I also gave every check a **name**. If the output only shows ✅, it is hard to tell what passed, and I may wonder, “Was this check even included?” Names make it easier to trace something like, **“Why was step 2 of the like toggle missing today?”**

## 7. Why document it in the README?

If I only create the script, **I may be the only person who knows how to use it**. So I also documented how to run it and what it checks in the README.

```bash
# 1) Start the local server
pnpm dev

# 2) Run verification
pnpm verify:api

# 3) Verify a deployment
pnpm verify:api --base=https://<deployment-address>
```

**An analogy:** A verification script is like **a pre-flight checklist**. It should not be known only to the pilot who wrote it; the whole crew should use the same checklist. That is why I wrote the instructions in the README.

Then, when a teammate first opens the project, they can **check for themselves in 30 seconds** whether it is working. They do not need to hear the steps explained verbally.

## 8. Measured verification results

| When | Result |
|---|---|
| First added | 51 passed / 0 failed |
| After adding social login | 53 passed / 0 failed (2 new checks) |
| After adding cookie sessions and automatic refresh | 56 passed / 0 failed |
| Deployed-server verification | Set `--base` → 51 passed / 0 failed |
| After later refactors and backfills | Confirmed 56/56 on every run |

## 9. Summary: three things to remember

📌 **Three things to remember**

1. **If verification lives in your head, a check you did not run that day is effectively not a check.** Put it in a script, and anyone can run the same 56 checks at any time.
2. **Check cases that should fail, not just normal cases.** Duplicate `409`s, forged-state `400`s, and expired-token `401`s are invisible in the UI, but can still break.
3. **`--base` is the key to remote verification.** You can run **the same checks** locally and against a deployment, making “it works locally, but not after deployment” easier to investigate.

One thing changed after I created this script: **I now run `pnpm verify:api` out of habit before committing.** Before, I would wonder, “Did I check that?” and rack my brain. That is exactly why I moved verification into code—I learned that **verification that depends on memory eventually becomes verification that never happens**.

## 10. References

- [MDN — Fetch API](https://developer.mozilla.org/en-US/docs/Web/API/Fetch_API)
- [MDN — HTTP response status codes](https://developer.mozilla.org/en-US/docs/Web/HTTP/Status)
- [npm Docs — Scripts](https://docs.npmjs.com/cli/v10/using-npm/scripts)
- [Node.js Docs — `process.exitCode`](https://nodejs.org/api/process.html#processexitcode)
- [Vercel Docs — CLI deploy](https://vercel.com/docs/cli/deploy)
