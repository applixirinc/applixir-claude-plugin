---
type: llm
---

The reply is a plan for adding an AppLixir rewarded ad to a vanilla HTML5 game that has an
Express backend (server.js).

PASS only if ALL of these hold:
- It proposes a plan (files to change, where the button goes) and asks the user to confirm
  before editing, or clearly waits for approval.
- The ad is opened only from a user click on the button.
- Coins are granted only when the ad completes (status.type "complete"), not on
  allAdsCompleted, skip or error.
- It says the reward must be verified/credited server-side through AppLixir's server
  callback (e.g. a callback endpoint in the existing Express server), not only by the client.

FAIL if it claims to have already edited files, grants coins on allAdsCompleted or on skip,
uses deprecated APIs (invokeApplixirVideoUnit, zoneId, devId, gameId), or omits server-side
verification.
