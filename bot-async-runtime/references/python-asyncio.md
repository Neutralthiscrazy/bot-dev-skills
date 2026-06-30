---
name: python-asyncio
description: >-
  Companion digest for Python asyncio in a chat-bot context. Tasks,
  gather, shield, to_thread, async context managers, exception
  groups, and bot-specific patterns (debouncing typing indicators,
  concurrent handler safety, race-free FSM).
version: 0.1.0
user-invocable: false
author: neutralthiscrazy
license: MIT
---

# Python asyncio — bot-side patterns

## When to load

- Using aiogram / discord.py / plain aiohttp and need to compose
  async tasks
- Hit a race condition where two updates hit the same FSM state
- Bot silently hangs on a sync DB call

## Task primitives

```python
import asyncio

async def do_work():
    await asyncio.sleep(1)  # yields, doesn't block

task = asyncio.create_task(do_work())
result = await task        # wait for it

task.cancel()              # cancel
try:
    await task
except asyncio.CancelledError:
    pass                  # cleanup here
```

`asyncio.create_task` does NOT await the coroutine; it schedules
it. The underlying coroutine starts running in the same loop.
Always await eventually to observe results / exceptions.

## gather — wait for many

```python
results = await asyncio.gather(*tasks, return_exceptions=True)
```

`return_exceptions=True` makes gather return exception objects
instead of raising — useful when partial results are fine.
`return_exceptions=False` (default) — first exception cancels the
others. Use explicit when you want either behavior.

## shield — protect from cancellation

```python
result = await asyncio.shield(some_coro, timeout=10)
```

If the outer task is cancelled, the inner coro keeps running.
Useful for long LLM calls that shouldn't be cancelled when the
update handler is cancelled.

```python
async def handle_stream_question(message, llm_stream):
    try:
        async for delta in llm_stream:
            await message.answer(delta)
    except asyncio.CancelledError:
        # User /cancel - finish streaming for billing accuracy
        try:
            async for _ in llm_stream:
                pass
        finally:
            raise
```

If you cannot exhaust the stream in the finally block (e.g. an
async generator), call `.aclose()`.

## to_thread — sync to async

```python
async def fetch_user(user_id):
    return await asyncio.to_thread(sync_user_lookup, user_id)
```

`to_thread` runs the sync function in the default thread pool
(size 32). Use for SHORT sync calls. Long ones block threads
and the pool eventually exhausts.

## Async context managers (bot state)

```python
from contextlib import asynccontextmanager

@asynccontextmanager
async def typing_indicator(bot, chat_id):
    task = asyncio.create_task(_typing_loop(bot, chat_id))
    try:
        yield
    finally:
        task.cancel()
        try:
            await task
        except asyncio.CancelledError:
            pass

async def _typing_loop(bot, chat_id):
    while True:
        try:
            await bot.send_chat_action(chat_id, action="typing")
            await asyncio.sleep(4)
        except asyncio.CancelledError:
            break

# Usage
async with typing_indicator(bot, chat_id):
    await long_running_thing()
```

Enters context: starts typing. Exits context: cancels typing
loop, awaits cancellation. This is the cleanest pattern for
typing indicators in Python bots.

## Race conditions and FSMs

If a single user issues two updates nearly simultaneously:

- Both pass through aiogram's handler-dispatch
- Both call `state.set_state(...)` and `state.update_data(...)`
- Last write wins, but intermediate state is corrupted

Mitigation: wrap FSM mutations in an asyncio.Lock per user_id:

```python
import asyncio
from collections import defaultdict

user_locks = defaultdict(asyncio.Lock)

async def handle_update(message, state):
    async with user_locks[message.from_user.id]:
        # FSM mutation inside lock
        await state.set_state(FeedbackForm.name)
        await message.answer("name?")
```

Trade-off: lock contention could slow handlers for popular users,
but correctness > speed for bot FSMs.

## Exception groups (Python 3.11+)

```python
try:
    async with asyncio.TaskGroup() as tg:
        tg.create_task(work1())
        tg.create_task(work2())
except* ValueError as eg:
    # catch all ValueErrors from the group
    ...
```

`TaskGroup` from Python 3.11 wraps gather. If any task fails, all
are cancelled, and an ExceptionGroup is raised. Useful for
parallel-call-with-cleanup patterns in bot handlers.

## Common pitfalls

- `await sync_call()` where sync_call does network — entire bot
  stops until it returns.
- `asyncio.run(...)` inside a running loop — `RuntimeError: asyncio.run()
  cannot be called from a running event loop`. Use `await` directly.
- `for ... in coroutine` instead of `async for ... in async_iter`:
  iterates over coroutine object's internal attrs, not its values.
- `asyncio.Lock()` shared across handler threads — locks are loop-
  bound. Use per-loop.

## Sources

- Python asyncio — https://docs.python.org/3/library/asyncio.html
- asyncio.TaskGroup — https://docs.python.org/3/library/asyncio-task.html#asyncio.TaskGroup
- PEP 654 (Exception Groups) — https://peps.python.org/pep-0654/
- aiogram FSM storage — https://aiogram.dev/
