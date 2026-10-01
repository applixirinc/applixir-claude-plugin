# Server-side reward verification

**Required for any reward that persists or has value** (currency, lives saved to
an account, items, progression). The client's `status.type === "complete"` is
optimistic UI only; `applixir-integration` `CLAUDE.md` calls the server callback
*"the source of truth for granting persistent rewards"*.

## Flow

```
player clicks "Watch ad" ──► SDK plays ad (options.userId = your player id)
                              │ complete
                              ├──► client: optimistic "reward on its way…"
                              ▼
                     AppLixir servers
                              │ HTTPS GET  your-callback-url?gameApiKey=…&gameId=…&userId=…&tid=…&signature=…
                              ▼
                 YOUR server: verify → dedupe tid → credit userId → 200
                              ▼
client re-fetches balance from YOUR server (never trusts its own "complete")
```

## The callback request

Configure the URL and secret in **AppLixir dashboard → Callbacks**. The setting can
be per game or account-wide; a per-game URL/secret/mode overrides the account's.

| | |
|---|---|
| Method | **GET** |
| Timeout | 10 s |
| Retries | **None.** One attempt; a failure is not re-sent. |
| Success | any `2xx` |

Query parameters:

| Param | Always? | Meaning |
|---|---|---|
| `gameApiKey` | yes | The API key the ad ran under. Check it's yours. |
| `gameId` | yes | AppLixir's id for the game |
| `secretKey` | yes | Your callback secret, **in plaintext** (legacy) |
| `userId` | if the SDK was given `options.userId` | Your player id, the one to credit |
| `customData` | if the SDK was given `options.customData` | URL-encoded JSON. It comes from the client, so don't trust amounts in it. |
| `tid` | only in mode **`md5AndTid`** | Unique transaction id (32 hex chars), your idempotency key |
| `signature` | in modes `md5Only` / `md5AndTid` | `lowercase hex MD5(gameApiKey + gameId + userId + tid + secret)`, with `""` for any absent value |

**Set the dashboard's callback mode to `md5AndTid`.** Without `tid` there's no way to
reject a replayed URL.

## What your endpoint must do

1. **Authenticate.** If `signature` is present, recompute the MD5 and compare in constant time. Otherwise compare `secretKey` to your secret in constant time. Also check `gameApiKey` equals your key.
2. **Require `userId`.** No user, no credit. Answer `200` anyway so nothing is retried or logged as an error, but grant nothing.
3. **Dedupe on `tid`.** Insert `tid` into a table with a **unique constraint** in the **same transaction** as the credit. On a duplicate-key error, do nothing and return `200`. That's what makes the reward exactly-once even under replay or concurrent duplicates. Keep `tid`s permanently (they're small).
4. **Decide the amount server-side** from your own config (e.g. `REWARD_PER_AD = 3`), keyed by a placement name if you use `customData`. Never take an amount from the request.
5. **Cap abuse.** e.g. a per-user daily maximum or minimum interval. AppLixir's dashboard has fields for this, but don't rely on them alone.
6. **Return `200` fast.** There's no retry, so do the DB write inline but keep it quick.
7. **Don't log the query string.** It contains your secret. Scrub `secretKey` from access logs and APM.
8. **HTTPS only.**

Load the API key and callback secret through **the project's existing secrets/config mechanism** (a secrets manager, a config module, the hosting platform's secret settings). The examples below call a `loadConfig()` you replace with that. Never hard-code the secret or commit it.

## Node.js (Express)

```js
// server.js: npm i express better-sqlite3
const crypto = require("crypto");
const express = require("express");
const Database = require("better-sqlite3");

// Replace with your project's secrets/config loader (secrets manager, config module, …).
const { applixirApiKey: ourApiKey, applixirCallbackSecret: callbackSecret } = require("./config").loadConfig();
const REWARD_PER_AD = 3;
const DAILY_CAP = 20;
if (!ourApiKey || !callbackSecret) throw new Error("AppLixir API key / callback secret not configured");

const db = new Database("game.db");
db.exec(`
  CREATE TABLE IF NOT EXISTS players (id TEXT PRIMARY KEY, lives INTEGER NOT NULL DEFAULT 0);
  CREATE TABLE IF NOT EXISTS ad_rewards (
    tid TEXT PRIMARY KEY, user_id TEXT NOT NULL, amount INTEGER NOT NULL,
    created_at TEXT NOT NULL DEFAULT (datetime('now')));
`);

function safeEqual(a, b) {
  const x = Buffer.from(String(a)), y = Buffer.from(String(b));
  return x.length === y.length && crypto.timingSafeEqual(x, y);
}

function authentic(q) {
  if (!safeEqual(q.gameApiKey || "", ourApiKey)) return false;
  if (q.signature) {
    const expected = crypto.createHash("md5")
      .update([q.gameApiKey, q.gameId, q.userId, q.tid].map((v) => v || "").join("") + callbackSecret)
      .digest("hex");
    return safeEqual(q.signature.toLowerCase(), expected);
  }
  return safeEqual(q.secretKey || "", callbackSecret);
}

const credit = db.transaction((tid, userId, amount) => {
  const today = db.prepare(
    "SELECT COUNT(*) n FROM ad_rewards WHERE user_id = ? AND created_at >= date('now')").get(userId).n;
  if (today >= DAILY_CAP) return "capped";
  db.prepare("INSERT INTO ad_rewards (tid, user_id, amount) VALUES (?, ?, ?)").run(tid, userId, amount); // throws on replay
  db.prepare("INSERT INTO players (id, lives) VALUES (?, ?) ON CONFLICT(id) DO UPDATE SET lives = lives + excluded.lives")
    .run(userId, amount);
  return "credited";
});

const app = express();

app.get("/applixir/callback", (req, res) => {
  const q = req.query;
  if (!authentic(q)) return res.status(403).send("forbidden");
  if (!q.userId) return res.status(200).send("no user");
  if (!q.tid) {
    // Mode lacks tid: cannot dedupe. Switch the dashboard to md5AndTid.
    console.error("AppLixir callback without tid; refusing to credit (replayable)");
    return res.status(200).send("no tid");
  }
  try {
    res.status(200).send(credit(q.tid, String(q.userId), REWARD_PER_AD));
  } catch (e) {
    if (String(e.code).startsWith("SQLITE_CONSTRAINT")) return res.status(200).send("duplicate");
    console.error("credit failed", e.message);
    res.status(500).send("error");
  }
});

// The game reads its authoritative balance here (protect with YOUR session auth).
app.get("/api/me", (req, res) => {
  const userId = req.get("x-player-id"); // replace with real session auth
  const row = db.prepare("SELECT lives FROM players WHERE id = ?").get(userId);
  res.json({ lives: row ? row.lives : 0 });
});

app.listen(3000);
```

## Python (FastAPI)

```python
# main.py: pip install fastapi uvicorn
import hashlib, hmac, os, sqlite3
from fastapi import FastAPI, Request
from fastapi.responses import PlainTextResponse

from config import load_config  # replace with your project's secrets/config loader
_cfg = load_config()
our_api_key, callback_secret = _cfg["applixir_api_key"], _cfg["applixir_callback_secret"]
REWARD_PER_AD, DAILY_CAP = 3, 20

db = sqlite3.connect("game.db", check_same_thread=False, isolation_level=None)
db.executescript("""
CREATE TABLE IF NOT EXISTS players (id TEXT PRIMARY KEY, lives INTEGER NOT NULL DEFAULT 0);
CREATE TABLE IF NOT EXISTS ad_rewards (tid TEXT PRIMARY KEY, user_id TEXT NOT NULL,
  amount INTEGER NOT NULL, created_at TEXT NOT NULL DEFAULT (datetime('now')));
""")

def authentic(q) -> bool:
    if not hmac.compare_digest(q.get("gameApiKey", ""), our_api_key):
        return False
    sig = q.get("signature")
    if sig:
        raw = "".join(q.get(k, "") for k in ("gameApiKey", "gameId", "userId", "tid")) + callback_secret
        return hmac.compare_digest(sig.lower(), hashlib.md5(raw.encode()).hexdigest())
    return hmac.compare_digest(q.get("secretKey", ""), callback_secret)

app = FastAPI()

@app.get("/applixir/callback", response_class=PlainTextResponse)
def callback(request: Request):
    q = dict(request.query_params)
    if not authentic(q):
        return PlainTextResponse("forbidden", status_code=403)
    user_id, tid = q.get("userId"), q.get("tid")
    if not user_id:
        return "no user"
    if not tid:
        return "no tid"  # switch dashboard mode to md5AndTid; never credit without a dedupe key
    try:
        db.execute("BEGIN IMMEDIATE")
        (today,) = db.execute(
            "SELECT COUNT(*) FROM ad_rewards WHERE user_id=? AND created_at >= date('now')", (user_id,)).fetchone()
        if today >= DAILY_CAP:
            db.execute("ROLLBACK"); return "capped"
        db.execute("INSERT INTO ad_rewards (tid, user_id, amount) VALUES (?,?,?)", (tid, user_id, REWARD_PER_AD))
        db.execute("INSERT INTO players (id, lives) VALUES (?, ?) "
                   "ON CONFLICT(id) DO UPDATE SET lives = lives + excluded.lives", (user_id, REWARD_PER_AD))
        db.execute("COMMIT")
        return "credited"
    except sqlite3.IntegrityError:
        db.execute("ROLLBACK"); return "duplicate"
```

## .NET (minimal API, C#)

```csharp
// Program.cs: dotnet new web; dotnet add package Microsoft.Data.Sqlite
using System.Security.Cryptography;
using System.Text;
using Microsoft.Data.Sqlite;

// Bind from your configuration (appsettings + user-secrets / a secrets manager), section "AppLixir".
var builder = WebApplication.CreateBuilder(args);
var apiKey = builder.Configuration["AppLixir:ApiKey"] ?? throw new("AppLixir:ApiKey not configured");
var secret = builder.Configuration["AppLixir:CallbackSecret"] ?? throw new("AppLixir:CallbackSecret not configured");
const int RewardPerAd = 3, DailyCap = 20;
const string Cs = "Data Source=game.db";

using (var c = new SqliteConnection(Cs)) {
    c.Open();
    new SqliteCommand(@"CREATE TABLE IF NOT EXISTS players (id TEXT PRIMARY KEY, lives INTEGER NOT NULL DEFAULT 0);
      CREATE TABLE IF NOT EXISTS ad_rewards (tid TEXT PRIMARY KEY, user_id TEXT NOT NULL, amount INTEGER NOT NULL,
      created_at TEXT NOT NULL DEFAULT (datetime('now')));", c).ExecuteNonQuery();
}

static bool Eq(string a, string b) =>
    CryptographicOperations.FixedTimeEquals(Encoding.UTF8.GetBytes(a), Encoding.UTF8.GetBytes(b));

var app = builder.Build();

app.MapGet("/applixir/callback", (HttpRequest req) =>
{
    string Q(string k) => req.Query[k].ToString();
    if (!Eq(Q("gameApiKey"), apiKey)) return Results.StatusCode(403);
    var sig = Q("signature");
    var ok = sig.Length > 0
        ? Eq(sig.ToLowerInvariant(), Convert.ToHexString(MD5.HashData(Encoding.UTF8.GetBytes(
              Q("gameApiKey") + Q("gameId") + Q("userId") + Q("tid") + secret))).ToLowerInvariant())
        : Eq(Q("secretKey"), secret);
    if (!ok) return Results.StatusCode(403);

    var userId = Q("userId"); var tid = Q("tid");
    if (userId.Length == 0) return Results.Text("no user");
    if (tid.Length == 0) return Results.Text("no tid"); // switch dashboard to md5AndTid

    using var c = new SqliteConnection(Cs); c.Open();
    using var tx = c.BeginTransaction();
    var count = new SqliteCommand("SELECT COUNT(*) FROM ad_rewards WHERE user_id=$u AND created_at >= date('now')", c, tx);
    count.Parameters.AddWithValue("$u", userId);
    if ((long)count.ExecuteScalar()! >= DailyCap) return Results.Text("capped");
    try {
        var ins = new SqliteCommand("INSERT INTO ad_rewards (tid,user_id,amount) VALUES ($t,$u,$a)", c, tx);
        ins.Parameters.AddWithValue("$t", tid); ins.Parameters.AddWithValue("$u", userId); ins.Parameters.AddWithValue("$a", RewardPerAd);
        ins.ExecuteNonQuery();
    } catch (SqliteException e) when (e.SqliteErrorCode == 19) { return Results.Text("duplicate"); } // constraint
    var upd = new SqliteCommand("INSERT INTO players (id,lives) VALUES ($u,$a) ON CONFLICT(id) DO UPDATE SET lives = lives + excluded.lives", c, tx);
    upd.Parameters.AddWithValue("$u", userId); upd.Parameters.AddWithValue("$a", RewardPerAd);
    upd.ExecuteNonQuery();
    tx.Commit();
    return Results.Text("credited");
});

app.Run();
```

These use SQLite so they run as-is. In a real project, use the project's existing
database and its user/session model. Keep the **unique `tid` + credit in one
transaction** shape.

## No backend at all?

Explain the trade-off, then offer one of these:

1. **Cosmetic or session-only rewards** (an extra life in this run, a hint): client-side `complete` is acceptable. State in a code comment that it's forgeable.
2. **Anything that persists or has value:** add the minimal service above (one file plus SQLite). It deploys to any Node/Python/.NET host. Serverless works too: one function for `/applixir/callback` with a managed DB that supports a unique key.

Don't silently ship persistent rewards from the client.

## Testing the endpoint before ads run

Compute a signature locally and hit your endpoint twice. The second call must
return `duplicate`. Type your own values in place of the placeholders.

```bash
# Run against your server locally first (e.g. http://localhost:3000).
BASE=http://localhost:3000/applixir/callback
K=YOUR-API-KEY; G=YOUR-GAME-ID; U=player-1; T=$(openssl rand -hex 16)
SIG=$(printf "%s" "$K$G$U$T" "<paste-your-callback-secret>" | md5sum | cut -d' ' -f1)
curl "$BASE?gameApiKey=$K&gameId=$G&userId=$U&tid=$T&signature=$SIG"
curl "$BASE?gameApiKey=$K&gameId=$G&userId=$U&tid=$T&signature=$SIG"   # → duplicate
```
