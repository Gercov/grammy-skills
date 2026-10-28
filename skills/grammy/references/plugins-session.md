---
name: grammy-session
description: Built-in session plugin - initial data, session keys, storage adapters, multi sessions, lazy sessions, timeouts and migrations
---

# Sessions (built-in)

Loads per-chat data into `ctx.session` before downstream middleware and writes it back afterwards. Bots can't read chat history, so anything you need later must be stored.

## Usage

```ts
import { Bot, type Context, session, type SessionFlavor } from "grammy";

interface SessionData { pizzaCount: number }
type MyContext = Context & SessionFlavor<SessionData>;

const bot = new Bot<MyContext>(token);
bot.use(session({
  initial: (): SessionData => ({ pizzaCount: 0 }), // MUST return a fresh object
  storage: adapter,                                  // default: in-memory (lost on restart)
}));

bot.hears(/🍕/, (ctx) => ctx.session.pizzaCount++);
bot.command("hunger", (ctx) => ctx.reply(`Level ${ctx.session.pizzaCount}`));
```

Never return a shared object from `initial` (`() => sharedObj`) — chats would share state. Without `initial`, reading `ctx.session` for a new chat throws.

### Session keys

```ts
session({ getSessionKey: (ctx) => ctx.chat?.id.toString() }); // default: per chat
session({ getSessionKey: (ctx) => ctx.from?.id.toString() }); // per user (across chats)
session({ getSessionKey: (ctx) => ctx.from && ctx.chat ? `${ctx.from.id}/${ctx.chat.id}` : undefined });
session({ getSessionKey: (ctx) => ctx.from?.id, prefix: "user-" });
```

Returning `undefined` → `ctx.session` is `undefined` (e.g. inline queries, polls have no chat).

### Storage adapters

```ts
import { MemorySessionStorage } from "grammy";            // default
import { freeStorage } from "@grammyjs/storage-free";      // hobby projects, auth via token
import { FileAdapter } from "@grammyjs/storage-file";
// Others: npm i @grammyjs/storage-<name> (redis, mongodb, psql, supabase, prisma, ...)
// Deno: npm:@grammyjs/storages/<name>/src

session({ initial, storage: freeStorage<SessionData>(bot.token) });
```

Custom adapter: any object implementing `StorageAdapter<T>` — `read(key)`, `write(key, value)`, `delete(key)` (optionally `has`, `readAllKeys`, ...).

### Multi sessions (different storage/key per field)

```ts
bot.use(session({
  type: "multi",
  user: { getSessionKey: (ctx) => ctx.from?.id.toString(), initial: () => ({ lang: "en" }), storage: redis },
  chat: { initial: () => ({ count: 0 }) },
  cache: {}, // all defaults
}));
// ctx.session.user, ctx.session.chat, ctx.session.cache — SessionFlavor type unchanged
```

### Lazy sessions (skip DB I/O for updates that don't touch the session)

```ts
import { lazySession, type LazySessionFlavor } from "grammy";
type MyContext = Context & LazySessionFlavor<SessionData>;
bot.use(lazySession({ initial }));

bot.command("stats", async (ctx) => {
  const s = await ctx.session; // read triggered only on access
  s.count++;
});
```

Plugins reading sessions should always `await ctx.session` so they work with both flavors.

### Storage enhancements: timeouts and migrations

```ts
import { enhanceStorage, type Enhance, MemorySessionStorage } from "grammy";

bot.use(session({
  initial: (): SessionData => ({ pets: [] }),
  storage: enhanceStorage({
    storage: new MemorySessionStorage<Enhance<SessionData>>(), // annotate Enhance<T>
    millisecondsToLive: 30 * 60 * 1000,  // expire if untouched (deleted on next read)
    migrations: {
      1: (old: { petNames: string[] }) => ({ pets: old.petNames.map((name) => ({ name })) }),
      2: addFavoriteFlag,                // any increasing numbers
    },
  }),
}));
```

## Key Points

- Sessions = small per-chat JS data used only by the bot (< ~3 MB/chat). Use a real DB for shared/large/transactional data.
- Concurrency: with runner, `sequentialize` using the **same** key function (return all keys for multi sessions). On webhooks, avoid custom `getSessionKey` — Telegram only serializes per chat.
- Group → supergroup migration changes the chat id, so per-chat session data effectively resets once (conversations too).
- Install `session()` before plugins that depend on it (i18n with `useSession`, router, menus reading session).
- Conversations can't use `ctx.session` inside the builder; use `conversation.external` (plugins-conversations).

<!--
Source references:
- https://grammy.dev/plugins/session
- https://grammy.dev/advanced/scaling#concurrency-is-hard
-->
