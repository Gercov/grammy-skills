---
name: grammy-commands-plugin
description: Commands plugin - CommandGroup, syncing setMyCommands, scoped and localized commands, commandNotFound suggestions, custom prefixes, regex commands
---

# Commands Plugin (`@grammyjs/commands`)

Defines commands once and uses that definition for both handling and the client command menu (`setMyCommands`), including scopes and translations.

## Usage

```ts
import {
  Command, CommandGroup, commandNotFound, commands, type CommandsFlavor, LanguageCodes,
} from "@grammyjs/commands";

type MyContext = CommandsFlavor<Context>;
const bot = new Bot<MyContext>(token);
bot.use(commands()); // installs ctx.setMyCommands, ctx.getCommandEntities

const userCommands = new CommandGroup<MyContext>();
userCommands.command("hello", "Say hello", (ctx) => ctx.reply("Hello!"));
userCommands.add([new Command<MyContext>("bye", "Say bye", (ctx) => ctx.reply("Bye"))]);

bot.use(userCommands);
await userCommands.setCommands(bot); // push menu for all scopes/languages
```

### Per-chat command menus

```ts
loggedOut.command("login", "Log in", async (ctx) => {
  await ctx.setMyCommands(loggedIn);           // this chat now shows loggedIn's commands
});
await ctx.setMyCommands(userCommands, devCommands); // merge several groups
```

### Scopes (menu visibility AND handler filtering)

```ts
userCommands
  .command("hello", "Say hello", (ctx) => ctx.reply("default scope"))
  .addToScope({ type: "all_group_chats" }, (ctx) => ctx.reply(`Hi ${ctx.chat.title}`))
  .addToScope({ type: "all_private_chats" }, (ctx) => ctx.reply(`Hi ${ctx.chat.first_name}`))
  .addToScope({ type: "all_chat_administrators" }, (ctx) => ctx.reply("Hi admin"));
```

The most specific matching scope handler runs; no manual `filter` needed.

### Localization

```ts
userCommands.command("chef", "Steak delivery", mw)
  .localize(LanguageCodes.Spanish, "cocinero", "Bife a domicilio")
  .localize("pt", "chef", "Entrega de bife");

// from i18n .ftl files
userCommands.commands.forEach((cmd) =>
  i18n.locales.forEach((l) =>
    cmd.localize(l, i18n.t(l, `${cmd.name}.command`), i18n.t(l, `${cmd.name}.description`))));
```

### "Did you mean …?"

```ts
bot
  .filter(commandNotFound([userCommands, adminCommands], { ignoreCase: true, similarityThreshold: 0.6 }))
  .use(async (ctx) => {
    if (ctx.commandSuggestion) return ctx.reply(`Did you mean ${ctx.commandSuggestion}?`);
    await ctx.reply("Unknown command");
  });
```

Register after all command groups. Only triggers for text that looks like commands of the given groups' prefixes.

### Per-command / group options

```ts
new CommandGroup({ ignoreCase: true, targetedCommands: "optional" }); // "ignored" | "optional" | "required"
group.command("dad", "calls dad", mw, { prefix: "?" });             // custom prefix (not shown in menu)
group.command("tag", "find tag", mw, { matchOnlyAtStart: false });    // anywhere in text
group.command(/delete_([a-z]+)/, "Delete", mw);                      // regex command
```

## Key Points

- Menu-visible command names: 1–32 chars of `a-z`, `0-9`, `_`. Invalid names still handle but `setCommands` throws for them.
- Command menus only control visibility, not access — enforce access with scopes or filters.
- `Command` instances do nothing until added to a `CommandGroup` that's installed with `bot.use`.

<!--
Source references:
- https://grammy.dev/plugins/commands
-->
