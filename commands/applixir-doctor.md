---
description: Audit an existing AppLixir rewarded-video integration (read-only) and report pass/fail with fixes
argument-hint: "[optional symptom, e.g. \"reward fires twice\"]"
---

Audit this project's AppLixir integration using the `applixir-rewarded-video` skill and its `references/` files. **This is read-only: don't edit any file until the user approves the fixes.**

First find the integration: search for `applixir`, `initializeAndOpenPlayer`, `preloadAd`, `new Application(`, `applixir-ad-container`, `PlayRewardedAd`, and any callback endpoint that reads `gameApiKey` / `secretKey` / `signature` / `tid`.

Then report each check as **PASS / FAIL / N/A** with `file:line` evidence:

| # | Check | Passes when |
|---|---|---|
| 1 | SDK version | Loads `https://cdn.applixir.com/applixir.app.v6.1.0.js`; no v2 APIs (`invokeApplixirVideoUnit`, `zoneId`/`devId`/`gameId`, `sdk2.1m`, `ApplixirWebGL.PlayVideo`) |
| 2 | User-triggered show | `initializeAndOpenPlayer` / `openPlayer` / `handle.show()` is reached only from a click/tap handler, never on load, timers, scene start, or inactivity |
| 3 | Status object | Reads `status.type`; never compares `status` to a string |
| 4 | Reward only on `complete` | Grants only on `status.type === "complete"`; **not** on `allAdsCompleted`, skip, close, error, or consent decline |
| 5 | All callbacks handled | Cleanup (hide container, re-enable button, resume game/audio) on `allAdsCompleted`, `skipped`/`skip`, `manuallyEnded`, `consentDeclined`, **and** `adErrorCallbackFn` |
| 6 | No-fill / dead-callback handling | A watchdog or equivalent recovers when no callback arrives (bad key, no-fill); a friendly message, no broken UI; SDK-missing (ad blocker) guarded |
| 7 | Client idempotency | Per-ad `rewarded` guard; single in-flight ad; button disabled while open; preload handles single-use |
| 8 | Server verification present | A callback endpoint credits the reward; the client doesn't credit persistent rewards by itself |
| 9 | Server idempotency + auth | Endpoint checks `signature` (MD5) or `secretKey` in constant time, requires `userId`, dedupes on `tid` with a unique constraint in the same transaction as the credit, decides the amount server-side; SDK passes `userId` |
| 10 | Consent | Existing TCF CMP loads before the SDK, or no CMP and the developer knows AppLixir's notice will show; no faked consent; GPP-only CMP flagged |
| 11 | Key handling | API key loaded from the project's config, not hard-coded in game logic; callback secret only on the server, never in client code or the repo |
| 12 | Domain / environment | Not relying on localhost; `ads.txt` present if the repo serves the site root; callback URL is https with no `?` |

Note: there's no AppLixir "test key" or test mode, so don't fail anything for "test vs production key". Check 11 covers keys.

After the table:
1. **Most likely cause** of the reported symptom, if one was given (see `references/troubleshooting.md`).
2. **Proposed fixes**, ordered by severity, each naming the file and the change.
3. Ask: "Want me to apply these?" and **wait**.

Symptom or extra context from the developer (may be empty):

$ARGUMENTS
