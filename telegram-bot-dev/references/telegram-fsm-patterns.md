---
name: telegram-fsm-patterns
description: >-
  Companion digest for Finite State Machine design in Telegram bots
  (aiogram FSMContext, grammy conversations). When to model states,
  how to handle back / cancel / timeout, state storage backends,
  pitfalls specific to chat-driven FSMs.
version: 0.1.0
user-invocable: false
author: neutralthiscrazy
license: MIT
---

# FSM design patterns for Telegram bots

Chat bots are inherently stateful — the user issues a partial input,
the bot replies, the user supplies more, etc. The naive approach
(`@router.message(F.text)` and a global buffer) fragments immediately
under real use. The disciplined approach is to model the dialogue as
a State Machine, with explicit transitions.

## When to load

- Multi-step flows (forms, wizards, dialogs)
- Need back / cancel / restart without losing data
- Multiple users in different states simultaneously (always)

## When NOT to load

- Stateless bots (commands → reply → done). Skip FSM.

## Standard states

| State | Meaning |
|-------|---------|
| **Idle** | No flow active; respond to global commands |
| **Awaiting input** | Bot questions, expects reply |
| **Awaiting choice** | Bot posted choices inline, expects callback_query |
| **Awaiting confirmation** | Destructive action queued, expects go/no-go |
| **Completing** | Doing the work, possibly long |
| **Done** | Finished; transition back to Idle |

## aiogram FSMContext cheat sheet

```python
from aiogram.fsm.context import FSMContext
from aiogram.fsm.state import State, StatesGroup

class SignupFlow(StatesGroup):
    name = State()
    email = State()
    confirm = State()

@router.message(Command("signup"))
async def start(message: Message, state: FSMContext):
    await state.set_state(SignupFlow.name)
    await message.answer("name?")

@router.message(SignupFlow.name)
async def take_name(message: Message, state: FSMContext):
    if not message.text.strip():
        await message.answer("name can't be empty, try again")
        return
    await state.update_data(name=message.text.strip())
    await state.set_state(SignupFlow.email)

@router.message(Command("cancel"), StateFilter("*"))
async def cancel(message: Message, state: FSMContext):
    await state.clear()
    await message.answer("cancelled")
```

### Storage backends

| Backend | When | Tradeoff |
|---------|------|----------|
| `MemoryStorage` | local dev | Lost on restart |
| `RedisStorage` | production, multi-instance | Fast, ephemeral |
| `MongoStorage` | MongoDB-heavy stack | One less moving part |
| `DynamoDBStorage` | AWS-native | Pay-as-you-go, harder locally |

Configure at Dispatcher: `Dispatcher(storage=RedisStorage(...))`.

## grammy conversations plugin

```typescript
async function signup(conversation: any, ctx: MyContext) {
  await ctx.reply("name?");
  const nameCtx = await conversation.waitFor("message:text");
  await ctx.reply(`Hi ${nameCtx.msg.text}. email?`);
  const eCtx = await conversation.waitFor("message:text");
  await ctx.reply(`Send to ${eCtx.msg.text}?`);
  await conversation.waitFor("callback_query:data");
  await ctx.reply("Done.");
}

bot.use(createConversation(signup));
bot.command("signup", (ctx) => ctx.conversation.enter("signup"));
```

The plugin handles state tracking + resume automatically. Long forms
need `idleTimeout` per conversation.

## Cross-cutting FSM design rules

1. **Always have a global `/cancel`.** Users WILL need to bail out. Make
   it `Command("cancel", state="*")` and `state.clear()`.
2. **Validation stays in the destination state.** Don't accumulate
   half-valid input. If `email` gets "abc", reply "needs an email,
   try again" without advancing.
3. **Backwards transitions are explicit.** Don't infer "user wants
   back" from arbitrary text. Provide an "← Back" inline button or
   `/back` command.
4. **Persistent state is a security boundary.** State may include PII
   (email, phone). Redis with auth, MongoDB with TLS.
5. **Multi-modal flows** (typing vs tapping) need both handlers —
   `MessageHandler(F.text)` AND `CallbackQueryHandler(F.data == "...")`
   on the same state.
6. Don't share state across users — FSM context is keyed on
   `chat_id + user_id` automatically.

## Pitfalls

- Clear state on every error path. Otherwise a failed validation
  hangs the user in the wrong state, and they have to `/cancel`.
- Don't store large blobs in state data. Keep IDs / references;
  load on demand. State machines are bookkeeping, not databases.
- Many parallel flows? Multiple `StatesGroup`s, not one big graph.
- `state.set_state(SignupFlow.name)` — typo of the class name doesn't
  error at runtime. Catch with `mypy` / strict TS.

## Sources

- aiogram FSM docs — https://aiogram.dev/
- grammy Conversations — https://grammy.dev/plugins/conversations
- grammy Sessions — https://grammy.dev/plugins/sessions
