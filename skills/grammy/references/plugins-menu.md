---
name: grammy-menu
description: Menu plugin - interactive inline-button menus with handlers, dynamic labels/ranges, navigation, payloads, outdated-menu detection
---

# Interactive Menus (`@grammyjs/menu`)

`Menu` is an `InlineKeyboard` whose buttons carry handlers. It stores no state: menus are re-rendered on click to find the pressed button.

## Usage

```ts
import { Menu, MenuRange } from "@grammyjs/menu";

const menu = new Menu<MyContext>("main")
  .text("A", (ctx) => ctx.reply("You pressed A")).row()
  .text(
    (ctx) => (ctx.session.notify ? "🔔 On" : "🔕 Off"),  // dynamic label
    (ctx) => { ctx.session.notify = !ctx.session.notify; ctx.menu.update(); },
  )
  .url("Docs", "https://grammy.dev");

bot.use(menu); // BEFORE other callback query handlers
bot.command("menu", (ctx) => ctx.reply("Settings:", { reply_markup: menu }));
```

### Inside handlers: `ctx.menu`

- `ctx.menu.update()` — re-render (lazy; merged with a following `editMessageText`). Editing the message text also refreshes buttons automatically.
- `ctx.menu.close()` — remove buttons.
- `ctx.menu.nav("id")` / `ctx.menu.back()` — navigate.
- `await ctx.menu.update({ immediate: true })` — do it now.

### Navigation

```ts
const main = new Menu("root").text("Welcome", mw).row().submenu("Credits", "credits");
const credits = new Menu("credits").text("Show", mw).back("Go Back");
main.register(credits);   // credits' parent = main
bot.use(main);            // install ONLY the root of a registered hierarchy
```

Independent menus are each installed with `bot.use`.

### Dynamic ranges (variable buttons)

```ts
const list = new Menu<MyContext>("items").dynamic((ctx, range) => {
  for (const item of ctx.session.items) {
    range.text({ text: item.name, payload: item.id }, (ctx) => ctx.reply(`Item ${ctx.match}`)).row();
  }
}).text("Cancel", (ctx) => ctx.deleteMessage());
```

Range builders (may be async) must be **pure and stable**: no side effects, no randomness/time — they run again on click and must yield the same layout.

### Payloads

`{ text, payload }` stores a short string (< ~50 bytes) on text/navigation buttons, read via `ctx.match`. Use for indexes/ids only.

### Options

```ts
new Menu("id", {
  autoAnswer: false,                     // answer callback queries yourself
  onMenuOutdated: "Updated, try again.", // or handler fn, or false to disable
  fingerprint: (ctx) => ctx.session.version, // replace layout heuristic
});
```

A menu is "outdated" if shape, button position, or pressed label changed since sending; default behavior re-renders it with a notice.

## Key Points

- Never create `new Menu()` or call `.text()` inside handlers (memory leak). Build all menus at startup; use `dynamic` for runtime variation.
- Install menus before handlers that read callback data, and include `callback_query` in custom `allowed_updates`.
- Persist state in sessions/DB, not in variables.
- Inside conversations use `conversation.menu()` (see plugins-conversations).

<!--
Source references:
- https://grammy.dev/plugins/menu
-->
