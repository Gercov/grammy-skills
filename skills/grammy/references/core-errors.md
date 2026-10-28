---
name: grammy-errors
description: Error handling with bot.catch, BotError/GrammyError/HttpError, error boundaries, and common Bot API error codes
---

# Error Handling

grammY catches every error thrown in middleware and wraps it in a `BotError` (`err.error` = original, `err.ctx` = context).

| Error | Meaning |
|---|---|
| `BotError` | Wrapper for anything thrown in middleware |
| `GrammyError` | Bot API returned `ok: false` (`error_code`, `description`, `parameters`) |
| `HttpError` | Bot API server unreachable (network) |

## Usage

```ts
import { BotError, GrammyError, HttpError } from "grammy";

bot.catch((err) => {
  const ctx = err.ctx;
  console.error(`Error while handling update ${ctx.update.update_id}:`);
  const e = err.error;
  if (e instanceof GrammyError) console.error("Request failed:", e.description);
  else if (e instanceof HttpError) console.error("Could not contact Telegram:", e);
  else console.error("Unknown error:", e);
});
```

- **Long polling (`bot.start()` / runner):** always install `bot.catch`. The default handler stops the bot and rethrows.
- **Webhooks:** errors propagate to your web framework; handle them there.

### Error boundaries

Fence a part of the middleware tree with its own handler:

```ts
const admin = new Composer<MyContext>();
admin.command("ban", mw);

bot.errorBoundary(async (err, next) => {
  console.error("admin module failed", err);
  // await next(); // optionally continue with middleware after the boundary
}).use(admin);

bot.use(laterMiddleware);
bot.catch(globalHandler); // outermost boundary
```

Rethrow inside a boundary handler to pass the error to the next outer boundary.

### Handling API errors locally

```ts
try {
  await ctx.api.sendMessage(userId, text);
} catch (e) {
  if (e instanceof GrammyError && e.error_code === 403) {
    await markUserBlocked(userId); // user blocked the bot
  } else throw e;
}
```

## Common Errors

| Code | Cause / fix |
|---|---|
| 400 `can't parse entities` | Broken `parse_mode` markup; the byte offset points to it. Prefer entities via the parse-mode plugin. |
| 401 / 404 on startup | Wrong token. |
| 403 `bot was blocked by the user` | Can't message them; clean up data. Detect proactively via `my_chat_member`. |
| 409 `terminated by other getUpdates request` | Bot running twice with long polling (or a webhook is set). Only one instance may poll. |
| 429 `retry after X` | Flood limit; use auto-retry (advanced-flood-limits). |
| `Cannot find type definition file for 'node-fetch'` | Set `skipLibCheck: true`. |

## Key Points

- `GrammyError` thrown in middleware still arrives wrapped in `BotError`.
- Unawaited promises bypass grammY's error handling entirely and become unhandled rejections.
- Error boundaries work the same for polling and webhooks.

<!--
Source references:
- https://grammy.dev/guide/errors
- https://grammy.dev/resources/faq
-->
