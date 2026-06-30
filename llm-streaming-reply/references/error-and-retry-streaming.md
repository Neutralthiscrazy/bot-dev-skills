---
name: error-and-retry-streaming
description: >-
  Companion digest for handling mid-stream errors during LLM → chat-bot
  streaming. Categories of failures, partial-output vs full-reset
  strategies, retry policies, and how to surface errors to users
  without losing the partial reply that was already shown.
version: 0.1.0
user-invocable: false
author: neutralthiscrazy
license: MIT
---

# Mid-stream errors and retries

## When to load

- LLM stream breaks mid-response (timeout, 5xx, schema mismatch)
- Some chunk edits succeed but later ones fail
- Deciding "show partial + error" vs "reset and reply"
- Chat-platform loss during streaming (rate limit, message deleted)
- Adding a new chat platform adapter (Slack, Matrix, etc.) — copy
  the same failure model

## Three categorisations of error

| Class            | Examples                                | Recovery                              |
|------------------|-----------------------------------------|---------------------------------------|
| Pre-stream       | Auth, network, 5xx, quota              | Retry with backoff                    |
| Mid-stream       | Timeout, partial SSE drop, schema drift| Show partial, append error note       |
| Post-stream      | Final edit fails, message deleted       | Send a fresh message with body       |

**Pre-stream** is the easy one — call again or refuse to start.
The interesting design choice is **mid-stream**: do you show the
partial text already visible to the user, or do you apologize and
restart? Most users prefer "partial + apology" because the
alternative feels like nothing happened.

## Mid-stream LLM error pattern

```python
async def stream_with_error_recovery(message, llm_stream, edit_fn):
    buffer = ""
    try:
        async for delta in llm_stream:
            buffer += delta
            await maybe_flush(buffer, edit_fn)
    except httpx.RequestError:
        # partial output; tell user
        await edit_fn(buffer + "

_⚠️ Connection issue — answer truncated._")
    except anthropic.APIStatusError as e:
        if "rate_limit" in str(e):
            await edit_fn(buffer + "

_⚠️ Rate limited — try in a moment._")
        else:
            await edit_fn(buffer + "

_⚠️ Server error — partial answer above._")
    except asyncio.TimeoutError:
        await edit_fn(buffer + "

_⚠️ Timed out — partial answer above._")
```

The trade-off is: showing partial + error note means user gets some
useful text. Discarding partial and showing only "error" means user
gets nothing useful. Most users prefer partial.

## Adding a new platform adapter

The three-place pattern for any chat platform:

1. **Placeholder send**: returns message ID for editing.
2. **Edit/flush**: push accumulated buffer; throttle per platform.
3. **Error handler**: same shape — partial + note.

A matrix / Slack / Matrix bridge adapter:

```python
class SlackAdapter:
    async def placeholder(self, channel): ...  # chat.postMessage
    async def edit(self, channel, ts, text): ...  # chat.update
    async def error_msg(self, channel, ts, error_text): ...
```

Each platform has its own quirks — Slack's `chat.update` requires
the original `ts` argument, Telegram requires the message_id,
Discord requires the interaction token. The error-handling wrapper
is identical.

## Retry strategies

For mid-stream errors that are clearly transient (network blip):

1. **In-place retry**: re-request the LLM with the same prompt BUT
   include the partial reply so far as "previous answer so far", then
   continue the stream from there. This is named "regeneration from
   partial" and works well for long responses.

```python
prompt_with_partial = (
    f"{original_prompt}

"
    f"Partial answer so far: {buffer}

"
    "Continue from where you left off, don't repeat content."
)
```

2. **Clean restart**: message the user "sorry, let me try again"
   and call the LLM fresh. Avoid mixing partial and re-generated —
   quality drops because the model doesn't know the original tone.

## Final edit can fail

If `editReply` / `editMessageText` raises on the final flush (after
the LLM stream completed successfully), fall back to a fresh
`send_message` with the buffer:

```python
try:
    await edit_fn(buffer)
except (MessageNotModified, MessageCantEdit, NotFound):
    send_fn(buffer)  # telegram `sendMessage`, discord `followUp`
```

Don't surface "edit failed" to user — the answer is visible via the
fallback send.

## Pitfalls

- In-place retry with "continue from partial" prompt usually produces
  degraded quality — the model doesn't have the original tone / style.
- TOKEN billing on retries: re-request that consumed tokens in the
  first run STILL count. Don't retry huge prompts carelessly.
- Chat platform's rate limits — during a stream you may hit
  flood-wait. The right answer is to slow down (don't pause the
  retry, you can wait).
- "Continue from partial" prompt can be re-poisoned if the partial
  contained the user's PII back to the LLM through the prompt — in
  shared chat context this leaks.

## Sources

- Anthropic retry behavior — https://docs.anthropic.com/en/api/errors
- OpenAI error codes — https://platform.openai.com/docs/guides/error-codes/api-errors
- Telegram flood-wait — https://core.telegram.org/bots/api#making-requests-when-getting-flood-wait
- Discord 429 handling — https://discord.com/developers/docs/topics/rate-limits#rate-limit-headers

Run all the writes
