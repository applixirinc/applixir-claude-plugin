---
type: llm
---

The reply should contain code integrating AppLixir rewarded video into a Phaser 3 game-over scene.

PASS only if ALL hold:
- Loads the SDK from https://cdn.applixir.com/applixir.app.v6.1.0.js and uses a div with id
  applixir-ad-container outside the Phaser canvas.
- The ad is triggered from a Phaser pointerdown (user input) handler.
- The extra life is granted only when status.type === "complete".
- Other endings (allAdsCompleted, skip/skipped, manuallyEnded, error) clean up / resume
  without granting.
- The API key comes from config or env (e.g. import.meta.env), not a hard-coded real key.

FAIL if it grants on allAdsCompleted, compares status to a string instead of status.type,
or uses invokeApplixirVideoUnit/zoneId.
