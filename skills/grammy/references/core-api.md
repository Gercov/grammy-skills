---
name: grammy-api
description: Calling the Bot API with bot.api/ctx.api/Api, the Other options object, raw calls, Bot API types, bot config, local Bot API server, proxies
---

# Calling the Bot API

Every Bot API method exists on `Api` with required params as positional args and optional params in a trailing `other` object. grammY serializes nested JSON fields and `InputFile` uploads for you.

## Usage

```ts
import { Api, Bot } from "grammy";

const bot = new Bot(token);
const msg = await bot.api.sendMessage(chatId, "<b>Hi</b>", {
  parse_mode: "HTML",
  reply_parameters: { message_id: 42 },
  link_preview_options: { is_disabled: true },
});
msg.message_id;

const api = new Api(token);            // standalone, no Bot (scripts, workers)
await api.sendMessage(chatId, "Yo");

// Raw: original single-object signatures, still with grammY serialization
await bot.api.raw.sendMessage({ chat_id: chatId, text: "Hi" });
```

A "missing" parameter is always an optional one — pass it in `other`.

### Types

```ts
import type { Chat, Message, Update, UserFromGetMe } from "grammy/types";
```

On Node with TS, `grammy/types` needs `"moduleResolution": "node16" | "nodenext" | "bundler"`. Never import from `grammy/out/...` (internal). On Deno: `npm:grammy/types`.

### Bot configuration

```ts
const bot = new Bot<MyContext, MyApi>(token, {
  botInfo,                                  // skip getMe on init (serverless!)
  client: {
    apiRoot: "http://localhost:8081",        // local Bot API server
    canUseWebhookReply: (m) => m === "sendChatAction",
    timeoutSeconds: 500,
    baseFetchConfig: { agent: socksAgent, compress: true }, // Node proxy
  },
});
await bot.init(); // fetches botInfo if not given; bot.start()/webhookCallback do this implicitly
bot.botInfo;      // available after init
```

Deno proxy: `client: { baseFetchConfig: { client: Deno.createHttpClient({ proxy: { url } }) } }`.

### Local Bot API server

Run `tdlib/telegram-bot-api` on a VPS to upload up to 2000 MB, download any size, and cut latency (host bot + server near your bot's DC; `api.telegram.org` is in Amsterdam).

1. Call `https://api.telegram.org/bot<token>/logOut` once.
2. Set `client.apiRoot` to the local server.
3. `getFile` now returns an absolute local `file_path` (the file is already on disk).

## Key Points

- `bot.api` is just a pre-built `Api`; inside handlers prefer `ctx.api` or ctx actions.
- `bot.api` types can't be extended by context flavors — plugins that change API types use API flavors (`Bot<C, A>`), see advanced-transformers.
- Use `API_CONSTANTS.ALL_UPDATE_TYPES` / `DEFAULT_UPDATE_TYPES` when building `allowed_updates`.
- Set `botInfo` explicitly on serverless/autoscaling platforms to avoid a `getMe` per cold start.

<!--
Source references:
- https://grammy.dev/guide/api
- https://grammy.dev/guide/basics
- https://grammy.dev/advanced/proxy
- https://grammy.dev/resources/faq
-->
