---
name: telegram-bot-dev
description: >-
  Use when developing Telegram bots in Python or TypeScript. Picks framework
  (aiogram 3.x for Python production bots, grammy for TypeScript-first or
  serverless, python-telebot for trivial single-purpose), explains polling
  vs webhook with the serverless/webhook rule, guides FSM design (aiogram
  StatesGroup or grammy conversations plugin), flags LLM-streaming integration
  for AI assistants, and points to official docs inline instead of duplicating
  them. Routing table loads exactly one reference per intent.
version: 0.1.0
user-invocable: false
author: neutralthiscrazy
license: MIT
---

# Telegram Bot Development

## Overview

Opinionated companion for building Telegram bots in 2026. We **do not**
wholesale-copy official documentation — every reference file is a digest
that links you to the canonical sources (`aiogram.dev`, `grammy.dev`,
`core.telegram.org/bots/api`) for deeper reading. Three framework choices
ship as separate reference files; pick the one matching your stack, not
all three.

Authored from first-party docs and battle-tested patterns. License is MIT
on our digest text; cited upstream sources retain their own licenses (BSD
for aiogram, MIT for grammy, GPL for pyTelegramBotAPI — see first link
in each reference before redistributing).

## When to Use

- Choosing between Telegram bot frameworks (aiogram / grammy / python-telebot)
- Deciding polling vs webhook for your deployment target
- Designing conversation flows with finite state machines
- Adding an LLM/chat-GPT backend that streams replies token-by-token
- Debugging handler ordering or webhook misconfiguration
- Need a quick opinionated overview before diving into upstream docs

## When NOT to Use

- Pure admin tasks (rate limits are visible in BotFather dashboard)
- Channel/forum management without a bot (use Telegram desktop + manual ops)
- Inline-mode deep dives — only basics touched here
- Bot payments / Telegram Stars / TON integration (out of scope)
- iOS/Android mobile app development (use a different pack)

## Routing Table

| User intent                                  | Load                                  | Notes |
|----------------------------------------------|---------------------------------------|-------|
| "Build a Python bot, async, production"      | `references/aiogram-3x.md`            | Plus choose webhook-vs-polling |
| "Build a TS bot, serverless deploy"          | `references/grammy.md`                | Plus choose webhook-vs-polling |
| "Small/sync Python bot, just a few commands" | `references/python-telebot.md`        | Pick polling only — sync model |
| "Design conversation with states/memory"     | `references/telegram-fsm-patterns.md` | Pair with framework reference |
| "Webhook or polling for my hosting?"         | `references/webhook-vs-polling.md`    | Decision tree inside |

If the task is "stream LLM tokens into Telegram edits" — pair
`aiogram-3x.md` (or `grammy.md`) with cross-skill pointer to
`llm-streaming-reply` (sibling sub-orchestrator in the same pack).
That pattern is verbose enough to deserve its own digest.

## Framework Decision (the only one we make for you)

| Need                                       | Pick                       | Why |
|--------------------------------------------|----------------------------|-----|
| Python, async, any size, production-ready  | **aiogram 3.x**            | Active, large community, FSM built-in, mature middleware, BSD license |
| TypeScript, serverless (Vercel, CF Workers)| **grammy**                 | ESM-first, `webhookCallback` adapter for serverless, conversations plugin |
| Small Python bot, sync, <10 commands       | **python-telebot**         | Decorator API, zero ceremony, no asyncio required |
| High-volume (10k+ RPS) deep control        | raw `Bot` API + custom HTTP | Not framework territory |

Don't mix. Pick one and own its conventions. Switching frameworks mid-project
costs more than rewriting.

## Polling vs Webhook Rule

- **Serverless / dynamic hosting** (Vercel, Cloudflare Workers, AWS Lambda):
  **webhook only**. There is no long-running process to poll.
- **Traditional VPS / dedicated container**: either works; webhook saves
  outbound traffic and Telegram pushes updates the instant they arrive.
- **Local dev on your laptop**: polling is easier (no HTTPS tunnel),
  unless you already have `ngrok` running.
- **Edge cases** (multi-region failover, behind some reverse proxies):
  polling is more forgiving because there's no public endpoint that
  breaks on cert rotation.

See `references/webhook-vs-polling.md` for the full tradeoff matrix
including HTTPS/cert pain and the famous "set_webhook returns True but
updates still don't arrive" gotcha.

## LLM Streaming Forward Pointer

If your bot replies with an LLM, three things matter for Telegram:

1. **Show "is typing"** (`sendChatAction` with `action='typing'`, refresh
   it every ~5s while the model runs — Telegram expires the indicator).
2. **Stream into edits** (`sendMessage` first, then `editMessageText` in
   a loop as tokens arrive). Don't open a flood of separate messages.
3. **Token budget** — a 4096-token reply hits Telegram's 4096-char
   message limit. Plan chunking, or switch to media attachments.

Full digest for streaming across Telegram/Discord lives in the sibling
`llm-streaming-reply` sub-orchestrator. Don't duplicate here.

## Common Pitfalls

1. **Picking aiogram 2 patterns for aiogram 3.x.** v2 had `@dp.message_handler()`;
   v3 uses router-based pattern (`@router.message(F.text)`). Old tutorials will
   teach the wrong API. Always check `aiogram.dev`'s version selector — it's
   pinned to current.
2. **Forgetting to call `dp.include_router(...)` in v3.** Handlers in a router
   that you never include are silently dead. Run it once at startup.
3. **Webhook handler ignores `Content-Type: application/json` leading zeros.**
   Some cloud functions strip bodies — `request.json()` returns `None` silently
   and Telegram retries the same update, eventually flooding your logs.
4. **Long-running handlers without `asyncio.shield()` on LLM calls.** If the
   upstream LLM API rate-limits you, your handler gets a CancelledError during
   graceful shutdown but the LLM still billed you. Use `shield()` around
   network I/O.
5. **`bot.send_chat_action(action='typing')` only lasts 5 seconds.** Telegram
   expires it. For replies longer than 5s, send it every 4s in a background
   task, or just send a "thinking…" placeholder message first.
6. **Mixing `ReplyKeyboardMarkup` (full keyboard replacement) and
   `InlineKeyboardMarkup` (button-attached-to-message) assumptions.** Reply
   keyboards come back as regular text messages; inline queries come back as
   `callback_query` updates. They're different handlers entirely.
7. **Putting bot TOKEN in source code.** Use `.env` + `pydantic-settings`
   (Python) or `dotenv` (Node). If the bot is open-source on a public repo,
   the token leaks and you have to revoke at `@BotFather`.

## Verification Checklist

- [ ] `references/<picked-framework>.md` loaded end-to-end (read once before coding)
- [ ] `references/webhook-vs-polling.md` decision applied to your deployment
- [ ] Bot token stored in environment variable, never in source
- [ ] Smoke-test: `getMe()` returns the bot's username — confirms token + reachability
- [ ] Smoke-test: send `/start` to your bot, see handler firings in logs
- [ ] If webhook: `getWebhookInfo()` from BotFather → expected URL, no pending updates
- [ ] If polling: `deleteWebhook(drop_pending_updates=True)` before `start_polling`
- [ ] Logs show first user_id of test sender — confirms update pipeline intact

## One-Shot Recipes

**Python aiogram prod bot on VPS (polling):**

```bash
pip install aiogram==3.*
python -c "from aiogram import Bot; import asyncio; \
asyncio.run(Bot(token='$TG_TOKEN').get_me())"
```

Then see `references/aiogram-3x.md` for full router setup.

**TS grammy on Vercel (webhook):**

```bash
npm init -y && npm i grammy && npm i -D @vercel/node typescript
```

Then see `references/grammy.md`.

**Webhook smoke from local dev (ngrok):**

```bash
ngrok http 8080
# copy https URL, setWebhook to that + /<route>
curl "https://api.telegram.org/bot$TOKEN/setWebhook?url=https://<ngrok>"
```

## Related Skills

- `discord-bot-dev` — Discord side (slash-commands vs prefix, interactions vs
  messageContent intent)
- `llm-streaming-reply` — cross-platform streaming pattern for LLM replies
- `serverless-bot-deploy` — when webhook target is Cloudflare Workers, Vercel,
  or AWS Lambda
- `bot-async-runtime` — Python asyncio and Node.js async runtime gotchas
  that bite bot code in particular

## Sources cited inline

- aiogram — official docs at https://aiogram.dev/ (BSD-3-Clause), GitHub
  https://github.com/aiogram/aiogram
- grammy — official docs at https://grammy.dev/ (MIT), GitHub
  https://github.com/grammyjs/grammY
- python-telebot (pyTelegramBotAPI) — https://github.com/eternnoir/pyTelegramBotAPI
  (GPL-3.0 — note license before choosing for commercial projects)
- Telegram Bot API reference — https://core.telegram.org/bots/api (canonical
  for any framework: the only source of truth for what fields exist)

We do **not** redistribute upstream docs. Every figure/number/idiom above is
opinionated digest text authored here. Upstream retains copyright; see
ATTRIBUTIONS.md at repo root for legal basis.
