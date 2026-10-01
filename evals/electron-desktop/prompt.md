---
max_turns: 25
timeout_seconds: 600
allowed_tools: [Read, Glob, Grep, Skill]
tags: [engine, desktop]
---

Our Steam game is an Electron app (in `project`). We also run the web version on starminer.io,
which is registered with AppLixir, and have ~20k DAU. Can I add AppLixir rewarded ads to the
desktop build by just adding the script tag to game/index.html? Plan approved, show me the code.

The `project` files are pasted below (you can't open them directly):

`project/package.json`
```json
{ "name": "star-miner", "main": "main.js", "devDependencies": { "electron": "^31.0.0" } }
```

`project/main.js`
```js
const { app, BrowserWindow } = require("electron");
app.whenReady().then(() => {
  const win = new BrowserWindow({ width: 1280, height: 720 });
  win.loadFile("game/index.html");
});
```

`project/game/index.html`
```html
<!DOCTYPE html><html><body><canvas id="c"></canvas><button id="bonus">Bonus</button><script src="game.js"></script></body></html>
```

`project/game/game.js`
```js
document.getElementById("bonus").onclick = () => {/* TODO rewarded ad */};
```
