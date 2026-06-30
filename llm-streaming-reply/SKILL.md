---
name: llm-streaming-reply
description: >-
  Use when building a bot that streams LLM tokens into chat messages.
  Covers OpenAI and Anthropic streaming APIs, the "send placeholder, then
  editMessage/followup" pattern common to all chat platforms, the typing
  indicator / chunking / rate-limit triangle, and graceful handling of
  mid-stream failure. Routing table picks the right reference per
  platform + per concern.
version: 0.1.0
user-invocable: false
author: neutralthiscrazy
license: MIT
---

# LLM Streaming Reply — chatbot-side integration

## Overview

Companion for the **"LLM is typing, then answers long-form"** pattern that
every modern AI chatbot eventually implements. We **do not** cover how to
train a model or prompt-engineer a system message — we cover the chat-bot-
side glue: receiving streamed tokens from the upstream LLM API and turning
them into live-updating chat messages on Telegram / Discord / WebSocket /
etc.

Two primary streaming APIs (OpenAI + Anthropic), plus chat-platform-
specific adapters. Routing table below maps intent → reference.

Authored from first-party docs (`platform.openai.com/docs/api-reference`,
`docs.anthropic.com/en/api/streaming`) and battle-tested patterns. License
is MIT on our digest text. Upstream sources retain their own API terms.

## When to Use

- Building a chatbot that talks to an LLM (GPT-4, Claude, etc.)
- Token-streaming into a Telegram edit / Discord edit / followup
- Showing "is typing" while a long reply is being generated
- Hitting Telegram 4096-char or Discord 2000-char per-message limits
- Implementing retries on mid-stream LLM errors
- Implementing token-budget-side guard (don't generate past 2048 tokens
  in one message)

## When NOT to Use

- Static bot responses (no LLM, no streaming) — skip
- Audio / speech synthesis endpoints — different API surface
- Fine-tuning or training runs — not in scope
- Single-shot completion (you don't need streaming) — use the simpler
  non-streaming endpoint

## Routing Table

| User intent                                        | Load                                       |
|----------------------------------------------------|--------------------------------------------|
| "Stream OpenAI completions / chat"                 | `references/openai-streaming.md`           |
| "Stream Anthropic / Claude messages"               | `references/anthropic-streaming.md`        |
| "Telegram-specific: editMessage chunking"          | `references/telegram-typing-and-edits.md`  |
| "Discord-specific: typed-chunked followups"        | `references/discord-typed-chunks.md`       |
| "Rate limit / token budget / chunk math"           | `references/rate-limit-and-token-budget.md` |
| "Mid-stream error, retry, graceful fallback"      | `references/error-and-retry-streaming.md`  |

If you want to pair a notifier (Slack, Matrix, etc.) — covered briefly in
`error-and-retry-streaming.md` under "Adding a new platform adapter".

## Universal pattern (the one we make for you)

For any chat platform + any LLM, the pattern is:

1. **Send placeholder immediately.** Telegram: `sendMessage("...")`,
   Discord: `interaction.reply("...")` or `followup.send("...")`. Returns
   a message_id you can edit.
2. **Start "is typing" / "thinking…" indicator.** Telegram: `sendChatAction`
   + a ~5s repeating timer. Discord: ephemeral followup or just a
   "thinking…" message in the original reply.
3. **Pull stream from LLM, chunk into ~50-token pieces.** Buffer tokens,
   flush to editMessage when buffer > threshold. Most platforms allow
   edits every ~1 second; faster edits hit rate limits.
4. **Hit message-char cap → start a new message.** Telegram: new message
   every 4096 chars. Discord: new `followup.send` every 2000 chars. Web:
   add a "show more" element.
5. **Final flush on `stream.finish`.** Last editMessage with full body,
   stop typing indicator.
6. **On error mid-stream:** tell user "sorry, hit a snag", but don't
   lose what was already shown. Position-based recovery is tricky;
   trade-off is "show partial + apology" vs "don't show partial".

The chat-platform-specific references below adapt this template.

## Common Pitfalls (cross-platform)

1. **Editing too fast.** Many chat APIs throttle edits at 30/minute per
   message. Buffer tokens, flush every 1.0-1.5s, and accept slight lag.
2. **`sendChatAction(action='typing')` only lasts 5 seconds on Telegram.**
   Repeat every 4s in a background task.
3. **Code blocks / markdown mid-stream.** Editing MarkdownV2 is fine as
   long as the unfinished chunk doesn't include malformed escape
   sequences. With HTML mode, escape `<` `>` `&` first.
4. **LLM timeout ≠ API error.** Some upstreams have 60-90s timeouts for
   long responses — your code must either chunk the request (use
   shorter prompts) or accept a final cut-off.
5. **Token billing on cancelled streams.** Most LLM providers bill for
   `prompt + completion tokens` even if you stop reading mid-stream.
   Cap prompt length and pick `max_tokens` to bound cost.
6. **Concurrent streams per user.** A user typing in 3 different chats
   with the same bot should not interfere. Key streams by user_id or
   chat_id in your internal router.

## Verification Checklist

- [ ] Placeholder message created <2s after user input
- [ ] Typing indicator visible by 5s — confirms streaming is alive
- [ ] Edit rate ~1/second average — confirms rate-limit discipline
- [ ] 4096-char (Telegram) / 2000-char (Discord) cutover triggers a new
      message — confirms chunk logic
- [ ] On forced LLM error, partial output preserved + error message
      appended — confirms graceful fallback
- [ ] Per-user rate limiter prevents accidental spam (e.g. 5 questions
      in 1 minute = 1 in queue)

## Related Skills

- `telegram-bot-dev` — Telegram-side framework choice
- `discord-bot-dev` — Discord-side framework choice
- `serverless-bot-deploy` — webhook-only delivery for LLM bot platforms
- `bot-async-runtime` — Python asyncio / Node.js stream-loop gotchas

## Sources cited inline

- OpenAI streaming — https://platform.openai.com/docs/api-reference/streaming
- Anthropic streaming — https://docs.anthropic.com/en/api/streaming
- OpenAI chat completions — https://platform.openai.com/docs/api-reference/chat
- Anthropic messages — https://docs.anthropic.com/en/api/messages
- Chat platforms' own edit-rate documents (Telegram, Discord per-link below)
