---
name: grammy-reactions
description: Setting message reactions and handling message_reaction / message_reaction_count updates with bot.reaction and ctx.reactions()
---

# Reactions

## Reacting

```ts
bot.on("message", (ctx) => ctx.react("👍"));
await ctx.api.setMessageReaction(chatId, messageId, [{ type: "emoji", emoji: "🎉" }]);
```

Bots set a single reaction; custom emoji only if already used on the message by a premium user or allowed by chat admins. Only emoji from Telegram's allowed list work (autocompleted; `Reactions` from `@grammyjs/emoji` gives named constants).

## Receiving

Requires the bot to be **admin** and the update types opted in:

```ts
bot.start({
  allowed_updates: ["message", "message_reaction", "message_reaction_count"],
  // or: API_CONSTANTS.ALL_UPDATE_TYPES
});
```

```ts
bot.reaction("🎉", (ctx) => ctx.reply("whoop whoop"));   // newly added emoji
bot.reaction(["👍", "👎"], mw);
bot.reaction({ type: "custom_emoji", custom_emoji_id: "id" }, mw);
bot.reaction({ type: "paid" }, (ctx) => ctx.reply("Thanks for the stars!"));

bot.on("message_reaction:new_reaction:emoji", mw);       // filter queries work too

bot.on("message_reaction", async (ctx) => {
  const { emoji, emojiAdded, emojiRemoved, emojiKept,
    customEmoji, customEmojiAdded, customEmojiRemoved, customEmojiKept,
    paid, paidAdded } = ctx.reactions();
  if (emojiRemoved.includes("👍") && !emoji.includes("👌")) await ctx.banAuthor();
});

// Channels and auto-forwarded posts: anonymous counts only
bot.on("message_reaction_count", (ctx) => ctx.messageReactionCount.reactions);
```

## Key Points

- `message_reaction` (private/groups) includes who reacted plus `old_reaction`/`new_reaction` arrays; users can change several at once.
- Updates contain only the message id, not message content — store messages yourself if you need them.

<!--
Source references:
- https://grammy.dev/guide/reactions
- https://grammy.dev/plugins/emoji#useful-data-for-reactions
-->
