---
name: grammy
description: grammY, the TypeScript framework for Telegram bots on Node.js, Deno, Bun, and serverless. Use when writing or debugging Telegram bots with grammY - middleware, filter queries, context, keyboards, files, sessions, conversations, menus, i18n, webhooks, the runner, and rate limits.
metadata:
  author: Gercov
  version: "2026.10.29"
  source: Generated from https://github.com/grammyjs/website, scripts located at https://github.com/antfu/skills
---

> The skill is based on grammY v1.46.0 (Telegram Bot API 10.2), generated at 2026-10-29.

grammY is a Telegram Bot API framework. A `Bot` is a `Composer` of middleware. Every incoming update becomes one `Context` (`ctx`) that flows down the middleware tree; handlers reply via `ctx` actions or `ctx.api`. It is installed from npm as `grammy` (Deno: `npm:grammy`), and official plugins are published as `@grammyjs/*`.

```ts
import { Bot, type Context, GrammyError, session, type SessionFlavor } from "grammy";

type MyContext = Context & SessionFlavor<{ count: number }>;
const bot = new Bot<MyContext>(process.env.BOT_TOKEN!);

bot.use(session({ initial: () => ({ count: 0 }) }));            // plugins first
bot.command("start", (ctx) => ctx.reply(`Hi! Arg: ${ctx.match}`)); // specific handlers
bot.on("message:text", async (ctx) => {                          // generic handlers last
  ctx.session.count++;
  await ctx.reply(ctx.msg.text, { reply_parameters: { message_id: ctx.msg.message_id } });
});
bot.on("callback_query:data", (ctx) => ctx.answerCallbackQuery()); // catch-all for buttons

bot.catch((err) => console.error(err.ctx.update.update_id, err.error));
bot.start(); // long polling. For webhooks, use webhookCallback(bot, adapter) instead
```

Rules that prevent most bugs:

- `await` every promise, especially `next()`. Lint with `no-floating-promises`.
- Registration order matters, because the first handler that doesn't call `next()` ends processing.
- Pass the custom context type everywhere it's used: `Bot<MyContext>`, `Composer<MyContext>`, `Menu<MyContext>`, and so on.
- Install `bot.catch`. Add `auto-retry` for 429 errors. Keep webhook handlers under about 10 seconds.
- Multi-step dialogs go in conversations, and inside a conversation, side effects must be wrapped in `conversation.external`.

## Core References

| Topic | Description | Reference |
|-------|-------------|-----------|
| Middleware | `next()`, ordering, Composer trees, `filter`, `branch`, `route`, `fork`, typed extracted handlers | [core-middleware](references/core-middleware.md) |
| Filter Queries | `bot.on()` query language, shortcuts, AND/OR, type narrowing, `matchFilter` | [core-filter-queries](references/core-filter-queries.md) |
| Handlers | `command`, `hears`, `callbackQuery`, `chatType`, `ctx.match`, deep links, has checks | [core-handlers](references/core-handlers.md) |
| Context | `ctx` shortcuts and actions, custom properties, context flavors | [core-context](references/core-context.md) |
| Bot API | `bot.api`/`ctx.api`, `Other` options, raw calls, types, bot config, local Bot API server, proxy | [core-api](references/core-api.md) |
| Errors | `bot.catch`, `BotError`/`GrammyError`/`HttpError`, error boundaries, common error codes | [core-errors](references/core-errors.md) |
| Deployment | Long polling vs webhooks, adapters, webhook timeouts, serverless, shutdown, testing, checklist | [core-deployment](references/core-deployment.md) |

## Features

| Topic | Description | Reference |
|-------|-------------|-----------|
| Keyboards | `InlineKeyboard`, `Keyboard`, callback queries, removing keyboards | [features-keyboards](references/features-keyboards.md) |
| Files | `file_id`, `getFile`, `InputFile`, files plugin downloads, media groups, size limits | [features-files](references/features-files.md) |
| Formatting | HTML and MarkdownV2 `parse_mode`, entities, the parse-mode plugin (`fmt`, `FormattedString`) | [features-formatting](references/features-formatting.md) |
| Inline Queries | `inlineQuery`, `InlineQueryResultBuilder`, `answerInlineQuery`, chosen results | [features-inline-queries](references/features-inline-queries.md) |
| Reactions | `ctx.react`, `bot.reaction`, `ctx.reactions()`, reaction count updates | [features-reactions](references/features-reactions.md) |
| Business | Telegram Business connections and managed chats | [features-business](references/features-business.md) |

## Plugins

| Topic | Description | Reference |
|-------|-------------|-----------|
| Sessions | Built-in `session`, storage adapters, keys, multi and lazy sessions, migrations | [plugins-session](references/plugins-session.md) |
| Conversations | Multi-step dialogs, the replay model, `conversation.external`, forms, persistence, parallel conversations | [plugins-conversations](references/plugins-conversations.md) |
| Menus | Interactive button menus, dynamic ranges, navigation, outdated menus | [plugins-menu](references/plugins-menu.md) |
| Commands | `CommandGroup`, synced and scoped command menus, localization, "did you mean" suggestions | [plugins-commands](references/plugins-commands.md) |
| i18n | Fluent translations, `ctx.t`, locale negotiation | [plugins-i18n](references/plugins-i18n.md) |
| Utilities | hydrate, chat-members, stream (LLM output), router, ratelimiter, emoji, autoquote, stateless-question | [plugins-utilities](references/plugins-utilities.md) |

## Advanced

| Topic | Description | Reference |
|-------|-------------|-----------|
| Scaling | grammY runner, `sequentialize`, worker threads, delivery guarantees | [advanced-scaling](references/advanced-scaling.md) |
| Flood Limits | auto-retry, broadcasting, paid broadcasts, why throttling is discouraged | [advanced-flood-limits](references/advanced-flood-limits.md) |
| Transformers | Outgoing API transformers, API flavors, test mocking, writing plugins | [advanced-transformers](references/advanced-transformers.md) |
