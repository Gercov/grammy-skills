---
name: grammy-inline-queries
description: Handling inline mode - bot.inlineQuery, InlineQueryResultBuilder, answerInlineQuery, results button, chosen_inline_result feedback
---

# Inline Queries (built-in)

Users type `@your_bot query` in any chat; the bot answers with a result list. Enable inline mode in @BotFather first.

## Usage

```ts
import { InlineKeyboard, InlineQueryResultBuilder } from "grammy";

bot.inlineQuery(/best bot (framework|library)/, async (ctx) => {
  const match = ctx.match;            // RegExpMatchArray
  const query = ctx.inlineQuery.query;

  const result = InlineQueryResultBuilder
    .article("id:grammy", "grammY", {
      reply_markup: new InlineKeyboard().url("Website", "https://grammy.dev"),
    })
    .text("<b>grammY</b> is great", { parse_mode: "HTML" }); // message content

  await ctx.answerInlineQuery([result], { cache_time: 30 * 24 * 3600 });
});

// Fallback: always answer
bot.on("inline_query", (ctx) => ctx.answerInlineQuery([]));
```

### Result builder

```ts
InlineQueryResultBuilder.photo("id-0", photoUrl);                   // sends the photo
InlineQueryResultBuilder.photo("id-1", photoUrl).text("Sends text instead");
InlineQueryResultBuilder.article("id-2", "Title").text("Message text");
InlineQueryResultBuilder.audioCached("id-3", audioFileId);         // *Cached = by file_id
```

Each result needs a unique id (≤ 64 bytes), a display object, and message content (implicit for media). Use `next_offset` in `answerInlineQuery` options for pagination.

### Button above results (e.g. login first)

```ts
await ctx.answerInlineQuery(results, {
  button: { text: "Open private chat", start_parameter: "login" },
});

bot.command("start").filter((ctx) => ctx.match === "login", async (ctx) => {
  await ctx.reply("DM open, you can go back now!", {
    reply_markup: new InlineKeyboard().switchInline("Go back"),
  });
});
```

### Chosen result feedback

Enable inline feedback in @BotFather to receive `chosen_inline_result`:

```ts
bot.chosenInlineResult(/id-\d+/, (ctx) => ctx.chosenInlineResult.query);
bot.on("chosen_inline_result", mw);
```

Messages sent via inline mode are "inline messages": edit them with `ctx.editMessageText` from callback queries (uses `ctx.inlineMessageId`).

## Key Points

- Inline queries have no `ctx.chat`; default session keys resolve to `undefined` → no session.
- Avoid mixing text and media results in one list; clients render it poorly.
- The bot doesn't need to be a member of the chat where the result is sent.

<!--
Source references:
- https://grammy.dev/plugins/inline-query
-->
