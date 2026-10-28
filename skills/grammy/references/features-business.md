---
name: grammy-business
description: Telegram Business bots - business_message updates, telling owner from customer, replying via business connection, managing chats
---

# Telegram Business

A Business account can connect a bot to manage its private chats with customers. The bot receives messages from **both** sides and can reply as the account owner without being a chat member.

## Usage

```ts
bot.on("business_message", async (ctx) => {
  const conn = await ctx.getBusinessConnection();
  if (ctx.from.id === conn.user.id) return; // sent by the owner/employee, not the customer
  if (conn.rights?.can_reply && ctx.msg.text?.endsWith("?")) {
    await ctx.reply("Soon."); // auto-routed via ctx.businessConnectionId
  }
});

// Edits/deletions in managed chats
bot.on("edited_business_message", mw);
bot.on("deleted_business_messages", mw);

// Delete customer messages (needs rights)
await ctx.deleteBusinessMessages([ctx.msg.message_id]);

// Connection lifecycle: connected, disconnected, rights changed
bot.on("business_connection:is_enabled", async (ctx) => {
  await ctx.api.sendMessage(ctx.businessConnection.user_chat_id, "Thanks for connecting me!");
});
bot.on("business_connection:can_reply", mw);

// "Manage bot" button in a managed chat sends /start bizChat<chatId>
bot.command("start", async (ctx) => {
  if (ctx.match.startsWith("bizChat")) {
    const chatId = ctx.match.slice("bizChat".length);
    await ctx.reply(`Let's manage chat #${chatId}!`);
  }
});
```

## Key Points

- Store `business_connection` objects (from `business_connection` updates) in your DB to avoid calling `getBusinessConnection()` on every message.
- `ctx.reply`, `ctx.editMessageText`, etc. use `ctx.businessConnectionId` automatically.
- Bots cannot forward or copy messages out of business chats.
- `user_chat_id` lets you message the connecting user even if they never started the bot.

<!--
Source references:
- https://grammy.dev/advanced/business
-->
