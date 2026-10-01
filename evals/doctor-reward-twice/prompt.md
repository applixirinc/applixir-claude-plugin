---
max_turns: 25
timeout_seconds: 600
allowed_tools: [Read, Glob, Grep, Skill]
tags: [doctor]
---

Players are getting 200 gold instead of 100 from my AppLixir rewarded ad. The code is in
`project`. Can you check the integration?

The `project` files are pasted below (you can't open them directly):

`project/public/index.html`
```html
<!DOCTYPE html>
<html><head>
<script src="https://cdn.applixir.com/applixir.app.v6.1.0.js"></script>
</head><body>
<div id="applixir-ad-container"></div>
<button id="ad">+100 gold</button>
<script src="ads.js"></script>
</body></html>
```

`project/public/ads.js`
```js
const opts = {
  apiKey: window.CFG.applixirKey,
  injectionElementId: "applixir-ad-container",
  adStatusCallbackFn: (status) => {
    if (status.type === "complete" || status.type === "allAdsCompleted") {
      fetch("/api/gold", { method: "POST", body: JSON.stringify({ add: 100 }) });
    }
  },
};
document.getElementById("ad").addEventListener("click", () => initializeAndOpenPlayer(opts));
```

`project/server.js`
```js
const express = require("express"); const app = express(); app.use(express.json());
const gold = new Map();
app.post("/api/gold", (req, res) => { const u = req.get("x-user"); gold.set(u, (gold.get(u) || 0) + req.body.add); res.json({ gold: gold.get(u) }); });
app.use(express.static("public")); app.listen(3000);
```
