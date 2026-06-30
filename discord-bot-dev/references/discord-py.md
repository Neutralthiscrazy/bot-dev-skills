---
name: discord-py
description: >-
  Companion digest for discord.py 2.x (Python async). Setup, Bot vs
  AutoShardedBot, cog pattern for modular commands, discord.app_commands
  tree for slash commands, and the asyncio-context gotchas that bite
  Python beginners.
version: 0.1.0
user-invocable: false
author: neutralthiscrazy
license: MIT
---

# discord.py 2.x — Python async Discord bot

## When to load

- Bot in Python (not JS/TS)
- Want a class-based, cogs-as-modules layout
- Asyncio-friendly stack (FastAPI server, aiohttp calls, etc.)
- Mature ecosystem (Jishaku, aiohttp, asyncpg all have known-good
  integration)

If your project is Node/TS-flavored, use `discordjs-v14.md` instead.

## Install + minimal bot

```bash
pip install discord.py[voice]   # or just `discord.py` if no voice
```

Bot `bot.py`:

```python
import discord
import os

intents = discord.Intents.default()
intents.message_content = True  # if reading message text

bot = discord.Bot(intents=intents)  # discord.py 2.x uses discord.Bot not discord.Client for slash

@bot.event
async def on_ready():
    print(f"Logged in as {bot.user}")

@bot.slash_command(description="Replies with Pong!")
async def ping(ctx: discord.ApplicationContext):
    await ctx.respond("Pong!")

bot.run(os.environ["DISCORD_TOKEN"])
```

`discord.Bot` is the 2.x entry-point with slash-command tree.
`discord.Client` still works for gateway-only bots (rare).

See https://discordpy.readthedocs.io/ for full quick-start.

## Cogs — modular commands

Cogs are files that group commands + listeners + checks. Standard
folder layout:

```
my_bot/
├── bot.py          # entry, loads cogs
├── cogs/
│   ├── general.py
│   ├── admin.py
│   └── fun.py
```

`cogs/general.py`:

```python
import discord
from discord.ext import commands

class General(commands.Cog):
    def __init__(self, bot):
        self.bot = bot

    @commands.slash_command(description="Echo what you say")
    async def echo(self, ctx: discord.ApplicationContext, text: str):
        await ctx.respond(text)

def setup(bot):
    bot.add_cog(General(bot))
```

Load in `bot.py`:

```python
for filename in os.listdir("./cogs"):
    if filename.endswith(".py"):
        bot.load_extension(f"cogs.{filename[:-3]}")
```

Slash-command tree is maintained per `discord.Bot` — adding cogs
auto-registers with the global slash tree. For guild-scoped
(development), use `discord.Bot(debug_guilds=[DEV_GUILD_ID])`.

## Slash-command tree (vs interaction listening)

discord.py 2.x's slash `tree`:

```python
@bot.slash_command(name="ban", description="Ban a user")
@commands.has_permissions(ban_members=True)
async def ban(
    ctx: discord.ApplicationContext,
    member: discord.Member,
    reason: str = "no reason",
):
    await member.ban(reason=reason)
    await ctx.respond(f"Banned {member}.")

# subcommand group
@bot.slash_command(name="settings")
@commands.has_permissions(administrator=True)
async def settings(ctx): ...

@settings.sub_command(name="get", description="Show setting")
async def settings_get(ctx, key: str): ...
```

vs manual interaction listening (lower-level — usually not needed
unless you want full control over component lifecycle).

## Async gotchas (Python-specific)

- `time.sleep(5)` blocks the entire bot. Use `await asyncio.sleep(5)`.
- Sync libraries (e.g. `requests`) can stall. Wrap in
  `asyncio.to_thread(requests.get, url)` or use `aiohttp`.
- DB calls: SQLAlchemy sync can stall — use
  `sqlalchemy.ext.asyncio` (AsyncSession).
- Rate-limited HTTP calls without backoff will hammer the Discord API
  and get you globally banned.
- `ctx.respond()` and `ctx.send()` are different — `respond` is the
  interaction reply (3-second deadline); `send` is just a regular
  channel.send. Use `ctx.respond()` for slash.

## Common gotchas

- Forgot to enable `MESSAGE_CONTENT` intent in dev portal AND in code
  (with `intents.message_content = True`). Without it, `on_message`
  fires but `msg.content` is empty.
- Cogs fail to load silently if you typo'd class name — Discord.py
  logs it but the listener registration is dead. Always check
  console output on startup.
- `commands.Bot.command()` is for prefix commands. With
  `discord.Bot`, prefix commands don't auto-load — you need
  `bot.add_command(...)` if you want both slash + prefix.
- Bot doesn't show up as online in the client — usually means a
  privileged intent is requested but not toggled in the dev portal.
- Async generator contexts (e.g. `async with`) inside cogs need
  proper teardown — `cog_unload` hook.

## Sources

- discord.py docs — https://discordpy.readthedocs.io/
- GitHub — https://github.com/Rapptz/discord.py
- Migration guide 1.x → 2.x — https://discordpy.readthedocs.io/en/stable/migrating.html
