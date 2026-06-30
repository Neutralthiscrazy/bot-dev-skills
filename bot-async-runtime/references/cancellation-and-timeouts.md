---
name: cancellation-and-timeouts
description: >-
  Companion digest for cancellation and timeouts in chat bots.
  asyncio.shield vs Task.cancel, AbortController patterns, the
  timeout-after-task dance, and the "billing on cancelled streams"
  trap for LLM integration.
version: 0.1.0
user-invocable: false
author: neutralthiscrazy
license: MIT
---

# Cancellation and timeouts

## When to load

- LLM call ran past 30s — you want to bail but still bill correctly
- User pressed /cancel mid-stream — propagate cancellation cleanly
- Webhook arrives past 15s — Telegram / Discord has invalidated the
  interaction; editing will fail

## Python: `asyncio.wait_for` (broad timeout)

```python
try:
    result = await asyncio.wait_for(coro(), timeout=30)
except asyncio.TimeoutError:
    # coro was cancelled
    ...
```

`wait_for` cancels the inner coro after `timeout` seconds and
raises `TimeoutError` (Python 3.9 had `asyncio.TimeoutError` —
Python 3.11 made it an alias for `TimeoutError`). Use this when
you DO want to abort the operation entirely.

## Python: `asyncio.shield` (don't cancel)

```python
try:
    result = await asyncio.shield(coro(), timeout=30)
except asyncio.TimeoutError:
    # The shield timed out — but coro keeps running
    # If you want to clean up, await the inner task manually
    ...
```

Shield is right for cases where CORO must complete (e.g. for
billing/correctness), but you also want a timeout on the OUTER
caller. Be careful — without awaiting the inner, you leak the
task into runtime.

Better pattern:

```python
async def with_timeout(coro, timeout):
    task = asyncio.create_task(coro)
    try:
        return await asyncio.wait_for(asyncio.shield(task), timeout)
    except asyncio.TimeoutError:
        return None  # task continues
```

## Node: AbortController

Already covered in `nodejs-eventloop.md`. Pattern:

```javascript
const controller = new AbortController();
setTimeout(() => controller.abort(), 30000);

try {
  await fetch(url, { signal: controller.signal });
} catch (e) {
  if (e.name === "AbortError") {
    // timed out
  }
}
```

For multiple in-flight operations, share the same controller.

## Bot-specific: webhook timeout

Webhook-mode bots have a platform-set deadline:

- **Telegram**: 5s before sendChatAction/editMessageText feels laggy
- **Discord** (deferred): 15 minutes from deferReply
- **CF Workers**: 30s CPU per request (paid) / 10s (free)

If your handler hits this deadline:

- Clone the data you need, return ack
- Schedule background completion (CF Cron / Vercel Background
  Function / Kafka / SQS)
- Background job sends the slow edit

Don't try to complete synchronously past the deadline. The platform
will reject your edit/followup with `Unknown Interaction`.

## "Billing on cancelled streams"

If user cancels mid-LLM-stream:

- The model has already generated partial tokens.
- Most providers (OpenAI, Anthropic) bill those tokens even if your
  handler never reads them.
- Reading the partial output (drain the stream) doesn't change
  the bill — useful only if you want to know the partial cost.

So: don't worry about "saving cost" by cancelling the consumer
side. The cost is the same. Just clean up so the stream isn't
leaked (open HTTP connection).

## Common pitfalls

- Catching `Exception` (not `BaseException`) → `CancelledError`
  (Python) won't be caught. Cancellation is intentional.
- `task.cancel(); return` — task keeps running, may log warnings.
  Always `await task` after cancelling.
- Wrapping `wait_for` around a sync call — no effect; wait_for only
  cancels awaitables.

## Sources

- asyncio cancellation — https://docs.python.org/3/library/asyncio-task.html#task-cancellation
- Node AbortController — https://developer.mozilla.org/en-US/docs/Web/API/AbortController
- Discord interaction timeout — https://discord.com/developers/docs/interactions/slash-commands#responding-to-an-interaction
- Telegram sendMessage rate — https://core.telegram.org/bots/api#sendmessage
- Anthropic streaming cancel — https://docs.anthropic.com/en/api/streaming
