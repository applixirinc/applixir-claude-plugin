# Troubleshooting

Start every diagnosis with the browser DevTools **Console** and **Network** tabs on
the **deployed, registered** domain. Expected on a full view:
`loaded → started → firstQuartile → midpoint → thirdQuartile → complete → allAdsCompleted`
(`applixir-integration` `CLAUDE.md` § Testing).

## Nothing happens on click / button stays disabled forever

| Cause | Check | Fix |
|---|---|---|
| SDK not loaded | `typeof initializeAndOpenPlayer` in the console is `"undefined"` | Script tag missing, blocked (ad blocker, CSP) or offline. Guard the call and show "Ads unavailable" (controller in `html5-js.md`). |
| Bad / unregistered API key | Sometimes an error callback, sometimes **no callback at all** | Copy the key from Dashboard → Sites. The **watchdog** (15 s) re-enables the UI. |
| Called before load | Call made at parse time, before the script finished | Call only from a click; with `Application`, `initialize()` after `window.onload`. |
| Not from a gesture | `autoplayDisallowed` (code 1205) | Call from the click/tap handler itself, not from a `setTimeout` or promise chain started elsewhere. With `preloadAd`, call `handle.show()` inside the click. |

## No fill ("no ad available")

- **Click path:** only `adErrorCallbackFn` fires (e.g. code `303` / `1009`); there's no `allAdsCompleted`. **`preloadAd()`:** the promise rejects and `adErrorCallbackFn` isn't called.
- UI: friendly message ("No ad right now, try again in a bit"), button re-enabled, no reward, game resumes.
- Common causes:
  - testing on **localhost** or a domain not registered in the dashboard (exact match required);
  - **origin `null`** (`file://`, inline HTML) gives `303`. Desktop apps (Electron/CEF) are out of scope for this reason;
  - missing **`ads.txt`** (Dashboard → Settings → Ads.txt);
  - normal no-fill: about 10–30% of preloads (`CLAUDE.md`).
- Before AppLixir approves the site (a manual review), it gets **AppLixir test ads**. That's expected.

## Ad blockers

The SDK script or ad requests are blocked. Never let that break the game:
- guard `typeof initializeAndOpenPlayer === "function"` before calling;
- show "Ads are blocked. Disable your ad blocker to earn rewards," or hide the button;
- the watchdog covers requests that hang;
- don't nag, block gameplay, or retry in a loop.

## Content-Security-Policy blocks the player

Symptom: console `Refused to load the script/frame/media … because it violates the
following Content Security Policy directive`.

Starting allowlist from AppLixir's SDK source:

```
script-src  'self' https://cdn.applixir.com https://imasdk.googleapis.com https://sdk.privacy-center.org https://securepubads.g.doubleclick.net;
connect-src 'self' https://api.applixir.com https://prebid.applixir.com https://*.doubleclick.net https://imasdk.googleapis.com https://*.privacy-center.org;
frame-src   https://imasdk.googleapis.com https://*.doubleclick.net https://*.privacy-center.org;
media-src   https: blob:;
img-src     https: data:;
style-src   'self' 'unsafe-inline' https://cdn.applixir.com;
```

Ad creatives and bidder user-syncs load hosts that can't be listed in advance. Deploy
`Content-Security-Policy-Report-Only` first, collect violations for a few days, then
enforce. A strict CSP and programmatic video ads are hard to combine; `https:` for
`media-src`/`img-src` is usually required.

## Iframes, sandboxed embeds, game portals

- Sandboxed iframes need at least `sandbox="allow-scripts allow-same-origin allow-popups allow-popups-to-escape-sandbox"` and `allow="autoplay; fullscreen"`. Without `allow-same-origin` the origin is `null`, which means no consent and `303` (AppLixir support docs).
- Portals that wrap your game in their iframe may have their own ad SDK and rules. AppLixir is complementary to portal ads. Check the portal's policy on third-party ads before enabling AppLixir there, and keep AppLixir to your own domain if the portal disallows it.
- Which domain to register for a game shown inside a portal's iframe isn't documented. Ask support@applixir.com.

## HTTP vs HTTPS

Serve the game over **https**. Mixed content (an http page loading https ads, or
the reverse) gets blocked, and consent, cookies and autoplay behave worse on http.
`localhost` is not a substitute; it won't match the registered domain.

## localhost and test domains

There's no test key; use the real key (`CLAUDE.md` § Testing). The domain must
**exactly** match the dashboard registration, so test on the deployed domain (a
staging subdomain registered as its own site works the same way). Locally you can
check the UI paths (button disables, watchdog fires, error toast) but not a real fill.

## Reward fires twice

| Cause | Fix |
|---|---|
| Granting on `allAdsCompleted` as well as `complete` | Grant **only** on `complete`. `allAdsCompleted` fires after every ad. |
| No per-ad guard | `rewarded` flag reset when an ad starts, set on first `complete` |
| Double click opened two players | `inFlight` guard + disable the button while an ad is open |
| Handler registered on every scene load / React render | Create the controller once; in React keep it in a ref and the anchor div mounted |
| `preloadAd` handle reused | Handles are single-use: clear it before `show()` |
| Server credits on both the client claim and the AppLixir callback | Credit **only** in the callback; dedupe on `tid` (`server-verification.md`) |
| Callback replayed | Dashboard callback mode `md5AndTid` + unique `tid` constraint |

## Server callback never arrives

- Dashboard → Callbacks: the URL is set (per game, or account-wide), https, and publicly reachable. **The URL must not already contain a `?` query string**: AppLixir appends `?gameApiKey=…`.
- The SDK must be given `userId`, or your handler can't credit anyone.
- Your endpoint must return `2xx` within 10 s. There are **no retries**.
- `403` from your own check: wrong secret, or you're verifying `signature` in a mode that doesn't send it. See the formula in `server-verification.md`.

## Unity WebGL: build strips or misses the `.jslib`

| Symptom | Fix |
|---|---|
| `EntryPointNotFoundException: PlayRewardedAd` | The `.jslib` must be under `Assets/Plugins/…` (or any folder) with **WebGL** ticked in its import settings |
| `OnAdStatusReceived` never called | The GameObject name passed to the bridge doesn't match, the object was destroyed, or the method was stripped. Keep it public, add `[Preserve]`, and use `gameObject.name`. |
| C# receives `[object Object]` | The bridge sent the status object; send `status.type` |
| SDK script missing after rebuild | It was added to the built `index.html` rather than the **WebGL template** |
| Works in Editor only | The Editor path is a simulation (`#if UNITY_WEBGL && !UNITY_EDITOR`) |

## Audio / focus problems after an ad

- **Game audio plays under the ad:** mute in `onOpen` (`AudioListener.pause = true`, `this.sound.pauseAll()`, Howler `Howler.mute(true)`) and restore in `onDone`.
- **Game audio silent after the ad:** a Web Audio `AudioContext` got suspended. Call `audioCtx.resume()` in `onDone` (Unity WebGL resumes on the next user input).
- **Keyboard/gamepad input dead:** focus moved to the ad overlay. In `onDone`, `canvas.focus()` (give the canvas `tabindex="0"`).
- **Game ran behind the ad:** pause the loop/scene in `onOpen`, resume in `onDone`. `onDone` runs on every ending, so the game always resumes.

## TCF string missing / consent problems

- `__tcfapi` undefined but the site has a CMP: the CMP loads **after** the SDK, or it's GPP-only. Load the CMP stub first, and enable its TCF v2 API. See `consent-tcf.md`.
- `Invalid target origin 'null'`: the page is `file://` or inline HTML. Host it on https.
- Two consent prompts: the site's CMP loaded too late, so the SDK showed its own. Fix the load order.
- `consentDeclined` / `consentUnavailable`: no reward, re-enable the button; later ads can still serve non-personalized.

## Still stuck

Collect the console log of one attempt, the `error.getError().data` object, the
page URL and the registered site URL, and send them to support@applixir.com.
