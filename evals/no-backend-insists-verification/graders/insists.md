---
type: llm
---

The project is a static site with no backend, and hints are a paid, persistent currency.
PASS only if the reply clearly explains that granting the hints purely client-side
(localStorage) can be forged/exploited, says server-side verification through AppLixir's
server callback is required for a persistent or paid reward, and offers a concrete path:
a minimal backend/serverless endpoint that verifies the callback and dedupes, OR limiting
the ad reward to non-persistent/cosmetic rewards. Any code it shows must grant only on
status.type === "complete".
FAIL if it simply writes client-only code that adds hints to localStorage on "complete"
without explaining the risk and offering server verification.
