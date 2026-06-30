# bot-dev-skills

Companion skill pack for **building bots** — Telegram, Discord, LLM-streaming
replies, serverless deploy, and async-runtime patterns. Five sub-skill
orchestrators, one `npx skills add` ships all of them.

## Status

This pack ships **progressively** as each sub-orchestrator is authored and
reviewed. The current commit ships sub #1 only:

| #  | sub-skill              | status   |
|---:|------------------------|----------|
| 1  | telegram-bot-dev       | shipped  |
| 2  | discord-bot-dev        | upcoming |
| 3  | llm-streaming-reply    | upcoming |
| 4  | serverless-bot-deploy  | upcoming |
| 5  | bot-async-runtime      | upcoming |

## Install

```bash
npx -y skills add Neutralthiscrazy/bot-dev-skills -a claude-code -g
```

Or for other agents (replace `<agent>` with `codex`, `opencode`, etc.):

```bash
npx -y skills add Neutralthiscrazy/bot-dev-skills -a <agent> -g
```

Without `-g`, install lands at `<cwd>/.claude/skills/` — usually wrong.
**Always pass `-g`.**

## What this pack is (and isn't)

### Is

- Opinionated digests written from first-party docs and battle-tested
  patterns. We cite upstream docs (aiogram.dev, grammy.dev, telegram Bot API,
  discord.js guide, cloudflare workers docs, etc.) inline; you verify there.
- Five separately-installable sub-orchestrators, each with its own routing
  table and `references/*.md` files loaded on demand.
- No vendor copies of upstream SKILL.md from other repositories.

### Isn't

- Not a wholesale-copy of any upstream documentation (specifically **not**
  a clone of aiogram/grammy/discord.js docs).
- Not vendored from existing skills.sh packages — each sub-skill is authored.

## Provenance

All content authored by [neutralthiscrazy](https://github.com/Neutralthiscrazy)
under MIT for the digest text. Upstream sources retain their own licenses
and copyrights; see `Sources cited` sections inside each sub-skill's
SKILL.md for the full list.

Re-sync discipline does not apply (we are not vendoring — there is nothing
to re-download).

## License

MIT.
