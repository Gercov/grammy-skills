---
name: grammy-keyboards
description: InlineKeyboard and Keyboard (custom/reply keyboard) builders, handling clicks with callback queries, removing keyboards
---

# Inline and Custom Keyboards (built-in)

- **Inline keyboard** (`InlineKeyboard`): buttons under a message; text buttons send an invisible `callback_query` with data.
- **Custom keyboard** (`Keyboard`): replaces the user's system keyboard; a button press just sends its label as a normal text message.

A message has one or the other, and the kind can't be switched by editing.

## Inline Keyboards

```ts
import { InlineKeyboard } from "grammy";

const kb = new InlineKeyboard()
  .text("« Prev", "page:prev").text("Next »", "page:next").row()
  .url("Docs", "https://grammy.dev")
  .switchInline("Share", "query"); // also switchInlineCurrent, switchInlineChosen, webApp, login, pay, copyText ...

await ctx.reply("Pick one", { reply_markup: kb });

// Functional style
const rows = items.map((it) => [InlineKeyboard.text(it.name, `item:${it.id}`)]);
await ctx.reply("Items", { reply_markup: InlineKeyboard.from(rows) });

// Handle clicks
bot.callbackQuery(/^page:(prev|next)$/, async (ctx) => {
  await ctx.answerCallbackQuery();                         // stop the loading spinner
  await ctx.editMessageText("Page …", { reply_markup: kb }); // or editMessageReplyMarkup
});

// LAST: answer anything unhandled
bot.on("callback_query:data", async (ctx) => {
  console.log("Unknown button", ctx.callbackQuery.data);
  await ctx.answerCallbackQuery();
});
```

Remove inline buttons: `ctx.editMessageReplyMarkup({ reply_markup: new InlineKeyboard() })`.

## Custom Keyboards

```ts
import { Keyboard } from "grammy";

const kb = new Keyboard()
  .text("Yes").text("No").row()
  .requestContact("Share phone").requestLocation("Share location")
  .resized()      // fit to buttons
  .oneTime()      // hide after a press
  .persistent()   // always show instead of system keyboard when hidden
  .placeholder("Decide now!")
  .selected();    // only for mentioned users / replied-to sender

await ctx.reply("Are you sure?", { reply_markup: kb });

bot.hears("Yes", (ctx) => ctx.reply("Great"));

// Functional style
const kb2 = Keyboard.from(labels.map((l) => [Keyboard.text(l)])).resized();

// Remove (only possible by sending a new message)
await ctx.reply("Done", { reply_markup: { remove_keyboard: true } });
```

## Key Points

- `callback_data` is limited to 64 bytes — store ids, not data; keep state in sessions/DB.
- If `.text(label)` has no data, the label is used as data.
- Custom keyboard buttons can't send text different from their label; you can't tell a button press from typed text. Use inline buttons if you need hidden payloads.
- `reply_markup: { force_reply: true }` makes the client reply to your message — useful for bots in privacy mode in groups.
- For multi-page, stateful button UIs, use the menu plugin (plugins-menu) instead of hand-rolled callback routing.

<!--
Source references:
- https://grammy.dev/plugins/keyboard
- https://grammy.dev/guide/basics#force-reply
-->
