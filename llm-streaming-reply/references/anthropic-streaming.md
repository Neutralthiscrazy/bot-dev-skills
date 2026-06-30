---
name: anthropic-streaming
description: >-
  Companion digest for Anthropic streaming (Claude). SSE format,
  message-delta events (content_block_delta, message_delta, message_stop),
  async SDK usage, prompt caching with streaming, and the differences
  from OpenAI's API surface.
version: 0.1.0
user-invocable: false
author: neutralthiscrazy
license: MIT
---

# Anthropic streaming (Claude messages)

## When to load

- Streaming Claude completions into a chat bot
- Multipart content blocks (text + tool_use + image) streaming
- Prompt caching with stream-time benefits
- The newer "messages" streaming API (since 2024)

If using OpenAI, see `openai-streaming.md` instead.

## Python: streaming

```python
import os
import asyncio
from anthropic import AsyncAnthropic

client = AsyncAnthropic(api_key=os.environ["ANTHROPIC_API_KEY"])

async def stream_yield(user_text: str):
    async with client.messages.stream(
        model="claude-sonnet-4-5",
        max_tokens=2048,
        messages=[{"role": "user", "content": user_text}],
    ) as stream:
        async for text in stream.text_stream:
            yield text
```

Note: the SDK wraps the SSE stream and exposes `text_stream` (cleaner
filtered stream of just text deltas) and the full event iterator for
tool_use payloads.

## SSE event types

When consuming raw SSE (no SDK), the stream emits these event types:

| Event                       | Fields                                  | Use |
|-----------------------------|-----------------------------------------|-----|
| `message_start`             | message metadata                        | Init |
| `content_block_start`       | type (`text` / `tool_use` / …), index   | Begin block |
| `content_block_delta`       | type (`text_delta` / `input_json_delta`), delta | Accumulate |
| `content_block_stop`        | index                                   | Block done |
| `message_delta`             | delta with `stop_reason`, `usage`       | Final prior to stop |
| `message_stop`              | -                                       | Stream end |
| `ping`                      | -                                       | Keepalive, ignore |
| `error`                     | error payload                           | Surface |

A typical streaming run ("Hi!" reply) emits:

```
message_start
content_block_start (text, idx=0)
content_block_delta (text_delta "Hi", idx=0)
content_block_delta (text_delta "!", idx=0)
content_block_stop (idx=0)
message_delta (stop_reason="end_turn", usage={...})
message_stop
```

## Tool use with streaming

For tool use, `content_block` of type `tool_use` carries `input` as
JSON, emitted via accumulated `input_json_delta` events. Collect them
into a string and parse as JSON at `content_block_stop`:

```python
async for event in stream:
    if event.type == "content_block_start" and event.content_block.type == "tool_use":
        tool_name = event.content_block.name
        tool_id = event.content_block.id
    elif event.type == "content_block_delta" and event.type == "input_json_delta":
        json_buffer += event.delta.partial_json
    elif event.type == "content_block_stop" and event.content_block.type == "tool_use":
        tool_args = json.loads(json_buffer)
        # execute tool, follow up with tool_result message
```

Multi-turn with tool execution is more involved; SDK abstracts this.
When a tool_use result is fed back into the conversation, another
round of streaming starts.

## Prompt caching with streaming

If your prompt includes `cache_control: { type: "ephemeral" }` on
content blocks, Anthropic caches the prefix server-side. Subsequent
requests with the same prefix get up-to 90% discount on cached tokens.
Streaming doesn't change cache behavior; the cache is "write-on-write"
or "write-on-read" depending on settings.

## Common pitfalls

- `message_delta` carries `stop_reason: "max_tokens"` when you've hit
  limit. Stream cuts off; user-visible response is partial. Increase
  `max_tokens` or warn the user.
- `content_block_delta` deltas are sometimes empty (zero chars).
  Yield them anyway — your handler must skip empty deltas.
- Stream cancellation: `stream.close()` mid-response triggers
  proper cleanup. Without close, the connection might remain until
  the SDK times out.
- Python `async with client.messages.stream()` keeps the connection
  open. Don't await on `stream.text_stream` indefinitely; it can stall
  on error boundaries.
- Anthropic API key vs bearer token — both supported, but `client.api_key`
  is the helper for the SDK.

## Sources

- Anthropic streaming docs — https://docs.anthropic.com/en/api/streaming
- Messages API — https://docs.anthropic.com/en/api/messages
- Tool use streaming — https://docs.anthropic.com/en/docs/tool-use
- Prompt caching — https://docs.anthropic.com/en/docs/build-with-claude/prompt-caching
- Python SDK — https://github.com/anthropics/anthropic-sdk-python
