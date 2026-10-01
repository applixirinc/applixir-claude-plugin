---
max_turns: 25
timeout_seconds: 600
allowed_tools: [Read, Glob, Grep, Skill]
tags: [engine, html5]
---

I want to add a "watch an ad for 25 coins" button to my browser game. The project is in the
`project` directory you have access to. I already have an AppLixir account and API key, and
about 40,000 daily players. What's the plan?

The `project` files are pasted below (you can't open them directly):

`project/index.html`
```html
<!DOCTYPE html>
<html>
<head><meta charset="utf-8"><title>Gem Smash</title><link rel="stylesheet" href="style.css"></head>
<body>
  <canvas id="game" width="480" height="640"></canvas>
  <div id="hud">Coins: <span id="coins">0</span></div>
  <div id="shop" class="hidden">
    <h2>Shop</h2>
    <button id="buy-bomb">Bomb - 50 coins</button>
  </div>
  <script src="game.js"></script>
</body>
</html>
```

`project/game.js`
```js
// Gem Smash - vanilla canvas game
const state = { coins: 0, running: true };
async function loadProfile() {
  const res = await fetch("/api/profile");
  const p = await res.json();
  state.coins = p.coins; renderHud();
}
function addCoins(n) { state.coins += n; renderHud(); fetch("/api/coins", { method: "POST", body: JSON.stringify({ delta: n }) }); }
function renderHud() { document.getElementById("coins").textContent = state.coins; }
function loop() { if (state.running) { /* draw */ } requestAnimationFrame(loop); }
document.getElementById("buy-bomb").onclick = () => { if (state.coins >= 50) addCoins(-50); };
loadProfile(); loop();
```

`project/server.js`
```js
// Express backend for Gem Smash
const express = require("express");
const app = express();
app.use(express.json());
const users = new Map(); // userId -> { coins }
app.get("/api/profile", (req, res) => res.json(users.get(req.get("x-user")) || { coins: 0 }));
app.post("/api/coins", (req, res) => { /* TODO validate */ res.sendStatus(204); });
app.listen(3000);
```

`project/package.json`
```json
{ "name": "gem-smash", "dependencies": { "express": "^4.19.0" } }
```
