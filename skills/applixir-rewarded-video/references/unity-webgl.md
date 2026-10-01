# Unity WebGL (Unity Web)

Built from `examples/unity-webgl/` (`AppLixirBridge.jslib`, `AppLixirManager.cs`,
`README.md`) and `prompts/unity.md` in `applixir-integration`.

The browser runs the SDK. Unity calls it through a `.jslib`, and the result comes
back to C# through `SendMessage`, which only takes a **string**, so the bridge
forwards `status.type`.

Do **not** generate `ApplixirWebGL.PlayVideo(...)` / `PlayVideoResult.ADWatched`.
`CLAUDE.md` and `llms.txt` name them, but no such package exists in any AppLixir
repo. The jslib bridge below is the supported path.

## Files

| File | Path in the Unity project |
|---|---|
| `AppLixirBridge.jslib` | `Assets/Plugins/WebGL/AppLixirBridge.jslib` |
| `AppLixirManager.cs` | any folder under `Assets/`, on a GameObject named **`AppLixirManager`** |
| SDK `<script>` | your WebGL template's `index.html` `<head>` (`Assets/WebGLTemplates/<Name>/index.html`) |

## 1. WebGL template `<head>`

```html
<script type="text/javascript" src="https://cdn.applixir.com/applixir.app.v6.1.0.js"></script>
```

If the project uses a built-in template (Default / Minimal), create a custom template
first. Copy the built-in one into `Assets/WebGLTemplates/AppLixir/` and select it in
**Player Settings → Resolution and Presentation → WebGL Template**. Otherwise the tag
gets lost on the next build.

## 2. `Assets/Plugins/WebGL/AppLixirBridge.jslib`

The API key and player id are passed **in from C#**, so they come from config,
not hard-coded in the bridge. The repo example hard-codes `apiKey` and has no
`userId`. The handler treats every non-`complete` ending as cleanup.

```js
mergeInto(LibraryManager.library, {

  PlayRewardedAd: function (apiKeyPtr, userIdPtr, callbackObjectNamePtr, callbackMethodNamePtr) {
    var apiKey = UTF8ToString(apiKeyPtr);
    var userId = UTF8ToString(userIdPtr);
    var callbackObjectName = UTF8ToString(callbackObjectNamePtr);
    var callbackMethodName = UTF8ToString(callbackMethodNamePtr);

    var container = document.getElementById("applixir-ad-container");
    if (!container) {
      container = document.createElement("div");
      container.id = "applixir-ad-container";
      container.style.cssText = "position:fixed;top:0;left:0;width:100%;height:100%;z-index:9999;";
      document.body.appendChild(container);
    }
    container.style.display = "block";

    function send(value) {
      // Use the global SendMessage that Unity's loader exposes (unityInstance.SendMessage on newer templates is also fine).
      SendMessage(callbackObjectName, callbackMethodName, value);
    }
    function hide() { container.style.display = "none"; }

    var options = {
      apiKey: apiKey,
      injectionElementId: "applixir-ad-container",
      userId: userId || undefined, // reaches your server callback so it knows whom to credit

      adStatusCallbackFn: function (status) {
        // status is an OBJECT { type, ad?, error? }; forward the string only
        var t = status.type;
        if (t !== "complete" && t !== "loaded" && t !== "started" && t !== "click" && t !== "paused" &&
            t !== "firstQuartile" && t !== "midpoint" && t !== "thirdQuartile") hide();
        send(t);
      },

      adErrorCallbackFn: function (error) {
        console.error("AppLixir error:", error.getError().data);
        hide();
        send("error"); // bridge sentinel, not an SDK status
      },
    };

    if (typeof initializeAndOpenPlayer === "function") {
      initializeAndOpenPlayer(options);
    } else {
      hide();
      send("sdk-not-loaded"); // bridge sentinel: script missing or blocked
    }
  },

});
```

## 3. `AppLixirManager.cs`

```csharp
using System;
using System.Runtime.InteropServices;
using UnityEngine;
using UnityEngine.Events;

public class AppLixirManager : MonoBehaviour
{
    [Tooltip("AppLixir API key. Set it in the Inspector or from build config; don't commit a real key to a public repo.")]
    [SerializeField] private string apiKey = "";

    // Set this to your logged-in player's id before showing an ad; AppLixir passes it to your server callback.
    public string PlayerId { get; set; } = "";

    private const float WatchdogSeconds = 15f; // bad key / no-fill can produce no callback at all

    public UnityEvent OnRewardEarned;   // optimistic UI: server callback is the source of truth
    public UnityEvent<string> OnAdFinishedWithoutReward; // reason: allAdsCompleted/skipped/...

    [DllImport("__Internal")]
    private static extern void PlayRewardedAd(string apiKey, string userId, string callbackObjectName, string callbackMethodName);

    private bool adInFlight;
    private bool rewardedThisAd;

    // Call from a button's OnClick (a user action), never automatically.
    public void ShowRewardedAd()
    {
        if (adInFlight) return;
        adInFlight = true;
        rewardedThisAd = false;
        Time.timeScale = 0f;              // pause gameplay behind the overlay
        AudioListener.pause = true;       // mute game audio during the ad

#if UNITY_WEBGL && !UNITY_EDITOR
        StartCoroutine(Watchdog());
        // gameObject.name must equal the string the bridge sends to
        PlayRewardedAd(apiKey, PlayerId, gameObject.name, nameof(OnAdStatusReceived));
#else
        Debug.Log("AppLixir runs only in WebGL builds. Simulating 'complete' in the Editor.");
        OnAdStatusReceived("complete");
        OnAdStatusReceived("allAdsCompleted");
#endif
    }

    private bool adStarted;

    private System.Collections.IEnumerator Watchdog()
    {
        adStarted = false;
        yield return new WaitForSecondsRealtime(WatchdogSeconds); // realtime: timeScale is 0
        if (adInFlight && !adStarted) Finish("timeout");
    }

    // Called by the jslib with status.type (a string). Must stay public (SendMessage).
    [UnityEngine.Scripting.Preserve]
    public void OnAdStatusReceived(string status)
    {
        switch (status)
        {
            case "loaded":
            case "started":
                adStarted = true;
                break;

            case "complete":
                if (!rewardedThisAd)
                {
                    rewardedThisAd = true;
                    OnRewardEarned?.Invoke();
                }
                break;

            // progress / interaction: nothing to do
            case "firstQuartile": case "midpoint": case "thirdQuartile":
            case "click": case "paused":
                break;

            // EVERY other value ends the ad without granting anything:
            // allAdsCompleted (end of any ad OR no ad), skipped/skip, manuallyEnded,
            // consentDeclined, consentUnavailable, adSkippedNoReward, thankYouModalClosed,
            // and the bridge sentinels "error" / "sdk-not-loaded".
            default:
                Finish(status);
                break;
        }
    }

    private void Finish(string reason)
    {
        if (!adInFlight) return;  // complete → allAdsCompleted both land here; finish once
        adInFlight = false;
        Time.timeScale = 1f;
        AudioListener.pause = false;
        if (!rewardedThisAd) OnAdFinishedWithoutReward?.Invoke(reason);
    }
}
```

Wire `OnRewardEarned` to UI that shows "reward pending" and asks your backend for the
authoritative balance. The backend credits the reward when AppLixir's signed callback
arrives (`server-verification.md`). Wire `OnAdFinishedWithoutReward` to a toast
("No ad available right now" for `allAdsCompleted`/`error`, "Watch the whole video" for
`skipped`/`manuallyEnded`).

## Build checklist

- [ ] Build target is **WebGL**. `#if UNITY_WEBGL && !UNITY_EDITOR` means the Editor only simulates.
- [ ] The `.jslib` is under a `Plugins` folder and its import settings include **WebGL** (select the file → Inspector → Platform settings → WebGL ✔). Otherwise you get `EntryPointNotFoundException: PlayRewardedAd` at runtime.
- [ ] The `[DllImport]` signature matches the jslib: **four** string args here (the repo example has two because it hard-codes the key and omits `userId`).
- [ ] The GameObject name matches what you pass (`gameObject.name` above).
- [ ] The custom WebGL template's `index.html` contains the SDK `<script>`.
- [ ] Code stripping: `AppLixirManager` is in a scene, and `OnAdStatusReceived` is public and marked `[Preserve]`. With **High** managed stripping, also add a `link.xml` entry for the assembly if the component is only added at runtime.
- [ ] Deployed to the registered domain over https. Localhost doesn't match.

## Unity on a game portal

If the build is uploaded to a portal that wraps it in an iframe, see
`troubleshooting.md` § Iframes / game portals.
