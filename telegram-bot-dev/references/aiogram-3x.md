---
name: aiogram-3x
description: >-
  Companion digest for aiogram 3.x. Setup, Bot/Dispatcher/Router pattern,
  FSMContext + StatesGroup for stateful conversations, middleware, inline /
  reply keyboards, webhook and polling starts, and the v3-specific gotchas
  that don't appear in v2 tutorials. Pair with telegram-bot-dev/SKILL.md
  routing table.
version: 0.1.0
user-invocable: false
author: neutralthiscrazy
license: MIT
---

# aiogram 3.x — Python async Telegram bot framework

## When to load

You're building Telegram bot in Python, need async-friendly stack, runtime
under 10k RPS, and want either FSM for conversation flows or middleware
for cross-cutting concerns (auth, logging, rate limiting).

If your project is small-sync-no-state, use `python-telebot.md` instead.

## Install + minimal bot

```bash
pip install aiogram==3.*
```

Minimal bot structure (runnable as `python main.py`):

```python
import asyncio
from aiogram import Bot, Dispatcher, Router, F
from aiogram.filters import CommandStart
from aiogram.types import Message

BOT_TOKEN = "..."  # load from env, NEVER hard-code

bot = Bot(token=BOT_TOKEN)
dp = Dispatcher()
router = Router()
dp.include_router(router)

@router.message(CommandStart())
async def on_start(message: Message):
    await message.answer(f"hello, {message.from_user.first_name}")

async def main():
    # polling mode (default)
    await dp.start_polling(bot)

if __name__ == "__main__":
    asyncio.run(main())
```

See https://aiogram.dev/ for the full quick-start.

## Router pattern (the v3 thing v2 tutorials get wrong)

v2 used `@dp.message_handler(...)`. v3 splits handlers into routers:

```python
# admin_handlers.py
admin_router = Router()

@admin_router.message(F.text == "/ban")
async def ban(message: Message): ...

# user_handlers.py
user_router = Router()

@router.message(F.text.startswith("/"))
async def unknown_command(message: Message): ...

# main.py
dp.include_router(admin_router)
dp.include_router(user_router)
```

`include_router` is the v3 replacement for v2's decorator on dp directly.
Routers that are never included are silently dead — see Pitfall #2 in
the parent SKILL.md.

## FSMContext for conversation flows

```python
from aiogram.fsm.context import FSMContext
from aiogram.fsm.state import State, StatesGroup

class FeedbackForm(StatesGroup):
    name = State()
    email = State()
    confirm = State()

@user_router.message(Command("feedback"))
async def start_form(message: Message, state: FSMContext):
    await state.set_state(FeedbackForm.name)
    await message.answer("What's your name?")

@user_router.message(FeedbackForm.name)
async def take_name(message: Message, state: FSMContext):
    await state.update_data(name=message.text)
    await state.set_state(FeedbackForm.email)
    await message.answer("And your email?")
```

For full FSM design patterns (state split, back/cancel, in-state
validation) — see `telegram-fsm-patterns.md`.

## Keyboards: inline vs reply

`ReplyKeyboardMarkup` REPLACES the user's keyboard with buttons. Replies
to these come back as regular text messages on the handler side.

`InlineKeyboardMarkup` ATTACHES buttons to a specific message. Taps come
back as `callback_query` updates, handled with
`@router.callback_query(F.data == "...")`.

```python
from aiogram.types import InlineKeyboardMarkup, InlineKeyboardButton
from aiogram.types import ReplyKeyboardMarkup, KeyboardButton

inline_kb = InlineKeyboardMarkup(inline_keyboard=[
    [InlineKeyboardButton(text="Yes", callback_data="vote_yes")],
    [InlineKeyboardButton(text="No",  callback_data="vote_no")]
])

reply_kb = ReplyKeyboardMarkup(keyboard=[
    [KeyboardButton(text="Menu")],
    [KeyboardButton(text="Help"), KeyboardButton(text="About")]
], resize_keyboard=True)

await message.answer("Pick:", reply_markup=inline_kb)
```

## Middleware

```python
from aiogram import BaseMiddleware
from typing import Callable, Dict, Any, Awaitable

class AuthMiddleware(BaseMiddleware):
    async def __call__(
        self,
        handler: Callable[[Message, Dict[str, Any]], Awaitable[Any]],
        event: Message,
        data: Dict[str, Any],
    ) -> Any:
        if event.from_user.id not in ALLOWED_IDS:
            await event.answer("nope")
            return
        return await handler(event, data)

dp.message.outer_middleware(AuthMiddleware())
```

Outer middleware runs BEFORE router dispatch — right place for logging,
auth, metrics.

## Polling vs webhook (in code)

```python
# polling
await dp.start_polling(bot, allowed_updates=dp.resolve_used_update_types())

# webhook aiohttp
from aiogram.webhook.aiohttp_server import SimpleRequestHandler, setup_application
from aiohttp import web

async def on_startup(bot: Bot) -> None:
    await bot.set_webhook(f"{BASE_URL}/webhook", drop_pending_updates=True)

async def on_shutdown(bot: Bot) -> None:
    await bot.delete_webhook()

dp.startup.register(on_startup)
dp.shutdown.register(on_shutdown)

app = web.Application()
SimpleRequestHandler(dispatcher=dp, bot=bot).register(app, path="/webhook")
setup_application(app, dp, bot=bot)
web.run_app(app, host="0.0.0.0", port=8080)
```

## Common gotchas

- `Message.answer()` accepts `parse_mode` ("HTML", "MarkdownV2"),
  but MarkdownV2 with special chars in user input errors. Prefer
  `parse_mode="HTML"` or escape with `aiogram.utils.text_decorations`.
- `bot.download_file()` for media — don't `requests.get(file_id)`,
  use the official `getFile` flow + the resulting path.
- `dp.resolve_used_update_types()` — pass to `allowed_updates` so
  dispatcher only requests event types you actually handle.

## Sources

- aiogram — https://aiogram.dev/ (canonical)
- GitHub — https://github.com/aiogram/aiogram
- Telegram Bot API — https://core.telegram.org/bots/api
- Community chat — https://t.me/aiogram_en (EN) / https://t.me/aiogram_ru (RU)
