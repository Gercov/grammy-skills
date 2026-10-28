---
name: grammy-formatting
description: Formatting messages with HTML/MarkdownV2 parse_mode or entities, and the parse-mode plugin's fmt/FormattedString
---

# Message Formatting

Three options: `parse_mode: "HTML"`, `parse_mode: "MarkdownV2"`, or raw `entities`. Entities avoid escaping bugs entirely and are the most reliable — the parse-mode plugin makes them ergonomic.

## parse_mode

```ts
await ctx.reply('<b>Hi!</b> <i>Welcome</i> to <a href="https://grammy.dev">grammY</a>.', {
  parse_mode: "HTML",
});
await ctx.reply("*Hi\\!* _Welcome_ to [grammY](https://grammy.dev)\\.", {
  parse_mode: "MarkdownV2", // every reserved char must be escaped
});
```

Interpolating user input into HTML/Markdown requires escaping; otherwise you get `400: can't parse entities`.

## Entities with the parse-mode plugin (`@grammyjs/parse-mode`)

```ts
import { b, fmt, FormattedString, u } from "@grammyjs/parse-mode";

bot.command("demo", async (ctx) => {
  // Tagged template: toggles like ${b}...${b}; user text is safe, no escaping
  const msg = fmt`${b}Bold${b} ${ctx.match} ${u}underlined${u}`;
  await ctx.reply(msg.text, { entities: msg.entities });

  // Captions
  await ctx.replyWithPhoto(url, { caption: msg.caption, caption_entities: msg.caption_entities });

  // Builder style
  const s = FormattedString.b("Status: ").plain("ok\n").i("details");
  await ctx.reply(s.text, { entities: s.entities });
});

// Compose: fmt accepts FormattedString, strings, and Telegram messages
bot.on("msg:text", async (ctx) => {
  const res = fmt`${ctx.msg}`.plain("\n---\n").bold("My reply");
  await ctx.reply(res.text, { entities: res.entities });
});
```

`fmt` accepts `TextWithEntities` (`{ text, entities }`, incl. incoming messages), `CaptionWithEntities`, entity tags, and anything with `toString()`. Results are always `FormattedString`, so `fmt`, static and instance methods mix freely.

## Echoing/copying formatted messages

Don't convert to HTML — forward the original entities:

```ts
await ctx.copyMessage(targetChatId);     // no "forwarded from"
await ctx.forwardMessage(targetChatId);  // with link to original
await ctx.reply(ctx.msg.text, { entities: ctx.msg.entities });
```

Converting entities to HTML (`@qz/telegram-entities-parser`) is only for displaying Telegram text **outside** Telegram (e.g. a website).

## Key Points

- `text` + `entities` and `caption` + `caption_entities` pairs must not be combined with `parse_mode`.
- Entity offsets are in UTF-16 code units — hand-computing them is error-prone; let the plugin do it.
- For streaming LLM output as formatted text, see the stream plugin in plugins-utilities.

<!--
Source references:
- https://grammy.dev/guide/basics#sending-message-with-formatting
- https://grammy.dev/plugins/parse-mode
- https://grammy.dev/plugins/entity-parser
- https://grammy.dev/resources/faq#_400-bad-request-cannot-parse-entities
-->
