---
name: error-propagation-in-bots
description: >-
  Companion digest for error propagation in chat-bot handlers.
  When to log-and-continue vs raise-and-stop, mid-stream error
  handling, surface-to-user messages, and the "global error trap"
  pattern (bot.catch in grammy, Bot.catch_error in discord.py).
version: 0.1.0
user-invocable: false
author: neutralthiscrazy
license: MIT
---

# Error propagation in chat bot handlers

## When to load

- Bot "freezes" — error swallowed somewhere, handler never
  continues
- User sees weird output — handler caught expected error but new
  error type crashes the bot
- Designing error contract for a multi-source / multi-agent bot

## Three-tier model

| Tier | Where              | Action                                  |
|------|--------------------|-----------------------------------------|
| Validation error (user input) | Parse / FSM | Reply to user, stay in current state |
| Recoverable (LLM 429, DB timeout) | Inside handler | Retry or fallback |
| Programming error (TypeError, AttributeError) | Anywhere | Log, surface "sorry I broke" to user |

Programming errors should never silently pass. Always surface.

## Python: `try/except` per handler

```python
@router.message(Command("do_thing"))
async def do_thing(message: Message):
    try:
        result = await compute(message.text)
        await message.answer(f"Result: {result}")
    except ValidationError:
        await message.answer("That input didn't parse. Try again.")
    except LLMRetryableError:
        await message.answer("I had trouble reaching the model. Try again in a moment.")
    except Exception:
        logging.exception("do_thing failed")
        await message.answer("Sorry, I hit a bug. The team has been notified.")
```

Always include an `Exception` (or `BaseException`) catch-all in
production. The user-facing message can be the same; the actual
exception is logged.

## discord.py: global error handler

```python
@bot.event
async def on_application_command_error(ctx, error):
    # Global slash-command error trap
    if isinstance(error, commands.CommandOnCooldown):
        await ctx.respond("Slow down.", ephemeral=True)
    elif isinstance(error, commands.MissingPermissions):
        await ctx.respond("You don't have permission.", ephemeral=True)
    else:
        logging.exception("Slash command error")
        await ctx.respond("Sorry, I hit a bug.", ephemeral=True)
```

For prefix-commands (`!ping`), `on_command_error` instead.

## grammy: `bot.catch`

```javascript
bot.catch((err) => {
  const ctx = err.ctx;
  console.error(`Error in update from ${ctx.update.update_id}:`, err.error);
  if (ctx.reply) {
    return ctx.reply("Sorry, I hit a bug.").catch(() => {});
  }
});
```

`bot.catch` is the global error trap. Always install one. Without
it, exceptions in handlers die silently (visible only in console).

## Mid-stream error (LLM streaming)

Different concern: the LLM stream emitted N tokens, then error.

```python
async def stream_with_recovery(message, llm_stream, edit_fn):
    buffer = ""
    try:
        async for delta in llm_stream:
            buffer += delta
            await maybe_flush(buffer, edit_fn)
    except (httpx.RequestError, LLMAPIError) as e:
        await edit_fn(buffer + "\n\n_⚠️ partial answer — connection issue_")
        logging.warning(f"LLM stream cut: {e!r}")
    except asyncio.TimeoutError:
        await edit_fn(buffer + "\n\n_⚠️ partial answer — timed out_")
```

User sees partial output, marker appended. Not perfect UX but
visible and recoverable.

## Common pitfalls

- Catching `Exception` but logging only `e.__class__` — harder
  to debug. Use `logging.exception` or `traceback.format_exc()`.
- Sending "Sorry" message without a fallback — if the chat reply
  itself fails, user sees nothing.
- Handler errors not propagating to global trap — if you catch
  `Exception` and don't re-raise, the global trap is bypassed.
  Re-raise + log.
- `try/except` that catches the platform error class with a bare
  `except Exception` — over-catches. Use specific subclasses.

## User-visible error message design

Don't expose: SQL schemas, stack traces, library versions, internal
class names. Do surface: "what happened" (network / model / bug),
"what to do next" (try again, contact admin).

Examples:

- "I had trouble reaching the model. Try again in a moment." (LLM 5xx)
- "I couldn't find that. Check the spelling?" (not-found)
- "Sorry, I'll need a sec — rephrase the question." (timeout)
- "Sorry, I hit a bug. The team has been notified." (unexpected)

## Sources

- grammy error handling — https://grammy.dev/guide/errors
- discord.py errors — https://discordpy.readthedocs.io/en/stable/ext/commands/commands.html#exception-handling
- aiogram error handling — https://aiogram.dev/
- Python logging — https://docs.python.org/3/library/logging.html
