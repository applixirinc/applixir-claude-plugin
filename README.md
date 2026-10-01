# AppLixir rewarded video plugin for Claude

Add [AppLixir](https://www.applixir.com) rewarded video ads to a browser game and get a correct, working integration in one session:
- the player loads;
- ads open only when the player chooses to watch;
- the reward is granted once, only on a completed view, and verified on your server;
- consent is handled;
- no-fill and ad blockers fail gracefully.

## What it does

- **Skill `applixir-rewarded-video`** runs the full workflow:
  1. scope check;
  2. account and eligibility;
  3. engine detection;
  4. a plan you approve before any edit;
  5. implementation;
  6. server-side reward verification;
  7. consent;
  8. a test checklist;
  9. a go-live checklist.

  It ships reference docs for the SDK API, each engine, server verification (Node, Python, .NET), TCF consent and troubleshooting.
- **`/applixir-integrate [context]`** runs that workflow on the current project, e.g. `/applixir-integrate Unity, put the button on the shop screen`.
- **`/applixir-doctor [symptom]`** is a read-only audit of an existing integration. It gives pass/fail with file and line evidence across:
  - user-triggered show;
  - callback handling;
  - server verification;
  - idempotency;
  - consent;
  - key handling;
  - no-fill handling.

  It proposes fixes and waits for your approval before editing.

Or just ask in plain language: *"add a watch-an-ad-for-coins button to my game"*, *"my AppLixir reward fires twice"*.

## Supported engines

- HTML5 / vanilla JavaScript, Phaser, PixiJS, Three.js, Cocos web builds
- React web (Vite, Next.js, CRA)
- Unity WebGL (Unity Web), via a `.jslib` bridge

Native iOS/Android apps and desktop apps (Electron, CEF, Steam builds) aren't supported. AppLixir runs in browsers only.

## Requirements

- An AppLixir publisher account and API key. Sign up at https://client.applixir.com/register.
- At least **100,000 rewarded-ad impressions a month**. Below that, the plugin explains the threshold and points you to signup instead of doing a full integration.
- A game served over https from the domain registered in your AppLixir dashboard.
- For persistent rewards, a server endpoint for AppLixir's reward callback. The plugin can add a minimal one.

## What the plugin runs and sends

Nothing. It contains only Markdown instructions and reference docs: no scripts, hooks, MCP servers or network calls. It only edits files in your project, after you approve a plan. See [PRIVACY.md](PRIVACY.md).

## Support

support@applixir.com · Docs: https://support.applixir.com · SDK examples: https://github.com/applixirinc/applixir-integration

## License

MIT, see [LICENSE](LICENSE).
