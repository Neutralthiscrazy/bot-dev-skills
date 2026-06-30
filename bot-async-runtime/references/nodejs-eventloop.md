---
name: nodejs-eventloop
description: >-
  Companion digest for Node.js event loop in a chat-bot context.
  Microtask vs macrotask, AbortController for stream cancellation,
  Promise.all vs Promise.allSettled, blocking the loop with sync
  code, and worker threads (Node 10.5+) for CPU-bound work.
version: 0.1.0
user-invocable: false
author: neutralthiscrazy
license: MIT
---

# Node.js event loop — bot-side patterns

## When to load

- Discord.js / grammy bot hit by an unresponsive event
- Long sync call blocks the loop and other updates pile up
- Stream cancellation isn't firing — bot keeps writing after
  handler returned
- Race condition between two concurrent interactions

## The event loop in 30 seconds

Node has ONE thread. Async I/O runs out-of-band in libuv; the
thread processes JavaScript synchronously between I/O completions.
Promise `.then`, `await`, and microtasks run BEFORE the next I/O
phase. Anything that runs synchronously on the thread blocks all
callbacks and pending promise resolutions.

```javascript
// This blocks the loop:
function sync_blocking() {
  for (let i = 0; i < 1e9; i++) { ... }
}

// This is fine:
function async_ok() {
  return new Promise(resolve => {
    setImmediate(resolve);
  });
}
```

## Promise.all vs Promise.allSettled

```javascript
// One rejection kills the whole thing:
const results = await Promise.all([task1(), task2(), task3()]);
// task2 throws → task1 and task3 cancelled, await throws.

// Tolerate partial failures:
const results = await Promise.allSettled([task1(), task2(), task3()]);
// [{status: "fulfilled", value: ...}, {status: "rejected", reason: ...}, ...]
```

For "send N notifications, ignore individual failures" pattern,
`allSettled` is the right call.

## AbortController — proper cancellation

Modern Node has `AbortController` for cancelling in-flight async
operations:

```javascript
async function streamLlmReply(prompt, signal) {
  const response = await fetch("https://api.openai.com/...", {
    method: "POST",
    body: JSON.stringify({ ... }),
    signal,  // <— abort-aware
  });
  for await (const chunk of parseSSE(response.body)) {
    yield chunk;
    if (signal.aborted) break;  // manual check too
  }
}

// In handler:
const ac = new AbortController();
setTimeout(() => ac.abort(), 30000);  // 30s timeout
try {
  for await (const delta of streamLlmReply(prompt, ac.signal)) {
    await message.edit({ content: delta });
  }
} catch (e) {
  if (e.name === "AbortError") {
    await message.edit({ content: "Timed out." });
  }
}
```

## Async iterators — async for-of

Many things in Node's std lib / discord.js return AsyncIterables:

```javascript
for await (const event of client.events) {
  // process
}
```

`for await of` awaits each yield. Cancellation requires the
underlying iterator to support it. Most do.

## Worker threads (CPU-bound)

```javascript
// Need pure CPU work (parsing, image manipulation)? Worker thread.
import { Worker } from "worker_threads";

const w = new Worker("./cpu-task.js", {
  workerData: { input },
});

const result = await new Promise((resolve, reject) => {
  w.on("message", resolve);
  w.on("error", reject);
});
w.terminate();
```

For a chat bot, you shouldn't need worker threads unless you do
heavy parsing / hashing. Standard library I/O is async.

## Race conditions and FSM

In JS bots, FSMs are typically database-backed (Redis, Postgres).
Two parallel updates → two SQL updates race.

Mitigation: a `Map<user_id, Promise>` mutex:

```javascript
const userLocks = new Map();

async function withUserLock(userId, fn) {
  const prev = userLocks.get(userId) ?? Promise.resolve();
  let release;
  const next = new Promise(resolve => { release = resolve; });
  userLocks.set(userId, prev.then(() => next));
  await prev;
  try {
    return await fn();
  } finally {
    release();
    if (userLocks.get(userId) === next) userLocks.delete(userId);
  }
}

// Usage
await withUserLock(ctx.from.id, async () => {
  await fsm.setState(userId, "awaiting_email");
  await ctx.reply("email?");
});
```

## Common pitfalls

- `forEach` over async work — runs sync, doesn't actually await.
  Use `for...of` or `Promise.all`.
- `await` missing on `Promise` returns — the value is the Promise,
  not its resolved value. Type system doesn't always warn.
- Listener leak: registering `client.on("...", handler)` in a hot
  path — handlers accumulate, fire many times per single event.
  Use named handlers you can `.off`.
- `setTimeout(...)` not cleared on cleanup → fires after handler
  expected to finish, causing "ghost" replies.

## Sources

- Node.js event loop — https://nodejs.org/en/learn/asynchronous-work/event-loop-timers-and-nexttick
- Async iteration — https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Statements/for-await...of
- AbortController — https://developer.mozilla.org/en-US/docs/Web/API/AbortController
- Worker threads — https://nodejs.org/api/worker_threads.html
- discord.js events — https://discordjs.guide/creating-your-bot/event-handling.html
