---
name: grammy-files
description: Receiving and sending files - file_id vs file_unique_id, getFile, InputFile uploads, files plugin download, media groups, size limits
---

# Files and Media

Bots only receive `file_id`s; contents must be fetched explicitly. Files are sent by `file_id`, URL, or upload via `InputFile`.

## Receiving

```ts
bot.on("message:voice", async (ctx) => {
  const { file_id, file_unique_id, duration } = ctx.msg.voice;
  const file = await ctx.getFile(); // for the current message; path valid >= 1 h
  const url = `https://api.telegram.org/file/bot${bot.token}/${file.file_path}`;
});
bot.on(":file", async (ctx) => { const f = await ctx.getFile(); }); // any file type
await ctx.api.getFile(someFileId); // a different file
```

Photos arrive as an array of sizes: `ctx.msg.photo.at(-1)` is the largest.

### Files plugin (`@grammyjs/files`)

```ts
import { type FileFlavor, hydrateFiles } from "@grammyjs/files";

type MyContext = FileFlavor<Context>; // transformative
const bot = new Bot<MyContext>(token);
bot.api.config.use(hydrateFiles(bot.token));

bot.on([":video", ":animation"], async (ctx) => {
  const file = await ctx.getFile();
  const tmpPath = await file.download();          // temp file
  await file.download("/data/video.mp4");         // explicit path
  const url = file.getUrl();                      // or download yourself
});
```

For typed results from `bot.api.getFile`, add `FileApiFlavor<Api>` as the bot's API type. With a local Bot API server, `getUrl()` returns a local path and `download()` copies it.

## Sending

```ts
import { InputFile } from "grammy";

await ctx.replyWithPhoto(existingFileId);                          // fastest, no upload
await ctx.replyWithPhoto("https://example.com/cat.png");           // Telegram downloads it
await ctx.replyWithPhoto(new InputFile("/tmp/cat.png"), { caption: "Cat" });
await ctx.replyWithDocument(new InputFile(buffer, "report.pdf"));  // Uint8Array/Buffer + filename
await ctx.replyWithDocument(new InputFile(createReadStream(path)));
await ctx.replyWithDocument(new InputFile(new URL("https://...")));// streamed through your server
await ctx.replyWithDocument(new InputFile(function* () { yield chunk; }));

const sent = await ctx.replyWithVideo(new InputFile(path));
saveFileId(sent.video.file_id); // reuse next time
```

Same `replyWith*` / `send*` naming for audio, video, animation, voice, video note, sticker, document.

### Media groups (albums) and editing media

```ts
import { InputMediaBuilder } from "grammy";

const photo = InputMediaBuilder.photo(new InputFile("/tmp/a.jpg"), { caption: "Album" });
const video = InputMediaBuilder.video("https://example.com/b.mp4");
await ctx.replyWithMediaGroup([photo, video]);

await ctx.editMessageMedia(InputMediaBuilder.photo(newFileId));
```

## Key Points

- `file_id`s are bot-specific — never use another bot's ids. Compare files via `file_unique_id` (stable across bots, can't download).
- Reuse `file_id` with the matching method (a photo id won't work with `sendVideo`).
- Limits on the cloud Bot API: download ≤ 20 MB, upload ≤ 50 MB (photo by URL ≤ 5 MB). Local Bot API server: upload ≤ 2000 MB, unlimited download.
- Prefer file paths/Buffers over streams: retries (auto-retry) re-run requests, and a consumed stream yields a broken file.
- Prefer sending by URL over `InputFile(URL)` when Telegram can fetch it itself.
- Albums don't exist for bots as a unit: each item is a separate message with a shared `media_group_id`, and you can't know when the album is complete. Design flows per message (e.g. "send /done when finished").

<!--
Source references:
- https://grammy.dev/guide/files
- https://grammy.dev/plugins/files
- https://grammy.dev/plugins/media-group
- https://grammy.dev/resources/faq#how-can-i-handle-albums
-->
