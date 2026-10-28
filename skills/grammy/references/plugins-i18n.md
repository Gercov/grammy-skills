---
name: grammy-i18n
description: i18n plugin with Fluent (.ftl) translations - ctx.t, locale negotiation, session-stored locale, placeables, localized hears
---

# Internationalization (`@grammyjs/i18n`)

Fluent-based translations exposed as `ctx.t(key, vars)`. Works on Node and Deno (supersedes the older `fluent` plugin).

## Usage

```
locales/
  en.ftl   ->  start = Hi { $name }, how can I /help you?
               cart-msg = You have { $items } items.
  de.ftl
```

```ts
import { hears, I18n, type I18nFlavor } from "@grammyjs/i18n";

interface SessionData { __language_code?: string }
type MyContext = Context & SessionFlavor<SessionData> & I18nFlavor;

const i18n = new I18n<MyContext>({
  defaultLocale: "en",                 // set to a locale you actually have
  directory: "locales",                // sync load; on Deno Deploy use await i18n.loadLocalesDir("locales")
  useSession: true,                    // read/write ctx.session.__language_code
  globalTranslationContext: (ctx) => ({ name: ctx.from?.first_name ?? "" }),
  fluentBundleOptions: { useIsolating: false }, // avoid Unicode isolation marks breaking links/entities
});

bot.use(session({ initial: () => ({}) })); // session BEFORE i18n
bot.use(i18n);

bot.command("start", (ctx) => ctx.reply(ctx.t("start")));
bot.command("cart", (ctx) => ctx.reply(ctx.t("cart-msg", { items: 10 })));

bot.command("language", async (ctx) => {
  if (!i18n.locales.includes(ctx.match)) return ctx.reply(ctx.t("language-invalid"));
  await ctx.i18n.setLocale(ctx.match); // persists to session + applies immediately
  await ctx.reply(ctx.t("language-set"));
});

// Match localized keyboard button text
bot.filter(hears("back-to-menu-btn"), (ctx) => ctx.reply(ctx.t("main-menu")));
```

### Loading translations

```ts
await i18n.loadLocalesDir("locales");                       // async (required on Deno Deploy)
i18n.loadLocaleSync("de", { filePath: "locales/de.ftl" });
await i18n.loadLocale("en", { source: "greeting = Hello { $name }!" });
i18n.t("de", "greeting", { name: "Ann" });                 // outside handlers
```

### Locale negotiation

Default order: session `__language_code` (if `useSession`) → `ctx.from.language_code` → `defaultLocale` → `en` → raw `{key}`.

```ts
new I18n<MyContext>({
  localeNegotiator: (ctx) => ctx.session.locale ?? ctx.from?.language_code ?? "en",
});
await ctx.i18n.renegotiateLocale(); // after changing the source mid-update
await ctx.i18n.useLocale("de");     // this update only (no session)
await ctx.i18n.getLocale();
```

## Key Points

- `ctx.from.language_code` is only present if the user has started the bot in private chat.
- Setting `ctx.session.__language_code` directly only takes effect next update unless you call `renegotiateLocale()`.
- Inside conversations, pass `i18n` via the conversation's `plugins` option and add `I18nFlavor` to the inside context type.
- Combine with the commands plugin's `.localize()` for translated command menus.

<!--
Source references:
- https://grammy.dev/plugins/i18n
- https://projectfluent.org/fluent/guide/
-->
