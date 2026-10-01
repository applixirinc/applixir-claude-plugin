---
type: llm
---

PASS only if ALL hold:
- Identifies the root cause: gold is granted on both "complete" and "allAdsCompleted"
  (allAdsCompleted fires after every ad, so a full view grants twice).
- Says to grant only on status.type === "complete".
- Also flags that the reward is credited by a client POST to /api/gold (forgeable, no
  server-side verification) and recommends AppLixir's server callback with dedupe/idempotency.
- Does not claim to have already edited files; proposes fixes and asks before applying them.
FAIL otherwise.
