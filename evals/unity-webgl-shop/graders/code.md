---
type: llm
---

The reply should integrate AppLixir into a Unity WebGL project.

PASS only if ALL hold:
- It provides a .jslib (mergeInto(LibraryManager.library, ...)) that calls
  initializeAndOpenPlayer and forwards status.type (a string) to C# via SendMessage.
- C# uses [DllImport("__Internal")] and grants gems only on "complete".
- The SDK script tag goes into the WebGL template's index.html.
- The button triggers the ad (user action).

FAIL if it uses ApplixirWebGL.PlayVideo / PlayVideoResult, grants on allAdsCompleted, or
passes the whole status object to SendMessage.
