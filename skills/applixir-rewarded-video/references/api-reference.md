# AppLixir Web SDK v6.1.0 — API reference

Every row cites the file in [`applixirinc/applixir-integration`](https://github.com/applixirinc/applixir-integration)
it came from. Do not use any API that is not in this table.

## Loading

| Item | Value | Source |
|---|---|---|
| SDK script | `<script type="text/javascript" src="https://cdn.applixir.com/applixir.app.v6.1.0.js"></script>` | `CLAUDE.md` § Current SDK Version; `llms.txt` § SDK |
| Version | `6.1.0`, pinned in the URL. Always pin it. | `CLAUDE.md`; `prompts/html5.md` § Hard rules |
| Package manager | None. There is no npm package; load the CDN script. | `examples/react/README.md` § Notes; `prompts/react.md` |
| Player anchor | `<div id="applixir-ad-container"></div>`, where the player injects itself | `CLAUDE.md` § Step 2; `examples/html5/index.html` |
| Globals exposed | `initializeAndOpenPlayer`, `Application`, `preloadAd` on `window` | `llms.txt` § SDK |

## Options object

The same object is passed to `initializeAndOpenPlayer`, `new Application` and `preloadAd`.

| Option | Required | Type | Meaning | Source |
|---|---|---|---|---|
| `apiKey` | yes | string, format `xxxx-xxxx-xxxx-xxxx` | Publisher API key from the dashboard (Sites → your site) | `CLAUDE.md` § Step 1, Step 4 |
| `injectionElementId` | yes | string | `id` of the anchor div | `CLAUDE.md` § Step 4 |
| `adStatusCallbackFn` | yes | `(status) => void` | Lifecycle callback; `status` is an **object** | `CLAUDE.md` § Step 4 |
| `adErrorCallbackFn` | optional, recommended | `(error) => void` | Error callback | `CLAUDE.md` § Step 4 |

Two more options are read by the SDK and reach the server callback. They aren't
in `applixir-integration` yet.

| Option | Required | Type | Meaning |
|---|---|---|---|
| `userId` | needed for server verification | string | Your player's id. AppLixir passes it to your callback as `userId` so your server knows whom to credit. |
| `customData` | optional | plain object | Passed to your callback as URL-encoded JSON `customData` (e.g. `{ placement: "shop" }`). |

Don't use any other option.

## Functions

| Function | Signature | Behavior | Source |
|---|---|---|---|
| `initializeAndOpenPlayer` | `initializeAndOpenPlayer(options): void` | Runs the auction, requests the video and opens the player. Call from a user gesture. Takes ~1.5–3.5 s from click to reveal. | `CLAUDE.md` § Step 4, § Faster Ad Loading |
| `Application` | `new Application(options)` | Advanced lifecycle control | `CLAUDE.md` § Advanced approach |
| `app.initialize()` | `(): void` | Must run after `window.onload` (DOM ready) | `CLAUDE.md` § Advanced approach; `llms.txt` § Application Class |
| `app.openPlayer()` | `(): void` | Opens the player; call on user action after `initialize()` | same |
| `preloadAd` | `preloadAd(options): Promise<{ show }>` | Runs the auction and video request **before** the click. Rejects on no-fill or network error. | `CLAUDE.md` § Faster Ad Loading; `llms.txt` |
| `handle.show()` | `(): Promise` | Reveals the preloaded ad in ~100–300 ms. **Single-use.** Call inside the click handler, which preserves the iOS/Safari gesture chain. | same |

### `preloadAd` rules (`CLAUDE.md` § Faster Ad Loading, `llms.txt`)

1. Preload on a **high-intent signal** (reward modal mount, level complete, out of coins), not on page load.
2. **Always keep an on-click fallback** to `initializeAndOpenPlayer(options)`. About 10–30% of preloads no-fill, and Incognito or blocked third-party cookies can fail silently. Never call `show()` on a missing handle.
3. **Re-preload after each `show()`.** Handles are single-use.
4. **Bids expire after about 5 minutes.** Refresh on a timer or re-preload on the next intent signal.
5. Always-visible button: preload on first render, refresh every ~4 min, refresh on `visibilitychange → visible`, and skip refreshes while hidden.

## `adStatusCallbackFn(status)`: status object

`status` is `{ type, ad?, error? }`. **Always read `status.type`.** Comparing
`status` to a string (`status === "ad-watched"`) is the old API and never matches.
Sources: `CLAUDE.md` § adStatusCallbackFn; `README.md` § Key Concepts; `llms.txt`.

| `status.type` | Meaning | Grant reward? |
|---|---|---|
| `loaded` | Ad data available | — |
| `started` | Playback began | — |
| `firstQuartile` / `midpoint` / `thirdQuartile` | 25 / 50 / 75 % | — |
| `complete` | User watched the full video | ✅ **only this** |
| `allAdsCompleted` | Fires at the end of **any** ad **and** when **no ad** was available | ❌ clean up / re-enable UI |
| `click` | Ad clicked | — |
| `paused` | Ad paused | — |
| `skipped` | User skipped | ❌ |
| `manuallyEnded` | User closed early | ❌ |
| `consentDeclined` | User declined personalized-ads consent | ❌ |

Order on a full view: `loaded → started → firstQuartile → midpoint → thirdQuartile → complete`
(then `allAdsCompleted`). Source: `CLAUDE.md` § Testing; `examples/react/useRewardedAd.js` comment
("allAdsCompleted also fires after a real completion").

**The SDK also emits types the table doesn't list.** Write handlers so that
**only `complete` grants** and every other terminal type cleans up. Never rely on
an exhaustive list. <!-- GAP #6: from source, pending confirmation -->

| Extra `status.type` (from SDK source) | Treat as |
|---|---|
| `skip` (IMA's skip event; the docs say `skipped`) | no reward, clean up |
| `adSkippedNoReward` | no reward, clean up |
| `thankYouModalClosed` (after a watched ad) | clean up |
| `consentUnavailable` (`status.reason`: `timeout` / `loader-error` / `api-error`) | no reward, clean up |

`status.ad` (when present) = `{ adId, isLinear, duration, title, adSystem }`.

### Callbacks that may never come: arm a watchdog

From SDK source <!-- GAP #7: from source, pending confirmation -->:
- **No-fill on click:** only `adErrorCallbackFn` fires (no `allAdsCompleted`).
- **`preloadAd()` no-fill:** the promise rejects and `adErrorCallbackFn` is **not** called.
- **Bad or unregistered `apiKey`:** sometimes `adErrorCallbackFn`, sometimes **no callback at all** (seen in a browser test on 2026-10-01).

So every integration must reset its UI on `adErrorCallbackFn` **and** on
`allAdsCompleted`, **and** arm a timeout (≈15 s) that resets the UI if neither
`loaded` nor `started` arrives.

## `adErrorCallbackFn(error)`

```js
adErrorCallbackFn: (error) => {
  const data = error.getError().data; // { type, errorCode, errorMessage, ... }
}
```
Source: `CLAUDE.md` § adErrorCallbackFn; type shape in `examples/react/useRewardedAd.ts`
(`{ type: string; errorCode: number; errorMessage: string }`).

| Code | Meaning | Source |
|---|---|---|
| `303` | "No Ads". Also returned when the request origin doesn't match the registered domain (e.g. origin `null`). | `examples/react-native/README.md`; `CLAUDE.md` § React Native |

More codes from the SDK's own documentation (IMA/VAST codes). <!-- GAP #8: from source (html-player DOCUMENTATION.md), pending confirmation -->

| Code | `type` | Usual meaning |
|---|---|---|
| `0` | `playerInitializationFailed` | SDK setup failed |
| `301` | `vastLoadTimeout` | Ad response too slow |
| `303` | `vastNoAdsAfterWrapper` | No fill |
| `1009` | `vastEmptyResponse` | No fill |
| `1012` | `adsRequestNetworkError` | Network / blocker |
| `1205` | `autoplayDisallowed` | Not started from a user gesture |

Treat every code the same in the UI ("No ad right now, try later"). Log `data` for debugging.

## Unity WebGL bridge values (strings forwarded to C#)

The `.jslib` forwards `status.type` as a string. It also adds two sentinels of its own.
Source: `examples/unity-webgl/README.md`, `AppLixirBridge.jslib`.

| Value | Origin |
|---|---|
| any `status.type` above | SDK |
| `error` | Bridge sentinel from `adErrorCallbackFn` |
| `sdk-not-loaded` | Bridge sentinel, `initializeAndOpenPlayer` missing |

Don't generate `ApplixirWebGL.PlayVideo` / `PlayVideoResult`. They're named in
`CLAUDE.md`/`llms.txt`, but no code for them exists. Use the `.jslib` bridge. <!-- GAP #5 -->

## Server-side reward callback

| Item | Value | Source |
|---|---|---|
| Configure | Dashboard → Callbacks → endpoint URL + secret | `CLAUDE.md` § Server-Side Reward Callback |
| Transport | Signed HTTPS POST from AppLixir to your server after each verified ad completion | same; `llms.txt` § Server-Side Callback |
| Role | **Source of truth** for persistent rewards. Client `complete` is optimistic UI only. | same |

Wire contract (method, parameters, signature, `tid`, retries): see
`server-verification.md`. It comes from AppLixir's server code, and the public
repo's "signed POST" wording is inaccurate.

## Account, domain and revenue requirements

| Item | Value | Source |
|---|---|---|
| Sign up | https://client.applixir.com/register | `CLAUDE.md` |
| Dashboard | https://client.applixir.com | `README.md` § Links |
| Register site | Dashboard → Sites, platform = **Web**. Exact domain match required. | `CLAUDE.md` § Step 1, § Testing |
| Localhost | Won't match a registered domain; test on the deployed domain | `CLAUDE.md` § Common Mistakes #7, § Testing |
| Test key | None. "No test/sandbox API key is required — use your real API key from the start." | `CLAUDE.md` § Testing |
| Before approval | Approval is a **manual review** by AppLixir. Until then the site is served **AppLixir test ads**, so the full lifecycle can be tested with the real key on the registered domain. | Confirmed by AppLixir (2026-10-01) |
| `ads.txt` | Entries from Dashboard → Settings → Ads.txt, served at `https://yourdomain/ads.txt`. Without it fill and CPM drop significantly. | `CLAUDE.md` § ads.txt |
| User gesture | Ads must be triggered by click/tap; browsers block autoplay | `CLAUDE.md` § Common Mistakes #6 |
| Support | support@applixir.com · https://support.applixir.com | `README.md` § Links |

## Deprecated: never generate

`applixir.sdk2.1m.js`, `invokeApplixirVideoUnit()`, `zoneId`, `devId`, `gameId`,
and string statuses such as `"ad-watched"` / `"no-ad"`.
Source: `CLAUDE.md` § Current SDK Version, § Common Mistakes #1, #5; `prompts/unity.md`.
