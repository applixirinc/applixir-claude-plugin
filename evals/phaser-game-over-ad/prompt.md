---
max_turns: 25
timeout_seconds: 600
allowed_tools: [Read, Glob, Grep, Skill]
tags: [engine, phaser]
---

My Phaser 3 game is in the `project` directory. Add an AppLixir rewarded ad on the game-over
screen: "Watch an ad to continue with an extra life". It's a session-only reward (the extra life
isn't saved anywhere), so no server is involved. I have an AppLixir key and ~15k DAU. The plan is
approved - skip the confirmation and give me the complete code changes in your reply (I'll paste
them in; you can't write files here).

The `project` files are pasted below (you can't open them directly):

`project/package.json`
```json
{ "name": "dino-dash", "dependencies": { "phaser": "^3.80.1" }, "devDependencies": { "vite": "^5.2.0" } }
```

`project/index.html`
```html
<!DOCTYPE html>
<html><head><meta charset="utf-8"><title>Dino Dash</title></head>
<body><div id="game"></div><script type="module" src="/src/main.js"></script></body></html>
```

`project/src/main.js`
```js
import Phaser from "phaser";
import { PlayScene } from "./scenes/PlayScene.js";
import { GameOverScene } from "./scenes/GameOverScene.js";
new Phaser.Game({ type: Phaser.AUTO, width: 800, height: 450, parent: "game", scene: [PlayScene, GameOverScene] });
```

`project/src/scenes/PlayScene.js`
```js
import Phaser from "phaser";
export class PlayScene extends Phaser.Scene {
  constructor() { super("PlayScene"); }
  init(data) { this.lives = data.lives ?? 1; }
  create() { this.add.text(20, 20, "Run!"); this.time.delayedCall(5000, () => this.scene.start("GameOverScene", { score: 120 })); }
}
```

`project/src/scenes/GameOverScene.js`
```js
import Phaser from "phaser";
export class GameOverScene extends Phaser.Scene {
  constructor() { super("GameOverScene"); }
  create(data) {
    this.add.text(300, 150, `Game over - ${data.score}`, { fontSize: "28px" });
    this.add.text(320, 250, "Restart", { backgroundColor: "#333", padding: { x: 12, y: 8 } })
      .setInteractive().on("pointerdown", () => this.scene.start("PlayScene", { lives: 1 }));
  }
}
```
