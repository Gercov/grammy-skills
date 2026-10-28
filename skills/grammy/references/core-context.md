---
name: grammy-context
description: The ctx object - shortcuts, actions, custom properties, custom context types and context flavors (additive vs transformative)
---

# Context and Context Flavors

One `Context` per update, shared across all middleware. It exposes update data, shortcuts, `ctx.api`, and action methods with chat/message ids pre-filled.

## Usage

### Reading data

```ts
ctx.update;          // raw Update
ctx.me;              // bot info (UserFromGetMe)
ctx.message; ctx.editedMessage; ctx.callbackQuery; ctx.myChatMember; // per update type

// Shortcuts (work across update types)
ctx.msg;             // message | edited | channel post | business message
ctx.msgId;           // message id (also for reactions)
ctx.chat; ctx.chatId;
ctx.from;            // author / clicker / querier
ctx.senderChat;      // anonymous admin / channel sender
ctx.inlineMessageId; ctx.businessConnectionId;
ctx.entities();               // [{ ...entity, text }]
ctx.entities(["email", "url"]);
ctx.reactions();              // see features-reactions
```

### Actions

```ts
await ctx.reply("Hi");                                  // sendMessage to ctx.chat
await ctx.reply("Quoted", { reply_parameters: { message_id: ctx.msg.message_id } });
await ctx.replyWithPhoto(fileIdOrUrlOrInputFile);
await ctx.editMessageText("New text");                  // message or inline message of the callback
await ctx.deleteMessage();
await ctx.answerCallbackQuery({ text: "Done" });
await ctx.react("👍");
await ctx.getAuthor();                                  // getChatMember for ctx.from
await ctx.banAuthor();
await ctx.api.sendMessage(otherChatId, "To someone else"); // different target
```

`ctx.reply` is **not** a Telegram reply; pass `reply_parameters` for that. Every action takes a trailing options object (`Other`) with the Bot API's optional params.

### Custom properties (recommended: via middleware + type)

```ts
import { Bot, type Context } from "grammy";

interface Config { isDeveloper: boolean }
type MyContext = Context & { config: Config };

const bot = new Bot<MyContext>(token);
bot.use(async (ctx, next) => {
  ctx.config = { isDeveloper: ctx.from?.id === DEV_ID };
  await next();
});
bot.command("start", (ctx) => ctx.reply(ctx.config.isDeveloper ? "Hi dev" : "Hi"));
```

Subclassing `Context` + `new Bot(token, { ContextConstructor: MyContext })` also works but combines badly with plugin flavors — prefer middleware.

### Context flavors

Plugins ship types that describe what they add to `ctx`.

```ts
// Additive: intersect with &, order irrelevant
type A = Context & SessionFlavor<SessionData> & I18nFlavor;

// Transformative: wrap; order can matter (innermost applied first)
type B = HydrateFlavor<FileFlavor<Context>>;

// Mixed: additive inside, transformative outside
type MyContext = HydrateFlavor<FileFlavor<Context & SessionFlavor<S> & I18nFlavor>>;

const bot = new Bot<MyContext>(token);
const composer = new Composer<MyContext>();
const menu = new Menu<MyContext>("id");
```

Each plugin's docs state whether its flavor is additive or transformative. Pass the final context type everywhere (`Bot`, `Composer`, `Menu`, `Router`, `CommandGroup`, extracted handlers).

## Key Points

- Prefer `ctx.api` / ctx actions over `bot.api` inside handlers — context-level transformers (e.g. plugins) only apply to `ctx.api`.
- Context types must be kept in sync with properties you actually set — TS can't verify middleware ran.
- Many type errors with plugins come from combining flavors wrongly; follow the additive-inside/transformative-outside pattern.
- Use autocomplete or https://grammy.dev/ref/core/context for the full list of actions and has checks.

<!--
Source references:
- https://grammy.dev/guide/context
-->
