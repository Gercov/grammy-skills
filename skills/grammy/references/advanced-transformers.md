---
name: grammy-transformers
description: API transformer functions for outgoing requests, API flavors, mocking API calls in tests, and authoring middleware/transformer plugins
---

# API Transformers and Writing Plugins

Middleware handles incoming updates; a **transformer** handles outgoing API calls `(prev, method, payload, signal)`. It can modify payloads, retry, mock, or short-circuit. The last transformer is grammY's built-in caller.

## Usage

```ts
// Pass-through (always forward signal so bot.stop() can abort)
bot.api.config.use((prev, method, payload, signal) => prev(method, payload, signal));

// Logging
bot.api.config.use(async (prev, method, payload, signal) => {
  const t = Date.now();
  try { return await prev(method, payload, signal); }
  finally { console.log(method, Date.now() - t, "ms"); }
});

// Per-update only: installed on ctx.api, discarded after this update
bot.on("message", async (ctx, next) => {
  ctx.api.config.use(myTransformer);
  await next();
});
```

Transformers on `bot.api` are also applied to every `ctx.api`.

### Mocking in tests

```ts
const bot = new Bot("test-token", { botInfo: fakeBotInfo }); // no getMe
const calls: Array<{ method: string; payload: unknown }> = [];
bot.api.config.use((_prev, method, payload) => {
  calls.push({ method, payload });
  return Promise.resolve({ ok: true, result: true } as any);
});
registerHandlers(bot);
await bot.handleUpdate({ update_id: 1, message: fakeMessage });
expect(calls[0].method).toBe("sendMessage");
```

### API flavors

Context flavors can't change `bot.api` types; plugins adding API methods/result types use API flavors:

```ts
type MyApi = HydrateApiFlavor<FileApiFlavor<Api>>; // transformative; additive uses &
const bot = new Bot<MyContext, MyApi>(token);
```

## Authoring Plugins

**Middleware plugin** — a factory returning `Middleware<C>`, generic over the context:

```ts
import type { Context, Middleware } from "grammy";

export function onlyAccept<C extends Context>(word: string): Middleware<C> {
  return async (ctx, next) => {
    const name = ctx.from?.first_name;
    if (name === undefined || name.includes(word)) await next();
    else await ctx.reply(`Not talking to you.`);
  };
}
bot.use(onlyAccept("grammY"));
```

**Transformer plugin** — a factory returning `Transformer`:

```ts
import type { Transformer } from "grammy";

export function autoChatAction(): Transformer {
  return async (prev, method, payload, signal) => {
    let handle: ReturnType<typeof setInterval> | undefined;
    if (method === "sendDocument" && "chat_id" in payload) {
      const action = { chat_id: payload.chat_id, action: "upload_document" as const };
      handle = setInterval(() => prev("sendChatAction", action).catch(console.error), 5000);
    }
    try { return await prev(method, payload, signal); }
    finally { clearInterval(handle); }
  };
}
bot.api.config.use(autoChatAction());
```

Plugin conventions: export a flavor type for anything added to `ctx`; when reading sessions, `await ctx.session` (supports lazy sessions); official plugins are Deno-first and built for Node with `deno2node` (`deps.deno.ts` / `deps.node.ts`), see `grammyjs/plugin-template`.

## Key Points

- Retrying in a transformer re-sends the payload; stream-based `InputFile`s may already be consumed.
- Plugins installed via `bot.api.config.use` must be installed per-context (`ctx.api.config.use`) inside conversations.
- Known transformer plugins: auto-retry, throttler, files (`hydrateFiles`), hydrate (`hydrateApi`), chat-members (`hydrateChatMember`); menu uses one internally.

<!--
Source references:
- https://grammy.dev/advanced/transformers
- https://grammy.dev/plugins/guide
- https://grammy.dev/plugins/
- https://grammy.dev/advanced/deployment#testing
-->
