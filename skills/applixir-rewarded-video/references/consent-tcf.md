# Consent (TCF v2 / GDPR, EEA + UK)

Players in the EEA and UK must be asked for consent before personalized ads,
under GDPR and IAB TCF v2. Don't skip this, and don't tell the developer it's optional.

## What the AppLixir SDK does

| Page state | SDK behavior |
|---|---|
| **A TCF v2 CMP is on the page** (`window.__tcfapi` or a `__tcfapiLocator` frame; Didomi, Sourcepoint `#spcloader`) | The SDK **adopts it read-only**: it reads the TC string via `__tcfapi`, waits up to ~3 s for it, and passes `gdpr` / `gdpr_consent` on ad requests. Without consent it serves a **non-personalized** ad. It shows no second prompt. |
| **No CMP** | The SDK shows **AppLixir's own consent notice** (Didomi, loaded from `sdk.privacy-center.org`) where consent is required. |
| **GPP-only CMP** (`__gpp` without `__tcfapi`) | **Not read.** The SDK behaves as if no CMP is present. |

Statuses (`adStatusCallbackFn`):

| `status.type` | When | Handle as |
|---|---|---|
| `consentDeclined` | The user declined **AppLixir's own** notice. Never fires behind a publisher CMP. | no reward, clean up (`CLAUDE.md` table) |
| `consentUnavailable` | AppLixir's notice failed to load (`status.reason`: `timeout` / `loader-error` / `api-error`) | no reward, clean up |

## What to do in the developer's project

1. **Detect an existing CMP.** Search the project for:
   - `__tcfapi`, `__tcfapiLocator`, `__gpp`;
   - script hosts `didomi`, `privacy-center.org`, `sourcepoint`, `cookiebot`, `onetrust`, `cookielaw.org`, `quantcast`, `usercentrics`, `consentmanager`, `fundingchoicesmessages.google.com` (Google's CMP).
2. **If a TCF v2 CMP exists:**
   - load its stub/loader **before** the AppLixir `<script>`, so `__tcfapi` exists when the SDK checks;
   - don't add another CMP, and don't build custom consent UI for ads;
   - if it's GPP-only, flag it: AppLixir won't read it. Ask the developer to enable the CMP's TCF v2 API (most do both).
3. **If there's no CMP:**
   - tell the developer the SDK will show AppLixir's consent notice to EEA/UK players on first ad;
   - check the layout tolerates that overlay (it appears above the game);
   - if the **rest of the site** also sets cookies or runs other ad/analytics tags, the site itself still needs a CMP. AppLixir's notice covers AppLixir's ad requests, not the site's other trackers. Say this plainly; don't silently skip it.
4. **Handle the statuses.** `consentDeclined` / `consentUnavailable` grant nothing and must re-enable the button. The controller in `html5-js.md` already does this.
5. **Don't** gate the "Watch ad" button on consent yourself, fake a consent string, or pre-set `gdpr_consent`.

## React Native / WebViews

Consent talks over `postMessage`. A page with origin `"null"` (inline HTML,
`file://`) throws `Invalid target origin 'null'`, and no consent string is
produced (`applixir-integration` `examples/react-native/README.md`). Always load
the ad page from a real `https://` URL on the registered domain.

## Checking it works (from the EEA/UK, or a VPN exit there)

- [ ] First ad: exactly one consent prompt (yours or AppLixir's), never two.
- [ ] Console: `__tcfapi('getTCData', 2, console.log)` returns `gdprApplies: true` and a `tcString`.
- [ ] Declining doesn't break the game. The button re-enables, and later ads still play (non-personalized).
