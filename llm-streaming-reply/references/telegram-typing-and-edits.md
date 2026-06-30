---
name: telegram-typing-and-edits
description: >-
  Companion digest for the "streaming into Telegram" pattern. Send
  placeholder + editMessage chunks + sendChatAction typing indicator +
  rate-limit-aware batching. Threshold for switching from editMessage
  to a new message at 4096 chars.
version: 0.1.0
user-invocable: false
author: neutralthiscrazy
license: MIT
---

# Telegram — typing indicator + editMessage streaming

## When to load

- Streaming an LLM reply into Telegram
- Showing "is typing" while a long reply is being generated
- Hitting 4096-char cap and chunking into multiple messages
- Edit-rate throttling (Telegram allows edits every ~1s)

## Setup

```python
chat_id = message.chat.id
BOT = message.bot  # aiogram

# 1) Send placeholder immediately
sent = await BOT.send_message(chat_id, "…", parse_mode="HTML")

# 2) Start typing indicator (5-second TTL)
typing_task = asyncio.create_task(repeat_typing(BOT, chat_id))

# 3) Process LLM stream
buffer = ""
last_edit = time.monotonic()
async for delta in llm_stream_yield(user_text):
    buffer += delta
    # Edit at most every 1s
    if time.monotonic() - last_edit > 1.0:
        if len(buffer) <= 4096:
            await BOT.edit_message_text(
                chat_id=chat_id, message_id=sent.message_id,
                text=buffer, parse_mode="HTML"
            )
        last_edit = time.monotonic()
        # Hit 4096, send a new continuation message
        if len(buffer) > 4096:
            sent = await BOT.send_message(chat_id, "...", parse_mode="HTML")
            buffer = buffer[4096:]

# 4) Stop typing
typing_task.cancel()

# 5) Final flush
await BOT.edit_message_text(chat_id=chat_id, message_id=sent.message_id,
                            text=buffer, parse_mode="HTML")
```

## Typing indicator loop

Telegram `sendChatAction(action='typing')` lasts 5 seconds. For a
30-second stream, repeat it every 4s:

```python
async def repeat_typing(bot, chat_id):
    while True:
        try:
            await bot.send_chat_action(chat_id, action="typing")
            await asyncio.sleep(4)  # < 5s TTL
        except asyncio.CancelledError:
            break  # graceful exit on cancel
```

## Markdown / HTML safety mid-stream

Editing messages in MarkdownV2 with stray backslashes mid-stream will
fail. Prefer HTML:

```python
def html_escape(text: str) -> str:
    return text.replace("&", "&amp;").replace("<", "&lt;").replace(">", "&gt;")
```

Open code blocks as `<pre>` and close them only in the FINAL edit —
don't append `</pre>` mid-stream or the closing tag might end up in
a half-edit chunk.

## Chunking at 4096 char limit

Telegram messages cap at 4096 chars. For longer responses, manually
split (don't trust the API to): every 4096 chars emit a new
`send_message`. Carry an internal "page N of M" indicator if user
expects multiple messages.

If your LLM response can have FOOTERS like "Sources:" / "Tools used:"
or full-form markdown tables — they pass through. Just don't end on
a half-finished bullet list or table.

## Rate-limit math

Telegram edit rate is conceptually unbounded but practically — at
high edit rates (5+ per second per message_id), HTTP responses slow
and edits may race. Stick to one edit per 1.0–1.5 seconds.

## Common pitfalls

- `editMessageText` on a stale message_id (message deleted) raises
  `MessageCantEdit`. Wrap in try/except; if raises, send a new
  message with the buffered text instead.
- format issue with parse_mode. Once parse_mode is set on a message,
  changing it in advance edit breaks things.
- `sendChatAction` errors on closed bot-initiated chats → ignore.
- Mid-stream `MessageIsTooLong` — happens if you push 4096+ chars
  via editMessageText. Catch and split into a new message.
- `RetryAfter` flood control — if your handler streams many chat
  requests, strict `sleep_retry_after` is mandatory.

## Sources

- sendMessage — https://core.telegram.org/bots/api#sendmessage
- editMessageText — https://core.telegram.org/bots/api#editmessagetext
- sendChatAction — https://core.telegram.org/bots/api#sendchataction
- Telegram message-length cap (4096) — https://core.telegram.org/bots/api#sendmessage
