# HTML5 / vanilla JS (also Phaser, PixiJS, Three.js, Cocos web builds)

Built from `examples/html5/index.html`, `examples/phaser3/game.js` and `CLAUDE.md`
in `applixir-integration`. The SDK is a DOM overlay, so every canvas engine uses
the same three pieces: script tag, anchor div, user-triggered call.

## The controller pattern (use this in every engine)

One small controller owns the ad lifecycle. Engines only call `showRewardedAd()`
from a user gesture and react to `onReward` / `onDone`.

Save as `applixir-ad.js`:

```js
// applixir-ad.js: AppLixir rewarded-video controller (SDK v6.1.0)
// Rules: show only from a user gesture; reward only on status.type === "complete";
// every other ending cleans up; a watchdog covers callbacks that never arrive.
(function () {
  const WATCHDOG_MS = 15000; // no "loaded"/"started" by then → give up and reset UI

  function createRewardedAd({ apiKey, userId, containerId = "applixir-ad-container", onOpen, onReward, onDone }) {
    let inFlight = false, rewarded = false, finished = false, watchdog = null;
    let preloaded = null;
    const container = () => document.getElementById(containerId);

    function finish(reason) {
      if (!inFlight || finished) return;   // complete → allAdsCompleted both arrive; finish once
      finished = true; inFlight = false;
      clearTimeout(watchdog);
      const c = container(); if (c) c.style.display = "none";
      onDone && onDone({ rewarded, reason });
    }

    const options = {
      apiKey,
      injectionElementId: containerId,
      // userId reaches your server callback so it knows whom to credit.
      userId,

      // status is an OBJECT { type, ad?, error? }: read status.type, never compare status to a string
      adStatusCallbackFn: (status) => {
        const t = status && status.type;
        if (t === "loaded" || t === "started") clearTimeout(watchdog);
        if (t === "complete") {
          if (!rewarded) { rewarded = true; onReward && onReward(); } // optimistic UI only
          return;
        }
        // Any other terminal type: allAdsCompleted (end of ANY ad, or no ad), skipped/skip,
        // manuallyEnded, consentDeclined, consentUnavailable, adSkippedNoReward, thankYouModalClosed
        if (["allAdsCompleted", "skipped", "skip", "manuallyEnded", "consentDeclined",
             "consentUnavailable", "adSkippedNoReward", "thankYouModalClosed"].includes(t)) {
          finish(t);
        }
      },

      adErrorCallbackFn: (error) => {
        // On click-path no-fill this is the ONLY callback that fires.
        try { console.warn("AppLixir error:", error.getError().data); } catch (_) {}
        finish("error");
      },
    };

    function begin() {
      inFlight = true; rewarded = false; finished = false;
      const c = container(); if (c) c.style.display = "block";
      clearTimeout(watchdog);
      // A bad/unregistered key can fire NO callback at all (verified in a browser); don't leave the button dead.
      watchdog = setTimeout(() => finish("timeout"), WATCHDOG_MS);
      onOpen && onOpen();
    }

    return {
      get busy() { return inFlight; },

      // Optional fast path: call on a HIGH-INTENT signal (out-of-lives modal opens), not on page load.
      async preload() {
        if (typeof window.preloadAd !== "function") return;
        try { preloaded = await window.preloadAd(options); }
        catch (_) { preloaded = null; } // no-fill rejects WITHOUT calling adErrorCallbackFn
      },

      // Call ONLY from a click/tap handler.
      async show() {
        if (inFlight) return false;
        if (typeof window.initializeAndOpenPlayer !== "function") {
          onDone && onDone({ rewarded: false, reason: "sdk-not-loaded" }); // blocked by ad blocker / CSP / offline
          return false;
        }
        begin();
        if (preloaded) {
          const h = preloaded; preloaded = null; // handles are single-use
          try { await h.show(); } catch (_) { window.initializeAndOpenPlayer(options); }
        } else {
          window.initializeAndOpenPlayer(options); // always-available fallback
        }
        return true;
      },
    };
  }

  window.createRewardedAd = createRewardedAd;
})();
```

## Complete minimal page

Serve it from the **registered domain** over https. Localhost doesn't match
(`CLAUDE.md` § Testing).

```html
<!DOCTYPE html>
<html lang="en">
<head>
  <meta charset="UTF-8">
  <meta name="viewport" content="width=device-width, initial-scale=1.0">
  <title>My Game</title>
  <!-- If the site has a TCF CMP, its stub/loader goes HERE, before the SDK (consent-tcf.md) -->
  <!-- 1. SDK, pinned (CLAUDE.md § Current SDK Version) -->
  <script type="text/javascript" src="https://cdn.applixir.com/applixir.app.v6.1.0.js"></script>
  <script src="game-config.js"></script>   <!-- window.GAME_CONFIG = { applixirApiKey: "..." } -->
  <script src="applixir-ad.js"></script>
  <style>
    #applixir-ad-container { position: fixed; inset: 0; z-index: 9999; display: none; }
  </style>
</head>
<body>
  <!-- 2. Anchor the player injects into -->
  <div id="applixir-ad-container"></div>

  <p>Lives: <span id="lives">0</span></p>
  <button id="watch-ad-btn">Watch an ad for +3 lives</button>
  <p id="ad-msg" aria-live="polite"></p>

  <script>
    const btn = document.getElementById("watch-ad-btn");
    const msg = document.getElementById("ad-msg");

    const ad = createRewardedAd({
      apiKey: window.GAME_CONFIG.applixirApiKey,
      userId: window.GAME_CONFIG.playerId,      // your logged-in player's id
      onOpen() { btn.disabled = true; msg.textContent = ""; /* pause loop, mute audio */ },
      onReward() { msg.textContent = "Reward on its way…"; },
      async onDone({ rewarded, reason }) {
        btn.disabled = false;                     /* resume loop, unmute audio */
        if (rewarded) {
          // Authoritative balance comes from YOUR server, which credited it on AppLixir's callback.
          // document.getElementById("lives").textContent = (await (await fetch("/api/me")).json()).lives;
        } else if (reason === "skipped" || reason === "skip" || reason === "manuallyEnded" || reason === "adSkippedNoReward") {
          msg.textContent = "Watch the whole video to get the reward.";
        } else if (reason !== "consentDeclined") {
          msg.textContent = "No ad available right now. Try again later.";
        }
      },
    });

    // 3. Only on a user gesture: never on load, never on a timer
    btn.addEventListener("click", () => ad.show());
  </script>
</body>
</html>
```

`game-config.js` is generated by your build or server, not hard-coded in game logic.
The AppLixir key is a client-side key and will be visible in the browser; that's
expected. Just don't commit it to a public repo.

```js
window.GAME_CONFIG = { applixirApiKey: "YOUR-API-KEY-HERE", playerId: "player-123" };
```

With a bundler, read it from env (`import.meta.env.VITE_APPLIXIR_API_KEY`,
`process.env.APPLIXIR_API_KEY`).

### No backend yet?

`onReward` alone is **client-side and forgeable**: anyone can call it from DevTools.
That's acceptable only for cosmetic, non-persistent rewards. For currency, lives
that persist, or anything tradeable, the server must credit on AppLixir's callback
(`server-verification.md`).

## Faster reveal with `preload()` (recommended for production)

Source: `CLAUDE.md` § Faster Ad Loading. The controller already implements the
rules: single-use handle, on-click fallback, a rejected preload falls back.

```js
openOutOfLivesModal();
ad.preload();               // high-intent moment; NOT page load
// modal button: onclick = () => ad.show();
```

Bids expire after ~5 min. If the modal can stay open longer, call `ad.preload()`
again on a timer. For an always-visible button: preload on first render, every
~4 min, and on `visibilitychange` → visible (not while hidden).

## Advanced: `Application` class

```js
const app = new Application(options);
window.onload = () => app.initialize();   // must be after DOM ready
btn.addEventListener("click", () => app.openPlayer());
```
Source: `CLAUDE.md` § Advanced approach. Prefer the controller. If the project
already uses `Application`, keep it and apply the same callback rules, guard and
watchdog.

## Phaser 3

Source: `examples/phaser3/game.js`, `prompts/phaser.md`.

- Script tags and the anchor div go in the HTML **before** the Phaser bundle, **outside** the canvas. Toggle `display`; don't remove the div.
- Trigger from `gameObject.on("pointerdown", …)`, which counts as a user gesture.
- `this.scene.pause()` in `onOpen`, `this.scene.resume()` in `onDone`. `onDone` runs on **every** ending (complete, no-fill, skip, error, timeout).

```js
class GameScene extends Phaser.Scene {
  create() {
    this.ad = createRewardedAd({
      apiKey: window.GAME_CONFIG.applixirApiKey,
      userId: window.GAME_CONFIG.playerId,
      onOpen: () => { this.scene.pause(); this.sound.pauseAll(); },
      onReward: () => this.events.emit("reward-pending"),
      onDone: ({ rewarded }) => {
        this.scene.resume(); this.sound.resumeAll();
        if (rewarded) this.refreshLivesFromServer();
        else this.showToast("No reward this time");
      },
    });
    this.add.text(400, 300, "Watch ad for +1 life", { backgroundColor: "#4CAF50", padding: { x: 16, y: 10 } })
      .setInteractive({ useHandCursor: true })
      .on("pointerdown", () => this.ad.show());
  }
}
```

If the button lives in a separate UI scene, pause the gameplay scene by key:
`this.scene.pause("GameScene")`. A paused scene can't receive the resume call
from its own input, so the controller's callbacks drive it.

## PixiJS / Three.js / Cocos / custom canvas

The same as vanilla: call `ad.show()` from the pointer event (`sprite.on("pointertap")`
in Pixi v7+, or a DOM button over the canvas). In `onOpen` stop the ticker
(`app.ticker.stop()` / `cancelAnimationFrame`) and mute audio; restore both in `onDone`.

## React (web)

The repo ships a hook: `examples/react/useRewardedAd.js` / `.ts`. `showAd()` resolves
`true` once on `complete`. Copy it, and add two things the repo hook lacks:
- `userId` in the options;
- the watchdog (resolve `false` after 15 s with no `loaded`/`started`), because a bad key otherwise leaves the promise pending forever.

The anchor div stays mounted, the trigger is a click, and with SSR (Next.js) the
call happens only in client components.
