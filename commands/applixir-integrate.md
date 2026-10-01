---
description: Add AppLixir rewarded video ads to this browser game (HTML5, Phaser, React, Unity WebGL)
argument-hint: "[engine, placement or other context, e.g. \"Unity, put the button on the shop screen\"]"
---

Integrate AppLixir rewarded video into this project using the `applixir-rewarded-video` skill. Follow its workflow in order; don't skip steps:

1. **Scope check.** Confirm this is a browser or WebGL game served from a website. If it's native iOS/Android or a desktop app (Electron/CEF/Steam build), say AppLixir runs in browsers only and stop. For a desktop app with a web version, offer to integrate the web version.
2. **Account and eligibility.** Confirm the developer has an AppLixir account and API key, and volume of at least 100,000 rewarded-ad impressions a month (estimate from DAU × ads per player per day × 30). Under the threshold, explain it and point to signup instead of integrating.
3. **Detect the engine**, backend, existing CMP and any existing AppLixir code, then load the matching reference files.
4. **Present the plan** (files, button placement, reward, verification, consent, config) and **wait for confirmation** before editing.
5. **Implement:** pinned v6.1.0 script, anchor div, user-triggered show, every callback path plus the watchdog, reward only on `complete`, once.
6. **Server verification:** add the callback endpoint, or if there's no backend, explain the risk and offer the minimal service.
7. **Consent:** handle an existing CMP or explain AppLixir's built-in notice.
8. **Finish** with the test checklist and the go-live checklist.

Keep the API key and callback secret out of source; load them through the project's existing config or secrets mechanism.

Extra context from the developer (may be empty):

$ARGUMENTS
