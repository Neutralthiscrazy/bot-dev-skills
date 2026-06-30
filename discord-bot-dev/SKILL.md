---
name: discord-bot-dev
description: >-
  Use when developing Discord bots in JavaScript/TypeScript or Python.
  Picks framework (discord.js v14 for Node/TS production, discord.py 2.x
  for Python async), explains slash-commands vs prefix, guides interaction
  patterns (buttons, select menus, modals, context menus), flags permission
  math (32-bit flags + OAuth scopes), and points to Discord developer
  docs inline instead of duplicating them. Routing table loads exactly
  one reference per intent.
version: 0.1.0
user-invocable: false
author: neutralthiscrazy
license: MIT
---

# Discord Bot Development

## Overview

Opinionated companion for building Discord bots in 2026. We **do not**
wholesale-copy Discord developer docs — every reference file is a digest
that links you to the canonical sources (`discord.com/developers/docs`,
`discordjs.guide`, documentation for `discord.py`) for deeper reading.
Two framework choices ship as separate reference files; pick the one
matching your stack, not both.

Authored from first-party docs and battle-tested patterns. License is MIT
on our digest text; cited upstream sources retain their own licenses (MIT
for both discord.js and discord.py — check each reference for specifics
before redistributing).

## When to Use

- Choosing between Discord bot frameworks (discord.js v14 vs discord.py 2.x)
- Designing slash-commands and interactions (buttons, modals, selects)
- Permission and OAuth scope math (creating a working bot invite URL)
- Setting up a guild-versus-global command deployment strategy
- Debugging "interaction failed" errors or missing permissions
- Hosting the bot on VPS, Docker, or serverless (Cloudflare Workers limited)

## When NOT to Use

- Plain webhook integration to a Discord channel (curl a webhook URL — no bot needed)
- User account automation (against Discord ToS — this skill is for bots only)
- Voice bot / music bot deep dives (basic mention only — voice has its own rabbit hole)
- Forum / scheduled events / automod (we touch scheduled events briefly)

## Routing Table

| User intent                                  | Load                                       | Notes |
|----------------------------------------------|--------------------------------------------|-------|
| "Build a Node/TypeScript bot, modern"        | `references/discordjs-v14.md`              | Most popular; full slash + interactions |
| "Build a Python async bot, with cogs"        | `references/discord-py.md`                 | discord.py 2.0+ with discord.app_commands |
| "Design buttons, menus, modals, routing"     | `references/interactions-and-modals.md`    | Pair with framework reference |
| "Math permissions, scopes, invite URL"       | `references/discord-permissions-and-scopes.md` | OAuth bot invite flow |
| "Where do I host it (VPS/serverless/Docker)" | `references/hosting-discord-bots.md`       | Includes CF Workers caveats |

If the bot integrates an LLM — pair `discordjs-v14.md` (or `discord-py.md`)
with cross-skill pointer to `llm-streaming-reply` (sibling sub-orchestrator
in the same pack). Streaming-then-editMessage works for Discord's edit
window (15 min vs Telegram's no-window if the message is in the same
channel).

## Framework Decision (the only one we make for you)

| Need                                       | Pick                       | Why |
|--------------------------------------------|----------------------------|-----|
| Node.js / TypeScript, any size             | **discord.js v14**         | Most features, decorator and builder APIs, mature |
| Python, asyncio, prefer classes-as-cogs    | **discord.py 2.x**         | Compact syntax, cogs are natural unit |
| Polyglot (JS backend + Python ML side)     | discord.js                 | Easier side-channel integration |
| Restricted to no-deps / scripting          | raw REST API               | Not framework territory |

Don't mix. Pick one.

## Slash Commands vs Prefix Commands (default rule)

In 2026, **build slash-commands by default**. They're discoverable in the
client UI, validate inputs, and don't require the bot to read message
content (which Discord gates behind the privileged
`MESSAGE_CONTENT` intent — bot owners must enable it in the dev portal).

Prefix commands (e.g. `!ban user`) require `MESSAGE_CONTENT` and are
de facto deprecated for new bots. They still work but the audit-trail /
flag / future-proofing case for slash is strong.

## LLM Streaming Forward Pointer

If your bot replies via LLM:

1. Acknowledge the interaction immediately with `deferReply()` (gives
   you 15 minutes for the full response).
2. `editReply()` in a loop as tokens arrive. Discord doesn't cap edits
   the way Telegram does — you have ~15 minutes from interaction time.
3. If you'd exceed 2000 chars (Discord message limit), split into
   multiple `followup.send()` calls.

Full streaming digest lives in the sibling `llm-streaming-reply` skill
within this same pack.

## Common Pitfalls

1. **Forgetting to add the `applications.commands` OAuth scope.** Bot
   works for gateway events but slash commands show up as "This
   command failed" — the user-side slash registration is incomplete.
   See `references/discord-permissions-and-scopes.md`.
2. **`SlashCommandBuilder()` created but never registered with the
   client.** A common mistake: construct the builder, print it as JSON,
   forget to `client.commands.set(name, command)`. Commands exist in
   memory but aren't part of `CommandsArray`. Always loop and register.
3. **`applicationId` confusion.** Discord identifies the bot two ways:
   your **user ID** (visible in dev portal) AND your **application ID**
   (often the same). Some endpoints require the application ID, some
   require the user/bot ID. Easy to swap. Verify in the dev portal.
4. **Editing a message in a different channel than the reply.** You
   can only `editReply` the original interaction message. Use
   `followup.send` for new messages.
5. **Permission bitset math error.** OR-ing the wrong bit produces
   silent failures at the API level. Use
   `PermissionsBitField.resolve([...])` (discord.js) or
   `discord.Permissions(...).value` (discord.py). Don't hand-roll.
6. **Not handling `InteractionNotFound` / `UnknownInteraction`.**
   If your handler runs >3s and Discord has already invalidated the
   interaction token, edits/followups fail with 404. Always
   `deferReply()` first.
7. **Guild-vs-global command deployment wrong scope.** Test commands on
   a hidden guild (instant), push to global later. Mixing guild IDs
   across envs causes "works on my dev guild, fails on server X"
   mysteriousness.

## Verification Checklist

- [ ] Bot token stored in environment variable, never committed
- [ ] OAuth invite URL contains `bot applications.commands` scopes
     (minimum for slash)
- [ ] Slash commands register successfully (`$application info` shows
     them; or REST: `GET /applications/{id}/commands`)
- [ ] Bot responds to `/ping` within 5s of being added to a guild
- [ ] Logs show guild_id, channel_id, command_name for test interaction
- [ ] Permissions properly resolved (slash command dropdown shows the
     right perms in the client UI)
- [ ] If using privileged intents (`MESSAGE_CONTENT`, `PRESENCE`), they
     are toggled enabled in dev portal AND included in gateway `identify`

## Related Skills

- `telegram-bot-dev` — Telegram-side companion (this is the Discord
  analog for bot frameworks)
- `llm-streaming-reply` — cross-platform streaming pattern for LLM
  replies
- `serverless-bot-deploy` — when webhook target is CF Workers, Vercel,
  or AWS Lambda
- `bot-async-runtime` — async-runtime gotchas

## Sources cited inline

- Discord developer docs — https://discord.com/developers/docs
  (canonical, all framework-independent API reference)
- discord.js guide — https://discordjs.guide (canonical for the JS lib)
- discord.js GitHub — https://github.com/discordjs/discord.js
- discord.py docs — https://discordpy.readthedocs.io/
- discord.py GitHub — https://github.com/Rapptz/discord.py
- Discord developer portal — https://discord.com/developers/applications
- Discord interactions webhook guidance — covered inline

We do **not** redistribute upstream docs. Every figure/number/idiom above
is opinionated digest text authored here.
