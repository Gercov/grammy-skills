---
name: grammy-filter-queries
description: bot.on() filter query language - L1/L2/L3 parts, omitted parts, shortcuts, AND/OR, type narrowing, matchFilter
---

# Filter Queries (`bot.on`)

`bot.on(query, ...mw)` filters updates with a typed query string (`"update_type:field:subfield"`) and narrows `ctx` so matched fields are non-optional. Every valid query autocompletes; invalid ones fail at compile time *and* at startup.

## Usage

```ts
bot.on("message");                  // any message
bot.on("message:text", (ctx) => {
  ctx.msg.text; // string (not string | undefined)
});
bot.on("message:photo");
bot.on("message:entities:url");     // L3: entity type
bot.on("edited_message:entities");
bot.on("callback_query:data");
bot.on("message:new_chat_members:is_bot"); // :is_bot sugar
bot.on("message:left_chat_member:me");     // :me = this bot
```

### Omitting parts

```ts
bot.on(":text");        // message OR channel_post with text
bot.on("message::url"); // url in text OR caption entities
bot.on("::email");      // both omitted
```

Use `ctx.msg` to read whichever message type matched.

### Shortcuts

| Query | Expands to |
|---|---|
| `msg` | `message` + `channel_post` |
| `edit` | `edited_message` + `edited_channel_post` (`edit:location` = live location update) |
| `:media` | `:photo` + `:video` |
| `:file` | photo, animation, audio, document, video, video_note, voice, sticker (so `ctx.getFile()` is safe) |

### Combining

```ts
bot.on(["message", "edited_message"], mw);       // OR: array
bot.on("::url").on(":forward_origin", mw);       // AND: chain
bot.on(["channel_post", ":forward_origin"])      // CNF
  .on(":text")
  .on(["::url", "::hashtag"], mw);
bot.on(":forward_origin").command("help", mw);   // mix with other Composer methods
```

### Sender type

```ts
bot.on("channel_post");                                          // channel author = ctx.senderChat
bot.on("message:is_automatic_forward");                          // linked channel auto-forward
bot.on("message").filter((ctx) => ctx.senderChat === undefined); // normal user
bot.on("message").filter((ctx) => ctx.senderChat?.id === ctx.chat.id); // anonymous admin
```

Filter queries never call the API. For user properties (admin status, etc.), add a `.filter(async (ctx) => (await ctx.getAuthor()).status === "administrator", mw)`.

### Reusing query logic and types

```ts
import { type Filter, matchFilter } from "grammy";

bot.drop(matchFilter(":text"), mw); // predicate compiled from query
function handle(ctx: Filter<MyContext, ":text">) { /* narrowed ctx */ }
```

## Key Points

- `my_chat_member`: bot blocked/unblocked (private) or added/removed (groups). Always delivered.
- `chat_member`: other users' status changes — only if the bot is admin **and** `"chat_member"` is in `allowed_updates`. Don't rely on `new_chat_members`/`left_chat_member` service messages to track joins; large groups omit them.
- Same for `message_reaction`, `message_reaction_count` (opt-in via `allowed_updates`).
- If the inferred type says a field may be `undefined`, there really are cases where it is — don't `!` it away.
- Matching is (amortized) O(1) per update regardless of query complexity.

<!--
Source references:
- https://grammy.dev/guide/filter-queries
-->
