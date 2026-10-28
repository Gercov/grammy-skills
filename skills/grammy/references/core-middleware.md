---
name: grammy-middleware
description: grammY middleware model - next(), ordering, Composer trees, filter/branch/fork/route, extracting typed handlers
---

# Middleware and Composer

Every handler passed to `bot.use`, `bot.on`, `bot.command`, etc. is middleware: `(ctx, next) => unknown`. `Bot extends Composer`, and every Composer method is built on `use`.

## Usage

```ts
import { Bot, Composer, type Context, type NextFunction } from "grammy";

const bot = new Bot<MyContext>(token);

// Runs for every update; wraps everything installed after it.
bot.use(async (ctx, next) => {
  const start = Date.now();
  await next(); // ALWAYS await next()
  console.log(`update ${ctx.update.update_id} took ${Date.now() - start} ms`);
});

bot.command("start", (ctx) => ctx.reply("Started")); // must come before ":text"
bot.on(":text", (ctx) => ctx.reply("Text!"));          // does not call next -> stops here
```

Order matters: an update goes down the stack until a handler doesn't call `next`. Registering `bot.on(":text")` before `bot.command("start")` means `/start` is swallowed by the text handler.

### Composer trees (splitting a bot into modules)

```ts
// features/todo.ts
export const todo = new Composer<MyContext>(); // pass your custom context type
todo.command("add", async (ctx) => { /* ... */ });
todo.on("callback_query:data", async (ctx, next) => { /* ... */ });

// bot.ts
bot.use(todo); // can still add handlers to `todo` after this, they will run
```

Chaining builds a tree; guards apply to everything chained after them:

```ts
// B is also guarded by predicate 1 (filter returns a new Composer)
bot.filter((ctx) => ctx.from?.id === ADMIN_ID).use(adminModule);

const pm = bot.chatType("private"); // scoped composer
pm.command("settings", handler);

bot.on(":photo").on("::hashtag", handler);   // AND of filters
bot.branch(predicate, trueMw, falseMw);       // if/else
bot.drop(predicate, mw);                       // inverse of filter
bot.fork(mw);                                  // run concurrently, don't wait
bot.lazy((ctx) => pickMiddleware(ctx));        // decide middleware per update
bot.route((ctx) => ctx.chat?.type, { private: pmMw, group: groupMw }, fallback);
bot.errorBoundary(errHandler).use(riskyModule); // see core-errors
```

`filter`/`drop` predicates may be `async` (e.g. `await ctx.getAuthor()`).

### Typing extracted handlers

When a handler lives outside a composer chain, use the exported narrowed types:

```ts
import type { CallbackQueryMiddleware, CommandContext, Filter, NextFunction } from "grammy";

function onStart(ctx: CommandContext<MyContext>, next: NextFunction) { /* ... */ }
const onClick: CallbackQueryMiddleware<MyContext> = (ctx) => ctx.answerCallbackQuery();
function onText(ctx: Filter<MyContext, ":text">) { ctx.msg.text; }

bot.command("start", onStart);
bot.callbackQuery("click", onClick);
bot.on(":text", onText);
```

## Key Points

- `await next()` (and every promise) — un-awaited `next()` breaks ordering, error handling (unhandled rejections), sessions (data loss), and runner backpressure. Enable `@typescript-eslint/no-floating-promises`.
- `next` takes no arguments; to fail, `throw`. Middleware may be sync or async; arity is irrelevant.
- One context object per update, shared by all middleware — upstream middleware can set properties for downstream (see core-context).
- Install plugins like `session()` with `bot.use` *before* handlers that need them.
- A module composer should be typed with the bot's custom context: `new Composer<MyContext>()`.
- `fork` is fire-and-forget concurrency inside one update; prefer the runner for concurrent update processing.

<!--
Source references:
- https://grammy.dev/guide/middleware
- https://grammy.dev/advanced/middleware
- https://grammy.dev/advanced/structuring
-->
