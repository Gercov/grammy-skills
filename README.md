# grammY Skills

[Agent Skills](https://agentskills.io/home) for building Telegram bots with [grammY](https://grammy.dev), generated from the official grammY documentation and kept in sync with it.

> [!NOTE]
> Based on [antfu/skills](https://github.com/antfu/skills), which generates agent skills from source documentation via git submodules.

## Installation

Install with the [skills](https://github.com/vercel-labs/skills) CLI:

```bash
pnpx skills add Gercov/grammy-skills --skill='*'
```

or globally:

```bash
pnpx skills add Gercov/grammy-skills --skill='*' -g
```

## Skills

| Skill | Description | Source |
|-------|-------------|--------|
| [grammy](skills/grammy) | grammY Telegram bot framework - middleware, filter queries, context, keyboards, files, sessions, conversations, menus, i18n, webhooks, runner, flood limits | [grammyjs/website](https://github.com/grammyjs/website) |

## Updating the Skills

1. Install dependencies: `pnpm install`
2. Run `pnpm start init` to clone the submodules
3. Run `pnpm start sync` to pull the latest source docs
4. Ask your agent to `Update skills for grammy`

The source commit used for the current skill is recorded in [GENERATION.md](skills/grammy/GENERATION.md). See [AGENTS.md](AGENTS.md) for detailed generation guidelines.

## License

Skills and the scripts in this repository are [MIT](LICENSE.md) licensed.
