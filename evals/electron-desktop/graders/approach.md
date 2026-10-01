---
type: llm
---

The game window loads a local file (file://).
PASS only if the reply explains that the SDK should NOT simply be injected into the file://
page, because a null/file origin breaks consent and/or produces no fill (AdError 303) since
the ad server matches the registered domain, AND proposes loading an ad page from a real
https URL on the registered domain (e.g. starminer.io) in an Electron window/view, passing
the result back to the game, with the reward granted only on "complete".
FAIL if it says adding the script tag to the file:// page is sufficient.
