---
name: grammy-deployment
description: Long polling vs webhooks, webhookCallback adapters and timeouts, serverless setup, allowed_updates, graceful shutdown, delivery guarantees, testing
---

# Running the Bot: Long Polling vs Webhooks

Middleware is identical for both; only the entry point changes. **Default to long polling** unless the platform is serverless/autoscaling or you need scale-to-zero.

## Long Polling

```ts
bot.catch(console.error);
process.once("SIGINT", () => bot.stop());   // Deno: Deno.addSignalListener
process.once("SIGTERM", () => bot.stop());
await bot.start({
  drop_pending_updates: true,               // optional: skip backlog on start
  allowed_updates: ["message", "callback_query", "message_reaction"],
  onStart: (me) => console.log(`@${me.username} started`),
});
```

`bot.start()` processes updates **sequentially** (simple, predictable). For concurrency use the runner (advanced-scaling). `bot.stop()` syncs the update offset so processed updates aren't redelivered.

## Webhooks

```ts
import express from "express";
import { webhookCallback } from "grammy";

const app = express();
app.use(express.json());
app.post(`/${secretPath}`, webhookCallback(bot, "express", {
  secretToken: process.env.WEBHOOK_SECRET, // checks X-Telegram-Bot-Api-Secret-Token
  onTimeout: "throw",                       // default; or "return" / custom fn
  timeoutMilliseconds: 10_000,              // default
}));
app.listen(3000);

// once, e.g. in a deploy script:
await bot.api.setWebhook(`https://example.com/${secretPath}`, {
  secret_token: process.env.WEBHOOK_SECRET,
  allowed_updates: ["message", "callback_query"],
});
```

Never call `bot.start()` together with webhooks.

### Adapters (`webhookCallback(bot, adapter)`)

`express` (default; also Google Cloud Functions), `fastify`, `koa`, `hono`, `http`/`https` (Node, Vercel serverless), `std/http` (`Deno.serve`, Fresh, Vercel Edge, Supabase), `oak`, `bun`, `cloudflare`, `cloudflare-mod`, `aws-lambda`, `aws-lambda-async`, `azure`, `next-js`, `sveltekit`, `nhttp`, `worktop`, `serveHttp`.

```ts
// Deno / Deno Deploy / Supabase
Deno.serve(webhookCallback(bot, "std/http"));

// Cloudflare Workers (module syntax) — create bot per request, cache botInfo
let botInfo: UserFromGetMe | undefined;
export default {
  async fetch(req: Request, env: Env) {
    const bot = new Bot(env.BOT_TOKEN, { botInfo });
    if (!botInfo) { await bot.init(); botInfo = bot.botInfo; }
    registerHandlers(bot);
    return webhookCallback(bot, "cloudflare-mod")(req);
  },
};
```

Better: put `BOT_INFO` (JSON of `getMe`) in an env var and pass `botInfo: JSON.parse(env.BOT_INFO)`.

### Webhook timeouts — the critical pitfall

- Telegram delivers updates **per chat sequentially** and **across chats concurrently**; it waits for your HTTP response before sending the next update of that chat.
- If you don't respond in time, Telegram **re-sends the update** → duplicated processing, often looping dozens of times.
- `webhookCallback` throws after 10 s (`onTimeout: "throw"`) so you see it in logs. `"return"` ends the request early but lets the next update of the same chat run concurrently → session data loss / races. Don't use it to hide slow handlers.
- **Fix:** keep handlers fast; push long work (file processing, LLM calls, broadcasts, auto-retry waits) to a queue and message the user when done.

### Webhook reply

`client.canUseWebhookReply: (method) => boolean` lets one API call ride on the webhook response (saves a request) but you lose the result, error handling, and abort; types won't reflect this. Only use for fire-and-forget calls like `sendChatAction`.

## Delivery Guarantees

- Webhooks: Telegram retries until `OK`; dedupe by `update_id` yourself if you need exactly-once.
- `bot.start()`: at-least-once; after a crash up to 100 updates may be reprocessed.
- Runner (concurrent): offsets are confirmed early → neither at-least-once nor at-most-once.

## Testing

- Mock outgoing calls with a transformer: `bot.api.config.use((prev, method, payload) => ({ ok: true, result: fake(method, payload) }) as any)`.
- Feed fake updates via `await bot.handleUpdate(update)` (pass `botInfo` in the constructor so no `getMe` is needed).

## Deployment Checklist

- Errors: `bot.catch` (polling) or framework error handler (webhooks); lint for floating promises.
- Add auto-retry; add a final `bot.on("callback_query:data")` answering all queries.
- Send uploads by path/Buffer, not streams (retries re-read the source).
- Polling at scale: runner + `sequentialize` + graceful shutdown.
- Webhooks: no long-running middleware; set `botInfo` on serverless; `secretToken`.
- Sessions: real `storage` adapter; consider `lazySession`.

<!--
Source references:
- https://grammy.dev/guide/deployment-types
- https://grammy.dev/advanced/reliability
- https://grammy.dev/advanced/deployment
- https://grammy.dev/hosting/cloudflare-workers
- https://grammy.dev/hosting/cloudflare-workers-nodejs
-->
