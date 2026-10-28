---
name: grammy-handlers
description: Composer handler methods - command, hears, callbackQuery, chatType, reaction, inlineQuery; ctx.match, deep linking, setMyCommands
---

# Handler Methods

Shorthands on `Bot`/`Composer` beyond `bot.on`. All accept multiple middleware and return a Composer for chaining.

## Usage

```ts
// Commands: only at the start of the message; also match /start@your_bot
bot.command("start", (ctx) => ctx.reply("Hi"));
bot.command(["a", "b"], mw);
bot.command("start@your_bot", mw); // only targeted form

bot.command("add", (ctx) => {
  const args = ctx.match; // "apple pie" for "/add apple pie"
});

// Deep links: t.me/your_bot?start=ref-42 -> "/start ref-42"
bot.command("start", (ctx) => {
  if (ctx.match === "ref-42") { /* referral */ }
});

// Text matching (string or RegExp); ctx.match is the RegExpMatchArray for regex
bot.hears(/echo (.+)/, (ctx) => ctx.reply(ctx.match[1]));

// Callback queries from inline buttons
bot.callbackQuery("click", (ctx) => ctx.answerCallbackQuery("Clicked"));
bot.callbackQuery(/^item:(\d+)$/, (ctx) => { const id = ctx.match[1]; });

// Chat type scoping
bot.chatType("private").command("settings", mw);
bot.chatType(["group", "supergroup"]).on("message", mw);

// Others
bot.reaction("🎉", mw);                        // see features-reactions
bot.inlineQuery(/search (.+)/, mw);            // see features-inline-queries
bot.chosenInlineResult(/id-\d+/, mw);
bot.gameQuery("my_game", mw);
```

### Command menu

```ts
await bot.api.setMyCommands([
  { command: "start", description: "Start the bot" },
  { command: "help", description: "Show help" },
]);
```

For scoped/localized command menus kept in sync with handlers, use the commands plugin (plugins-commands).

### Has checks (imperative narrowing)

Same matching logic as the methods above, usable anywhere (especially in conversations):

```ts
if (ctx.hasCommand("start")) { /* ... */ }
if (ctx.hasCallbackQuery(/^item:\d+$/)) ctx.callbackQuery.data; // string
if (ctx.hasChatType("private")) { /* ... */ }
if (ctx.has(":text")) ctx.msg.text;

const isText = Context.has.filterQuery(":text"); // reusable predicate
```

## Key Points

- `ctx.match` type depends on the method: string for `command`, `RegExpMatchArray | string` for `hears`/`callbackQuery`/`inlineQuery`.
- Commands not at message start (e.g. "please /start") are not matched by core; the commands plugin can (`matchOnlyAtStart: false`).
- Always install a catch-all `bot.on("callback_query:data", (ctx) => ctx.answerCallbackQuery())` last, or clients spin for up to a minute on unhandled buttons.
- Business bots get `/start bizChat<chat_id>` deep links from Telegram (see features-business).

<!--
Source references:
- https://grammy.dev/guide/basics
- https://grammy.dev/guide/commands
- https://grammy.dev/guide/context#probing-via-has-checks
- https://grammy.dev/plugins/keyboard#responding-to-inline-keyboard-clicks
-->
