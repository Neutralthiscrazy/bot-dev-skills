# bot-dev-skills

Companion skill pack for **building bots** — Telegram, Discord,
LLM-streaming replies, serverless deploy, and async-runtime patterns.
Five sub-skill orchestrators, one `npx skills add` ships all of them.

## Status

This pack ships **all 5 sub-orchestrators** as a unit. Each is authored
fresh from first-party docs and battle-tested patterns; the SKILL.md is
yours under MIT, the cited upstream sources retain their own licenses.

| #  | sub-skill              | status |
|---:|------------------------|--------|
| 1  | telegram-bot-dev       | shipped |
| 2  | discord-bot-dev        | shipped |
| 3  | llm-streaming-reply    | shipped |
| 4  | serverless-bot-deploy  | shipped |
| 5  | bot-async-runtime      | shipped |

## Install

```bash
npx -y skills add Neutralthiscrazy/bot-dev-skills -a claude-code -g
```

Without `-g`, install lands at `<cwd>/.claude/skills/` — usually wrong.
**Always pass `-g`.** Replace `claude-code` with `codex` / `opencode` /
other agents as needed.

## What this pack is (and isn't)

### Is

- Opinionated digests written from first-party docs and battle-tested
  patterns. We cite upstream docs (aiogram.dev, grammy.dev, telegram
  Bot API, discord.js guide, discord.com/developers/docs,
  platform.openai.com, docs.anthropic.com, Cloudflare Workers docs,
  Python asyncio stdlib, Node.js event-loop docs, etc.) inline; you
  verify there.
- Each sub-orchestrator has its own routing table and `references/*.md`
  files loaded on demand.
- No vendor copies of upstream SKILL.md from other repositories.

### Isn't

- Not a wholesale-copy of upstream documentation.
- Not vendored from existing skills.sh packages — each sub-skill is
  authored fresh.

## Provenance

All content authored by [neutralthiscrazy](https://github.com/Neutralthiscrazy)
under MIT for the digest text. Upstream sources retain their own licenses
and copyrights; see `Sources cited` sections inside each sub-skill's
SKILL.md for the full list.

Re-sync discipline does not apply (we are not vendoring — there is
nothing to re-download).

## License

MIT.
