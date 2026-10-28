---
name: grammy-utility-plugins
description: Smaller plugins - hydrate, chat-members, stream (LLM output), router, ratelimiter, emoji, autoquote, stateless-question
---

# Utility Plugins

Install pattern: middleware plugins via `bot.use(...)`, transformer plugins via `bot.api.config.use(...)`. Add each plugin's flavor to your context type.

## Hydrate (`@grammyjs/hydrate`) — methods on API results

```ts
import { hydrate, type HydrateFlavor } from "@grammyjs/hydrate";
type MyContext = HydrateFlavor<Context>; // transformative
bot.use(hydrate());

bot.on(":photo", async (ctx) => {
  const status = await ctx.reply("Processing…");
  await work();
  await status.editText("Done!");
  setTimeout(() => status.delete().catch(() => {}), 3000);
});
```

Only `ctx.api` results are hydrated. For `bot.api` too: `bot.use(hydrateContext()); bot.api.config.use(hydrateApi());` with `Bot<HydrateFlavor<Context>, HydrateApiFlavor<Api>>`.

## Chat Members (`@grammyjs/chat-members`) — membership transitions

```ts
import { chatMemberFilter, chatMemberIs, myChatMemberFilter } from "@grammyjs/chat-members";

const groups = bot.chatType(["group", "supergroup"]);
groups.filter(chatMemberFilter("out", "in"), (ctx) =>
  ctx.reply(`Welcome ${ctx.chatMember.new_chat_member.user.first_name}!`));
groups.filter(myChatMemberFilter("out", "admin"), (ctx) => ctx.reply("Thanks for making me admin"));
groups.filter(chatMemberFilter(["restricted", "kicked"], ["free", "left"]), mw);

const m = await ctx.getChatMember(ctx.from.id);
if (chatMemberIs(m, "admin")) { /* m.status: "creator" | "administrator" */ }

bot.start({ allowed_updates: [...API_CONSTANTS.DEFAULT_UPDATE_TYPES, "chat_member"] });
```

Groups: `in` (admin/creator/member/restricted_in), `out` (left/kicked/restricted_out), `free`, `admin`, `regular`, `restricted_in`, `restricted_out`, plus raw statuses. `chat_member` needs the bot as admin.
Caching: `bot.use(chatMembers(storageAdapter))` with `ChatMembersFlavor` → `await ctx.chatMembers.getChatMember(chatId?, userId?)`. `hydrateChatMember()` transformer adds `member.is("admin")`.

## Stream (`@grammyjs/stream`) — stream text/LLM output into a message

```ts
import { stream, type StreamFlavor } from "@grammyjs/stream";
type MyContext = StreamFlavor<Context>;
bot.api.config.use(autoRetry()); // strongly recommended: many rapid API calls
bot.use(stream());

bot.chatType("private").command("ask", async (ctx) => {
  const { textStream } = streamText({ model, prompt: ctx.match }); // any (async) iterable of strings
  await ctx.replyWithMarkdownStream(textStream); // also replyWithHtmlStream, replyWithStream (plain)
});
```

Private chats only. Markdown/HTML variants use Telegram rich messages and are recommended.

## Router (`@grammyjs/router`)

```ts
const router = new Router<MyContext>((ctx) => ctx.session.step);
router.route("day").on("message:text", mw);
router.route("month", mw);
router.otherwise(mw);
bot.use(router);
```

Like `bot.route` with chainable routes. For step-by-step forms prefer conversations.

## Ratelimiter (`@grammyjs/ratelimiter`) — throttle spamming users

```ts
import { limit } from "@grammyjs/ratelimiter";
bot.use(limit({
  timeFrame: 2000, limit: 3,                 // default: 1 per 1000 ms
  storageClient: redis,                      // default in-memory; any client with incr + pexpire
  onLimitExceeded: (ctx) => ctx.reply("Slow down!"), // called once per timeFrame
  keyGenerator: (ctx) => ctx.from?.id.toString(),    // or chat id for groups
}));
```

Limits incoming updates per user, not outgoing API calls.

## Others

- **Emoji** (`@grammyjs/emoji`): `bot.use(emojiParser())` + `EmojiFlavor` → ``ctx.emoji`Hi ${"smiling_face_with_sunglasses"}` ``, ``ctx.replyWithEmoji`…` ``; `Reactions.thumbs_up` for `ctx.react`.
- **Autoquote** (`@roziscoding/grammy-autoquote`): `bot.use(autoQuote())` makes every send a reply to the triggering message; per handler: `ctx.api.config.use(addReplyParam(ctx))`.
- **Stateless question** (`@grammyjs/stateless-question`): `new StatelessQuestion("id", handler)`, `bot.use(q.middleware())`, `q.replyWithMarkdown(ctx, text)` — answers recognized via an invisible suffix + force reply; works in privacy mode without storage.

<!--
Source references:
- https://grammy.dev/plugins/hydrate
- https://grammy.dev/plugins/chat-members
- https://grammy.dev/plugins/stream
- https://grammy.dev/plugins/router
- https://grammy.dev/plugins/ratelimiter
- https://grammy.dev/plugins/emoji
- https://grammy.dev/plugins/autoquote
- https://grammy.dev/plugins/stateless-question
-->
