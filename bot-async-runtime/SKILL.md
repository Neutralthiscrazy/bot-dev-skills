---
name: bot-async-runtime
description: >-
  Use when your bot code uses asyncio (Python) or the Node.js event
  loop and you hit a runtime gotcha - cancelled tasks that should
  not have cancelled, blocking the polling loop, race conditions in
  handler ordering, or graceful-shutdown patterns. Covers Python
  asyncio + Node.js event loop in parallel, with bot-side examples.
version: 0.1.0
user-invocable: false
author: neutralthiscrazy
license: MIT
---

# Bot async runtime — Python asyncio + Node.js event loop

## Overview

Companion for the **async-runtime layer of a chat bot**. Frameworks
(aiogram, grammy, discord.py, discord.js) handle platform-level
async; this skill covers the runtime underneath.

Two runtimes covered in parallel:
- **Python**: asyncio (built-in since 3.4, default since 3.10)
- **Node.js**: single-threaded event loop with libuv, microtask queue

Both have specific bot-side pain points that aren't obvious from
generic async tutorials.

Authored from first-party docs (`docs.python.org/3/library/asyncio.html`,
`nodejs.org/en/learn/asynchronous-work/event-loop-timers-and-nexttick`)
and common bot bug patterns.

## When to Use

- You hit a `CancelledError` in a long-running handler and the LLM
  call wasn't actually cancelled
- Sync function blocks the loop (chat bot stops responding for 4s)
- Race condition: two updates arrive at nearly the same time and
  the FSM gets corrupted
- Graceful shutdown leaks open HTTP connections or DB transactions
- "Why is my retry loop running 100x and not 3 times?"

## When NOT to Use

- Single-threaded script bot that runs once (no concurrency needed)
- High-throughput custom HTTP / RPC bot (different concurrency model)
- Embedded / MCU bot (this is desktop / server)

## Routing Table

| User intent                                      | Load                                            |
|--------------------------------------------------|-------------------------------------------------|
| "asyncio tasks / gather / shield in Python bot"  | `references/python-asyncio.md`                  |
| "Node.js event loop / microtasks / promises"     | `references/nodejs-eventloop.md`                |
| "Graceful shutdown of bot process"               | `references/graceful-shutdown.md`               |
| "Cancellation, timeouts, asyncio.shield"         | `references/cancellation-and-timeouts.md`       |
| "Error propagation across handler chains"        | `references/error-propagation-in-bots.md`       |

## Universal async-runtime rules

1. **Never block the loop.** Sync calls (DB, requests.get, file
   read) halt ALL other handlers. Use `asyncio.to_thread` (Python)
   or worker threads / async APIs (Node).
2. **Cancellation is cooperative, not preemptive.** A `Task.cancel()`
   posts a `CancelledError` at the next `await` — only if your code
   awaits there. Block inside a sync helper, and cancellation
   never fires.
3. **Streams need explicit cleanup.** Open HTTP connections for
   SSE streams leak if your handler exits without cancelling the
   stream task.
4. **Race conditions are everywhere.** Two updates for the same
   user arriving in parallel — the FSM context needs locking or
   queueing.
5. **`finally` and `try/except` for cleanup.** Many bots forget
   typing-indicator cancellation, background polling tasks, etc.

## Common Pitfalls

1. **`asyncio.run()` in handler.** `asyncio.run` creates a new loop;
   you can't nest. Use `await` in existing handlers.
2. **`time.sleep` instead of `await asyncio.sleep`.** Blocks the loop.
3. **`asyncio.gather(*tasks)` without `return_exceptions=True`.** A
   single failure cancels the gather entirely; you may want to
   tolerate partial failures.
4. **Unhandled `CancelledError`.** In Python 3.8+, CancelledError is
   a BaseException. Code that catches `Exception` only WON'T catch
   cancellation — verify intentionality.
5. **`Promise.all` without `Promise.allSettled`.** One rejection kills
   all. Same pitfall, different API.
6. **Background tasks never cancelled.** Create_task for typing-
   indicator on every message; never cancel them on completion →
   memory / connection leak.
7. **`event.preventDefault()` not called in handler.** Wrong handler
   fires concurrently and you double-respond.

## Verification Checklist

- [ ] No calls to `time.sleep` / `requests.get` / `open()` without
      async wrapper
- [ ] Every `asyncio.create_task` has a corresponding cancel + wait
- [ ] `try/finally` around every "open" outside the bot's known
      resources (DB conn, HTTP stream)
- [ ] Tested with `kill -SIGTERM` mid-stream — graceful shutdown
      takes < 5s
- [ ] Two-parallel-request stress test passes (race-free FSM)

## Related Skills

- `telegram-bot-dev` — Python async bot (aiogram)
- `discord-bot-dev` — Node/TS bot (discord.js, grammy)
- `llm-streaming-reply` — streaming async-patterns

## Sources cited inline

- Python asyncio docs — https://docs.python.org/3/library/asyncio.html
- Node.js event loop — https://nodejs.org/en/learn/asynchronous-work/event-loop-timers-and-nexttick
- PEP 654 — https://peps.python.org/pep-0654/ (Exception Groups)
- Node.js async best practices — https://nodejs.org/en/learn/asynchronous-work/overview-of-blocking-vs-non-blocking
