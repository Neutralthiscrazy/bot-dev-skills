---
name: openai-streaming
description: >-
  Companion digest for OpenAI streaming completions and chat. Setup,
  the SSE stream format, parse with httpx/AsyncOpenAI SDK, handling
  function-call deltas, and tool-use streaming. Pair with chat-platform
  adapters for end-to-end integration.
version: 0.1.0
user-invocable: false
author: neutralthiscrazy
license: MIT
---

# OpenAI streaming (completions + chat + responses)

## When to load

- Bot needs to stream completions from OpenAI (gpt-4, gpt-4o, gpt-4-turbo)
- Tool/function-calling with streaming
- The `responses` endpoint (newer API)

If using Claude, see `anthropic-streaming.md` instead. If using a local
model, the protocol differs — OpenAI-compatible servers (vLLM, llama.cpp)
use the same shape but with custom endpoint.

## Python: streaming chat completions

```python
import asyncio
from openai import AsyncOpenAI

client = AsyncOpenAI(api_key=os.environ["OPENAI_API_KEY"])

async def stream_reply(user_text: str) -> str:
    final = []
    async for chunk in await client.chat.completions.create(
        model="gpt-4o",
        messages=[{"role": "user", "content": user_text}],
        stream=True,
    ):
        delta = chunk.choices[0].delta.content or ""
        final.append(delta)
    return "".join(final)
```

To process incrementally (vs collect-all-then-return), yield from
the loop:

```python
async def stream_yield(user_text):
    stream = await client.chat.completions.create(
        model="gpt-4o",
        messages=[{"role": "user", "content": user_text}],
        stream=True,
    )
    async for chunk in stream:
        delta = chunk.choices[0].delta.content
        if delta:
            yield delta
```

## SSE wire format

OpenAI streams over Server-Sent Events (SSE): each token arrives as
`data: <json>

` line, terminated by `data: [DONE]

`. The openai
SDK parses this for you. If using raw httpx, do `response.aiter_lines()`
and check for the `data:` prefix.

```python
async with httpx.AsyncClient() as cli:
    async with cli.stream(
        "POST", "https://api.openai.com/v1/chat/completions",
        headers={"Authorization": f"Bearer {API}"},
        json={"model": "gpt-4o", "messages": [...], "stream": True},
        timeout=None,
    ) as r:
        async for line in r.aiter_lines():
            if line.startswith("data: ") and line != "data: [DONE]":
                chunk = json.loads(line[6:])
                delta = chunk["choices"][0]["delta"].get("content", "")
                yield delta
```

If you're embedding the LLM in a Discord/Telegram bot, prefer the SDK
over raw httpx — error parsing, retries, and tool/function-call streaming
are already done.

## Function-calling stream

When the assistant is calling tools, `delta` carries `tool_calls` deltas
that have to be accumulated:

```python
tool_call_buffers = {}  # index -> (name, args_str)
async for chunk in stream:
    for tc in chunk.choices[0].delta.tool_calls or []:
        idx = tc.index
        if idx not in tool_call_buffers:
            tool_call_buffers[idx] = (tc.function.name, "")
        if tc.function.arguments:
            tool_call_buffers[idx] = (
                tool_call_buffers[idx][0],
                tool_call_buffers[idx][1] + tc.function.arguments,
            )
```

Final parse: `json.loads(tool_call_buffers[idx][1])` once the stream
ends. Parse args as JSON only after the full delta is collected.

## Cost and billing

OpenAI bills for `prompt_tokens + completion_tokens`, including tokens
generated but then discarded (which happens when the user closes the
chat mid-stream). Three controls:

- `max_tokens=N` — upper bound, hard cap.
- `stream=True` with `'stream_options': {'include_usage': True}` — final
  chunk carries `usage` data; the SDK can return total billable.
- Streaming avoids waiting on sync responses; partial reads are fine if
  billed tokens are acceptable.

## Common pitfalls

- Client timeout. Default 60s may be too short for long completions.
  Set `timeout=None` in the SDK call OR a long `httpx.Timeout(...)`.
- `delta.role` only present in the first chunk. Don't trigger
  "is assistant" logic on every chunk.
- `delta.content` can be None on tool-call chunks. Use `or ""` fallback.
- Unexpected `finish_reason`. `length` means the model capped at
  `max_tokens`. Adjust or accept partial.
- Empty choices (`chunk.choices == []`) appear in `usage`-only final
  chunks when `include_usage=True` — guard against `IndexError`.

## Sources

- OpenAI streaming reference — https://platform.openai.com/docs/api-reference/streaming
- Chat completions — https://platform.openai.com/docs/api-reference/chat
- `include_usage` option — https://platform.openai.com/docs/api-reference/chat/streaming
- SDK async streaming — https://github.com/openai/openai-python#async-usage
