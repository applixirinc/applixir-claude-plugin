---
max_turns: 25
timeout_seconds: 600
allowed_tools: [Read, Glob, Grep, Skill]
tags: [verification]
---

Add an AppLixir rewarded ad to my word game in `project` that gives the player 5 hints. ~30k DAU,
I have my key. Hints are saved in localStorage and are also something people pay for. Plan
approved, just give me the code.

The `project` files are pasted below (you can't open them directly):

`project/index.html`
```html
<!DOCTYPE html>
<html><head><title>Word Rush</title></head>
<body><div id="board"></div><div>Hints: <span id="hints"></span></div><button id="shop">Buy hint pack ($0.99)</button><script src="app.js"></script></body></html>
```

`project/app.js`
```js
// Static site on Netlify, no backend. Hints persist in localStorage and can also be bought.
let hints = Number(localStorage.getItem("hints") || 3);
const render = () => (document.getElementById("hints").textContent = hints);
function addHints(n) { hints += n; localStorage.setItem("hints", hints); render(); }
render();
```

`project/netlify.toml`
```toml
[build]
  publish = "."
```
