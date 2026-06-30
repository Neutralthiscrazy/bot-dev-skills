---
name: python-telebot
description: >-
  Companion digest for pyTelegramBotAPI / python-telebot. Sync API,
  decorator-style handlers, when to use it (small simple bots) and when
  NOT to use it (anything FSM-heavy or high-scale). License is GPL —
  flag for commercial work.
version: 0.1.0
user-invocable: false
author: neutralthiscrazy
license: MIT
---

# python-telebot (pyTelegramBotAPI) — small simple sync bots

## When to load

Only if:
- Your bot has fewer than ~10 commands
- It has no real FSM (no conversation flows)
- You're fine with synchronous request/response
- You want minimum dependencies (just `pytelegrambotapi` + maybe requests)

Don't use if: long conversations, async I/O, production-scale traffic,
state machines. Reach for aiogram 3.x or grammy.

## Install + minimal bot

```bash
pip install pyTelegramBotAPI
```

```python
import telebot

bot = telebot.TeleBot(token="...")  # from env, not hard-coded

@bot.message_handler(commands=["start"])
def on_start(message):
    bot.send_message(message.chat.id, "hello, " + str(message.from_user.first_name))

bot.infinity_polling()
```

## Common patterns

**Inline keyboard:**

```python
from telebot.types import InlineKeyboardMarkup, InlineKeyboardButton

kb = InlineKeyboardMarkup()
kb.add(InlineKeyboardButton(text="yes", callback_data="vote_yes"),
       InlineKeyboardButton(text="no",  callback_data="vote_no"))

@bot.message_handler(commands=["vote"])
def vote(message):
    bot.send_message(message.chat.id, "vote?", reply_markup=kb)

@bot.callback_query_handler(func=lambda c: c.data.startswith("vote_"))
def handle_vote(call):
    answer = "yes" if call.data == "vote_yes" else "no"
    bot.answer_callback_query(call.id, text=f"vote: {answer}")
```

**Stateful flow without FSM (manual dict):**

```python
PENDING = {}  # user_id -> next expected input

@bot.message_handler(commands=["feedback"])
def feedback_start(message):
    PENDING[message.from_user.id] = "name"
    bot.send_message(message.chat.id, "name?")

@bot.message_handler(func=lambda m: m.from_user.id in PENDING)
def capture(message):
    state = PENDING[message.from_user.id]
    if state == "name":
        PENDING[message.from_user.id] = "email"
        bot.send_message(message.chat.id, "email?")
```

Works at toy scale, breaks under concurrent users.

## Gotchas

- **Sync API blocks the polling loop.** Any slow handler stalls ALL
  updates. Wrap with `threading.Thread` if you must, but error
  handling gets messy.
- **No built-in storage.** Restart = full state loss.
- **License is GPL-3.0.** Linking python-telebot via pip may make
  your project subject to GPL. Commercial projects: consult counsel
  or pick aiogram (BSD-3) or grammy (MIT) instead.
- Canonical import name: `pyTelegramBotAPI` (lower-case 'p' on PyPI).

## Sources

- GitHub — https://github.com/eternnoir/pyTelegramBotAPI
- PyPI — https://pypi.org/project/pyTelegramBotAPI/
- License — GPL-3.0 (commercial caution)
