---
name: applixir-rewarded-video
description: Integrate, audit or debug AppLixir rewarded video ads in a browser game or web app — HTML5/JavaScript, Phaser, PixiJS, Three.js, Cocos web, React web, or Unity WebGL (Unity Web). Use when someone asks to add rewarded ads or rewarded video, "watch an ad for coins/lives", monetize a browser/web/HTML5/WebGL game with ads, or verify ad rewards server-side. Always use it for ANY question that mentions AppLixir — including whether AppLixir works in a desktop (Electron/CEF/Steam) or native mobile app — so the answer reflects what AppLixir actually supports. Not for native iOS/Android ad SDK work that doesn't involve AppLixir (AdMob, Unity Ads, AppLovin).
---

# AppLixir rewarded video

AppLixir is a web-first rewarded video SDK: one CDN script, a player overlay, and a
callback that says when the player finished watching. Your job is to produce an
integration where:

- the player loads;
- an ad shows **only on a user action**;
- the reward is granted **once**, **only on completion**;
- the reward is **verified server-side**;
- consent is handled;
- no-fill and errors fail gracefully.

**Facts come only from the reference files in `references/`.** They're built from
AppLixir's integration docs and SDK. Never invent API names, options, events or
endpoints. If something isn't in the references, say so; don't guess.

**Never generate deprecated v2 APIs:** `invokeApplixirVideoUnit`, `zoneId`/`devId`/`gameId`,
`applixir.sdk2.1m.js`, string statuses like `"ad-watched"`, or the non-existent Unity
`ApplixirWebGL.PlayVideo`.

## 1. Scope check

Is this a **browser** context: a web page, HTML5 canvas game, WebGL build, or React web
app, served from a website?

- **Native iOS / Android / Unity mobile / Godot native**: stop. Say AppLixir is web-first and doesn't have a native mobile SDK, and that mobile ad networks are the usual fit for native apps. Don't write integration code. *Exception:* a React Native app can show a hosted https ad page in a WebView. Mention it only if the user is on React Native, and point to `applixir-integration/examples/react-native`.
- **Desktop apps** (Electron, CEF, NW.js, Steam/desktop builds): AppLixir supports browser contexts only, and desktop applications are out of scope. Don't write Electron/CEF integration code. If the game also has a **web version** on its own domain, offer to integrate there instead. Loading the SDK from a `file://` page fails anyway: origin `null` gives no consent and `AdError 303: No Ads`.
- **Server-only or non-game requests** (e.g. "build me an ad server"): out of scope.

## 2. Account and eligibility

Ask, if the conversation hasn't answered it already:

1. **Do you have an AppLixir account and an API key for this site?**
   - **No:** they need one first. Sign up at https://client.applixir.com/register, add the site under **Sites** with platform **Web**, and copy the API key. You can still scaffold the integration with a placeholder key read from config, but tell them ads won't serve until the site is registered.
2. **Enough volume?** AppLixir works with publishers serving at least **100,000 rewarded-ad impressions a month**.
   - If they give impressions, use that. If they give daily players (DAU), estimate: **DAU × ads a player would watch per day × 30**. Take the ads-per-day figure from them; if they haven't said, ask rather than assume.
   - **Clearly under 100K/month** (e.g. 150 players watching about one ad a day ≈ 4,500/month): explain the threshold and suggest they sign up at https://client.applixir.com/register so AppLixir can review the site as traffic grows. **Don't do the full integration.** Offer a brief summary of what it will involve (script tag, button, callback, server endpoint) so they can plan.
   - Unknown, borderline, or above: continue, and say AppLixir confirms eligibility when it reviews the site.

Never ask for, print, or hard-code the API key or callback secret in source.
Load them through the project's existing config/secrets pattern. The API key is client-side and ends up in
the browser, which is expected. The callback secret must stay server-side.

## 3. Detect the engine

Inspect the project before proposing anything:

| Signal | Engine | Reference |
|---|---|---|
| `index.html` + plain JS, or `phaser`/`pixi.js`/`three`/`cocos` in `package.json` | HTML5 / canvas | `references/html5-js.md` |
| `react` / `next` / `vite` + JSX/TSX | React web | `references/html5-js.md` § React |
| `Assets/`, `ProjectSettings/`, `*.unity`, `Build/*.loader.js`, `*.jslib`, `Assets/WebGLTemplates/` | Unity WebGL | `references/unity-webgl.md` |
| `electron` in `package.json`, `main.js` with `BrowserWindow`, CEF/CefSharp, NW.js | Desktop app: **out of scope** (step 1) | — |

Also look for:
- **A backend**: `server/`, `api/`, Express/Fastify/Flask/FastAPI/Django/ASP.NET/Laravel, serverless functions, Firebase/Supabase. Its presence decides step 6.
- **An existing CMP**: see the detection list in `references/consent-tcf.md`.
- **An existing AppLixir integration**: if you find one, audit it with the `/applixir-doctor` checks before changing it.
- **Where rewards are granted today**: currency, lives, shop. Where would a "Watch ad" button naturally go?

Load `references/api-reference.md` plus the engine's reference file.

## 4. Plan, then confirm

Before editing anything, present a short plan:

- **Files** to create or change, by path.
- **Trigger:** where the button goes and what it offers (e.g. "+3 lives on the game-over screen"). It must be an explicit, optional player choice.
- **Reward:** what is granted, and how it's verified (server callback endpoint path, or "no backend: see options").
- **Consent:** existing CMP found (name), or "the SDK will show AppLixir's notice".
- **Config:** where the API key, player id and callback secret will live.

**Wait for the user to confirm.** If they invoked this with specific instructions
(e.g. "Unity, put the button on the shop screen"), fold them into the plan, but
still confirm before editing.

## 5. Implement

Follow the engine reference exactly. Non-negotiable in every engine:

1. **Load** `https://cdn.applixir.com/applixir.app.v6.1.0.js` (pinned) and render `<div id="applixir-ad-container">`, kept in the DOM and toggled with `display`.
2. **Initialize** with `apiKey`, `injectionElementId`, `adStatusCallbackFn`, `adErrorCallbackFn`, and `userId` (the player's id, needed by the server callback).
3. **Show only from a user gesture**: the click/tap handler calls `initializeAndOpenPlayer(options)`, or `handle.show()` for a preloaded ad. Never on page load, timers, level start, or inactivity.
4. **Read `status.type`.** The callback receives an object, never a string.
5. **Reward only on `status.type === "complete"`, once per ad.** Use a per-ad `rewarded` flag. `allAdsCompleted` is **not** a reward: it fires after every ad and on no-fill.
6. **Every other ending cleans up.** That means `allAdsCompleted`, `skipped`/`skip`, `manuallyEnded`, `consentDeclined`, `consentUnavailable`, `adErrorCallbackFn`, plus a **watchdog timeout**, since a bad key or no-fill can produce no callback at all. Cleanup hides the container, re-enables the button, resumes the game loop and audio, and shows a short message when no reward was earned.
7. **One ad at a time**: an `inFlight` guard and a disabled button.
8. **Degrade gracefully** when the SDK is missing (ad blocker, CSP): no crash, a short message.
9. **Optional fast path:** `preloadAd(options)` on a high-intent moment, `handle.show()` in the click, always with the `initializeAndOpenPlayer` fallback. The rules are in `api-reference.md`.

For HTML5/React/Phaser/Pixi, reuse the `createRewardedAd` controller from
`references/html5-js.md`, adapted to the project's module style. Don't write the
lifecycle logic from scratch.

## 6. Server verification (required)

The client's `complete` is optimistic UI. Persistent or valuable rewards must be
credited by the game's server when AppLixir's callback arrives
(`references/server-verification.md`):

- the endpoint authenticates the callback (MD5 `signature`, else `secretKey`, compared in constant time);
- it dedupes on `tid` with a unique constraint, in the same transaction as the credit;
- the server decides the reward amount;
- the client re-fetches the balance from the server after `complete`;
- tell the user to set **Dashboard → Callbacks**: URL, secret, and mode **`md5AndTid`**.

**No backend in the project?** Don't skip this silently. Explain:
"Client-only rewards can be granted by anyone from DevTools. That's acceptable only
for cosmetic or session-only rewards."

**If the reward is persistent, paid, or tradeable** (a currency players can also buy,
saved progress, items), this is a decision the user must make. **Ask it even if
they said the plan is approved or "just give me the code"**: an approved plan
didn't include this trade-off. Note that if the balance itself lives client-side
(e.g. `localStorage`), verifying the ad alone doesn't help. The **balance** must
move to a server too. Offer:
- (a) the minimal Node/Python/.NET service from `server-verification.md`, which holds the balance, receives AppLixir's callback and dedupes on `tid`;
- (b) making the ad reward non-persistent (e.g. hints for this session only);
- (c) accepting the risk, recorded in a code comment.

Stop and wait for their choice before writing code. You may show the client-side
controller in the same reply, but don't present client-only crediting of a paid
currency as the finished solution.

## 7. Consent

Follow `references/consent-tcf.md`:
- **Existing TCF v2 CMP:** load it **before** the SDK, and add nothing else.
- **No CMP:** the SDK shows AppLixir's own notice to EEA/UK players. Tell the user. If the site runs other trackers, the site still needs its own CMP.
- **GPP-only CMP:** flag it, because the SDK doesn't read `__gpp`.

Never skip consent, fake a consent string, or tell the user it's optional.

## 8. Test

There's no test API key. Test with the real key on the **registered domain over
https**; localhost won't match. Until AppLixir approves the site, it receives **AppLixir
test ads**, which exercise the full lifecycle. Give the user this checklist:

- [ ] Console shows `loaded → started → firstQuartile → midpoint → thirdQuartile → complete` on a full view.
- [ ] Reward appears **once** after `complete`. The server credited it exactly once, and replaying the callback URL returns "duplicate".
- [ ] Skip / close early: no reward, the game resumes, the button re-enables.
- [ ] No fill or error (block `cdn.applixir.com` in DevTools → Network → Block request URL): friendly message, no reward, nothing broken.
- [ ] Double-clicking the button opens one player.
- [ ] Ad blocker on: the game still works.
- [ ] From the EEA/UK (or a VPN exit there): exactly one consent prompt.
- [ ] Audio and input work after the ad.

`references/troubleshooting.md` covers each failure.

## 9. Go-live checklist

- [ ] Site registered in the dashboard with the **exact** production domain (Sites → platform Web).
- [ ] The production API key comes from production config, not a dev placeholder.
- [ ] `ads.txt` entries from **Dashboard → Settings → Ads.txt** are live at `https://<domain>/ads.txt`.
- [ ] **Dashboard → Callbacks** points at the production endpoint (https, no `?` in the URL), mode `md5AndTid`, and the secret is in server env only.
- [ ] SDK pinned to `v6.1.0`.
- [ ] **Site approved by AppLixir.** Approval is a manual review. Until it happens, the site gets test ads. Contact support@applixir.com if approval is pending.

## Guardrails (always)

- **No auto-play** of rewarded ads. The player chooses to watch, every time.
- **No dark patterns:** don't hide or delay the player's close/skip, fake clicks, disguise ads as game content, or put ads where they block core gameplay or the only way to progress.
- **No reward on skip, close, no-fill, error or consent decline.**
- **No incentivized clicks:** reward watching, never clicking the ad.
- Don't modify the SDK, its creatives or its DOM.
- Keep sales language out of code and comments.

## Auditing an existing integration

When asked to check, review or debug an AppLixir integration, run the checks
in the `/applixir-doctor` command:
- user-triggered show;
- every callback path handled;
- server verification present;
- idempotency;
- consent;
- key handling;
- no-fill handling.

Report pass/fail with file:line evidence, then propose fixes and wait before
editing. Most "reward fires twice" bugs are covered in `troubleshooting.md`.
