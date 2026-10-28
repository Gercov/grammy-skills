---
name: grammy-conversations
description: Conversations plugin v2 - multi-step dialogs with wait calls, the replay model and golden rule (conversation.external), types, plugins, persistence, forms, menus, parallel conversations
---

# Conversations (`@grammyjs/conversations` v2)

Write multi-step dialogs as one async function that `await`s user replies. Use for any flow with more than one step (forms, wizards, captchas).

## Usage

```ts
import { Bot, type Context } from "grammy";
import {
  type Conversation, type ConversationFlavor, conversations, createConversation,
} from "@grammyjs/conversations";

type MyContext = ConversationFlavor<Context>;       // OUTSIDE (middleware) context
type MyConversationContext = Context;               // INSIDE (conversation) context
type MyConversation = Conversation<MyContext, MyConversationContext>;

const bot = new Bot<MyContext>(token);
bot.use(conversations());

async function signup(conversation: MyConversation, ctx: MyConversationContext) {
  await ctx.reply("What is your name?");
  const { message } = await conversation.waitFor("message:text", {
    otherwise: (ctx) => ctx.reply("Please send text!"),
  });
  const age = await conversation.form.int({ otherwise: (ctx) => ctx.reply("A number please") });
  await conversation.external(() => db.saveUser(message.text, age)); // side effect!
  await ctx.reply(`Welcome, ${message.text}!`);
}
bot.use(createConversation(signup));                 // id = function name
// bot.use(createConversation(signup, "signup-v2")); // or custom id

bot.command("signup", (ctx) => ctx.conversation.enter("signup"));
bot.command("cancel", (ctx) => ctx.conversation.exit("signup"));
```

Install `conversations()` and `createConversation(...)` **before** handlers that call `enter`.

## The Replay Model and the Golden Rule

The builder function is **re-executed from the top for every new update**. Past wait calls return stored updates, and `ctx.api` calls made by previous runs are not repeated. Therefore:

> **Code that behaves differently between replays must be wrapped in `conversation.external`.**

```ts
const user = await conversation.external(() => db.getUser(id)); // DB, files, network, globals
const rnd  = await conversation.random();    // instead of Math.random()
const now  = await conversation.now();       // instead of Date.now()
await conversation.log("debug");             // logs only on non-replay runs
await conversation.external(() => bot.api.sendMessage(adminId, "x")); // bot.api / other Api instances
```

Do **not** wrap `ctx.reply`, ctx actions, or `ctx.api.*` — the plugin handles them. Return values of `external` are stored, so they must be serializable (or provide `beforeStore`/`afterLoad`).

## Waiting

```ts
const ctx1 = await conversation.wait();
const ctx2 = await conversation.waitFor(":photo", { otherwise: (c) => c.reply("Photo please") });
const ctx3 = await conversation.waitForHears(/yes|no/i);
const ctx4 = await conversation.waitForCommand("done");
const ctx5 = await conversation.waitForCallbackQuery(/^opt:/);
const ctx6 = await conversation.waitFor(":text").andFrom(userId).and((c) => valid(c.msg.text));
await conversation.wait({ maxMilliseconds: 60 * 60 * 1000 }); // halt if the next update arrives later

// forms: extract + validate in one step
const photo = await conversation.form.photo();
const n = await conversation.form.number();
const op = await conversation.form.select(["+", "-"], { otherwise: (c) => c.reply("+ or -") });
```

Non-matching updates are dropped (unless the conversation is `parallel`). Concurrent waits via `Promise.all` work; an un-awaited `conversation.waitForCommand("exit").then(() => conversation.halt())` acts as a global escape hatch.

## Exiting and Control Flow

- Return, throw, or `await conversation.halt()` ends it; `ctx.conversation.exit("id")` / `exitAll()` from middleware.
- `const cp = conversation.checkpoint(); ... await conversation.rewind(cp);` jumps back (doesn't undo sent messages).
- Normal JS works: variables, loops, helper functions, calling other conversation functions (they can return values).
- Avoid infinite conversations — all updates and external results are stored and replayed.

## Context Types and Plugins Inside Conversations

Inside contexts are fresh objects without outside plugins. Outside type must have `ConversationFlavor`; inside type must not.

```ts
type MyConversationContext = HydrateFlavor<Context>;
bot.use(createConversation(convo, { plugins: [hydrate()] }));
// defaults for all conversations:
bot.use(conversations<MyContext, MyConversationContext>({ plugins: [hydrate()] }));

// transformer plugins inside conversations
createConversation(convo, {
  plugins: [async (ctx, next) => { ctx.api.config.use(autoRetry()); await next(); }],
});
```

**Sessions** can't be passed as a plugin. Access them through the outside context:

```ts
const session = await conversation.external((ctx) => ctx.session);
session.count++;
await conversation.external((ctx) => { ctx.session = session; });
```

## Arguments, Options, Persistence

```ts
await ctx.conversation.enter("order", productId, { qty: 2 }); // JSON-serializable, untyped!
async function order(conversation: MyConversation, ctx: MyConversationContext, productId: string, opts: { qty: number }) {}

createConversation(convo, { parallel: true, maxMillisecondsToWait: 3_600_000 });

bot.use(conversations({
  storage: { type: "key", version: 3, adapter: new FileAdapter({ dirName: "convo" }),
    getStorageKey: (ctx) => ctx.chat?.id.toString(), prefix: "convo-" },
  onEnter: (id, ctx) => {}, onExit: (id, ctx) => {},
}));

ctx.conversation.active();        // { captcha: 2, settings: 1 }
ctx.conversation.active("convo"); // 0 | 1 | n
```

Default storage is in-memory. Bump `version` whenever you change conversation code with persisted state, or replays desync.

## Conversational Menus

```ts
const menu = conversation.menu()
  .text("Set email", async (ctx) => {
    await ctx.reply("Email?");
    email = (await conversation.waitFor(":text")).msg.text;
    ctx.menu.update();
  });
await ctx.reply("Menu", { reply_markup: menu });
// ... ctx.menu.close() before leaving
```

Use `conversation.menu("same-id")` with an outside `new Menu("same-id")` of identical shape/labels to hand control in and out.

## Key Points

- One active conversation per chat by default; `enter` throws if one is active. `parallel: true` allows many and passes rejected updates on to other conversations/middleware.
- Storage key defaults to chat id; custom keys are risky on webhooks/serverless (same race issues as sessions).
- v1 → v2: stored data must be dropped; `conversation.session` and `conversation.run` are gone; two context types required; `drop` option became `next`.

<!--
Source references:
- https://grammy.dev/plugins/conversations
-->
