---
name: graceful-shutdown
description: >-
  Companion digest for graceful shutdown of a chat bot. SIGTERM
  handling, draining in-flight handlers, closing HTTP streams, and
  the bot-specific pitfall of "user just sent /start, then process
  got SIGTERM, partial state corruption".
version: 0.1.0
user-invocable: false
author: neutralthiscrazy
license: MIT
---

# Graceful shutdown — bot process

## When to load

- Production bot needs to handle SIGTERM cleanly (Kubernetes pods,
  ECS tasks, systemd stop, Railway / Render deploy)
- Mid-update corruption on restart (FSM saved half-way,
  LLM stream truncated, message in DB but not finished)
- Want <2s graceful shutdown with up-to-N pending requests

## Universal signal handler

```python
import asyncio
import signal

class GracefulKiller:
    def __init__(self):
        self.shutdown = asyncio.Event()

    def install(self, loop):
        for sig in (signal.SIGTERM, signal.SIGINT):
            loop.add_signal_handler(sig, self.shutdown.set)

killer = GracefulKiller()
# later, in main: killer.install(asyncio.get_running_loop())

# In main loop:
while not killer.shutdown.is_set():
    # process, but checked between events
    ...
print("shutting down...")
```

```javascript
// Node.js
let isShuttingDown = false;

process.on("SIGTERM", () => {
  if (isShuttingDown) return;
  isShuttingDown = true;
  console.log("SIGTERM; draining...");
  client.destroy().then(() => process.exit(0));
});
```

In Kubernetes, set `terminationGracePeriodSeconds: 30` so the pod
has time to drain. SIGKILL follows.

## Draining in-flight handlers

Track active tasks / handler contexts, await on shutdown:

```python
active_tasks = set()

async def handle_update(update):
    task = asyncio.current_task()
    active_tasks.add(task)
    try:
        await dispatch(update)  # your normal handler
    finally:
        active_tasks.discard(task)

async def main():
    killer.install(asyncio.get_running_loop())
    ...
    await killer.shutdown.wait()
    # Now drain
    if active_tasks:
        await asyncio.gather(*active_tasks, return_exceptions=True)
    # Close bot, exit
    await bot.close()
```

## Stream cleanup

If your bot is mid-LLM-stream when SIGTERM arrives:

```python
async def stream_handler(message, llm_stream):
    try:
        async for delta in llm_stream:
            await message.edit(delta)
    except asyncio.CancelledError:
        # Drain LLM call so billing is correct
        try:
            async for _ in llm_stream:
                pass
        except Exception:
            pass  # lstream close error during shutdown
        raise
```

For Node with `AbortController`:

```javascript
process.on("SIGTERM", () => {
  controller.abort();  // stops all in-flight streams
  setTimeout(() => process.exit(0), 1000);
});
```

## Bot-specific pitfall: half-saved FSM

If SIGTERM hits mid-handler:

- FSM state was set to "awaiting input"
- DB transaction half-committed
- User sends another message after bot restarts → state corruption

Mitigation: wrap FSM mutation in DB transaction; either fully
applied or fully rolled back on shutdown.

```python
async def fsm_transition(state, new_state, data):
    async with state.fsm_storage.transaction() as txn:
        await txn.set_state(new_state)
        await txn.update_data(data)
        # commit at exit
```

If shutdown mid-way, transaction is auto-rolled back, state
unchanged. The next bot instance starts in the prior state
(which may itself be incomplete — but at least consistent).

## Order of operations on shutdown

1. Stop accepting new updates (close listeners or stop polling).
2. Cancel long-running background tasks (typing indicators, cron).
3. Drain active handlers (with timeout; 5-10s is reasonable).
4. Close HTTP / DB connections cleanly.
5. Flush logs.
6. Exit.

## Sources

- Kubernetes termination — https://kubernetes.io/docs/concepts/workloads/pods/pod-lifecycle/#pod-termination
- asyncio signals — https://docs.python.org/3/library/asyncio-eventloop.html#asyncio.loop.add_signal_handler
- Node process events — https://nodejs.org/api/process.html#process-events
- discord.js drain — https://discordjs.guide/additional-info/rare-tips.html#graceful-shutdown
- aiogram shutdown — https://aiogram.dev/
