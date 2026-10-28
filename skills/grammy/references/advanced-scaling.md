---
name: grammy-scaling
description: Concurrent long polling with grammY runner, sequentialize to avoid session races, graceful shutdown, multi-core bot workers
---

# Scaling: grammY Runner (`@grammyjs/runner`)

`bot.start()` handles one update at a time. For throughput on long polling, use `run(bot)` (concurrent, default limit 500 in-flight updates, backpressure, timeouts). Webhooks are already concurrent across chats.

## Usage

```ts
import { Bot, type Context, session } from "grammy";
import { run, sequentialize } from "@grammyjs/runner";

const bot = new Bot<MyContext>(token);

function getSessionKey(ctx: Context) {
  return ctx.chat?.id.toString();
}
bot.use(sequentialize(getSessionKey));  // BEFORE session, same key fn
bot.use(session({ getSessionKey, initial }));
bot.on("message", (ctx) => ctx.reply("Got it"));

bot.catch(console.error);                 // used by run() as sink error handler
const runner = run(bot, {
  runner: { fetch: { allowed_updates: ["message", "callback_query"] } },
  // source: {...}, sink: { concurrency: 500, timeout: {...} }
});

const stop = () => runner.isRunning() && runner.stop();
process.once("SIGINT", stop);
process.once("SIGTERM", stop);
await runner.task(); // resolves when stopped
```

### Why `sequentialize`

Concurrent updates from the same chat race on session data (read A, read B, write A, write B → A's changes lost). `sequentialize(fn)` delays only updates sharing a constraint. Return several constraints to order across dimensions:

```ts
bot.use(sequentialize((ctx) =>
  [ctx.chat?.id.toString(), ctx.from?.id.toString()].filter((k) => k !== undefined)));
```

With multi sessions, return **all** session keys.

### Multi-core: bot workers (only > ~500 updates/s)

```ts
// bot.ts
import { distribute, run } from "@grammyjs/runner";
const bot = new Bot(token);
bot.use(sequentialize(getKey));   // optional
bot.on(["message", "callback_query"], distribute(__dirname + "/worker", { count: 4 }));
run(bot);

// worker.ts — all handlers and plugins live here
import { BotWorker } from "@grammyjs/runner";
const worker = new BotWorker<MyContext>(token);
worker.use(session(...));
worker.on("message", (ctx) => ctx.reply("yay"));
```

Deno: `distribute(new URL("./worker.ts", import.meta.url))`. Only raw updates are sent to workers; context properties set in `bot.ts` don't transfer. Don't exceed physical cores.

## Key Points

- Concurrency breaks in-chat ordering unless sequentialized; plugins that store state need it.
- Runner confirms update offsets early: a hard kill can lose up to 100 updates; under load, duplicates are possible. For strict guarantees, build a queue pipeline from runner sources/sinks.
- Graceful shutdown matters for both `bot.stop()` and the runner, so offsets are synced on deploys.
- Many bots never need the runner; sequential `bot.start()` is easier to debug.

<!--
Source references:
- https://grammy.dev/plugins/runner
- https://grammy.dev/advanced/scaling
- https://grammy.dev/advanced/reliability
-->
