---
name: grammy-flood-limits
description: Handling Telegram rate limits (429) with auto-retry, broadcasting correctly, paid broadcasts, and why not to throttle
---

# Flood Limits and Broadcasting

Any API call can fail with 429 + `retry_after` at any time — regardless of bot age, traffic, or method (`getMe` and `getUpdates` included). Exact limits are unspecified and dynamic. The only correct response: **wait `retry_after`, then retry**. Hitting limits is fine; ignoring them gets bots banned.

## Usage: auto-retry (`@grammyjs/auto-retry`)

```ts
import { autoRetry } from "@grammyjs/auto-retry";

bot.api.config.use(autoRetry()); // install on every bot

autoRetry({
  maxRetryAttempts: 3,              // then fail with GrammyError
  maxDelaySeconds: 10,              // fail immediately if retry_after is larger
  rethrowInternalServerErrors: false, // 5xx retried with exponential backoff (3 s .. 1 h) by default
  rethrowHttpErrors: false,           // network errors retried too by default
});
```

With auto-retry, a request can take a long time. On webhooks this collides with the update timeout → cap `maxDelaySeconds` or move sending to a queue.

## Broadcasting

```ts
bot.api.config.use(autoRetry());
for await (const { chatId, text } of loadRecipientsSlowlyFromDb()) {
  try {
    await bot.api.sendMessage(chatId, text);
  } catch (e) {
    if (e instanceof GrammyError && e.error_code === 403) await markInactive(chatId);
    else throw e;
  }
}
```

- Sequential, no artificial delays, never ignore 429. At most a few (~3) in parallel.
- ~30 msg/s across users for bulk sends; a broadcast to many users takes hours — plan for it.
- Replying to incoming messages is not "bulk"; 1000+/s is fine.
- Per chat: ~1 msg/s (bursts allowed); groups: ~20 msg/min. auto-retry handles both.

### Paid broadcasts (≥ 10,000 Stars balance)

Enable in @BotFather, pass `allow_paid_broadcast: true` on each call, and run sends concurrently via a queue — up to 1000 msg/s. Rule of thumb: concurrency ≈ average send latency in ms (57 ms → 57 concurrent). Still respect 429s; one ignored 429 among concurrent calls extends the cooldown.

## Throttling (discouraged)

`@grammyjs/transformer-throttler` (`bot.api.config.use(apiThrottler())`, Bottleneck-based `global`/`group`/`out` configs) pre-delays calls. Telegram calls knowing/throttling to limits useless and harmful: you still get 429s from undocumented limits, and your bot is slower. Prefer auto-retry; if you use the throttler, add auto-retry too.

## Key Points

- Don't confuse with `@grammyjs/ratelimiter`, which limits *incoming* spam per user (plugins-utilities).
- The Bot API also returns 429 during its own maintenance restarts.
- Retries re-run the request; upload by path/Buffer, not streams.

<!--
Source references:
- https://grammy.dev/advanced/flood
- https://grammy.dev/plugins/auto-retry
- https://grammy.dev/plugins/transformer-throttler
-->
