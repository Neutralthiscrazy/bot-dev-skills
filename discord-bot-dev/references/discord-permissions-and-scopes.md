---
name: discord-permissions-and-scopes
description: >-
  Companion digest for Discord permission math and OAuth scopes. Bit
  flags, PermissionsBitField helpers, OAuth bot invite URL builder,
  scope checklist (bot, applications.commands, guilds.join for
  user-install), and the "works on my dev guild but fails publicly"
  permutation-mismatch catch.
version: 0.1.0
user-invocable: false
author: neutralthiscrazy
license: MIT
---

# Discord permissions, OAuth scopes, and invite URLs

## When to load

- You need a bot invite URL that actually works
- Slash commands didn't appear after invite
- You're checking permissions on a member / channel / role
- Building a role-selection UI
- Token / OAuth mismatches between dev/prod

## Permission bitset basics

Discord permissions are **powers of two** (single bits) combined into
a **bitstring**. For a member on a channel, the effective perms are
calculated as: `@everyone role + other roles on the user + channel
overrides`.

Common bits (remember: values, not flags as labels):

| Permission            | Bit    | Flag (discord.js v14)         |
|-----------------------|--------|-------------------------------|
| Send Messages         | 1<<11  | `PermissionFlagsBits.SendMessages` |
| Manage Messages       | 1<<13  | `PermissionFlagsBits.ManageMessages` |
| Admin                 | 1<<3   | `PermissionFlagsBits.Administrator` |
| Kick Members          | 1<<1   | `PermissionFlagsBits.KickMembers` |
| Ban Members           | 1<<2   | `PermissionFlagsBits.BanMembers` |
| Manage Roles          | 1<<28  | `PermissionFlagsBits.ManageRoles` |
| Connect (voice)       | 1<<20  | `PermissionFlagsBits.Connect` |

Use the framework's helper, **never hand-roll**:

```javascript
// discord.js
const perms = new PermissionsBitField([
  PermissionFlagsBits.SendMessages,
  PermissionFlagsBits.ManageMessages,
]).bitfield;  // bigint string
```

```python
# discord.py
perms = discord.Permissions(
  send_messages=True,
  manage_messages=True
).value  # int
```

Hand-rolling bit-shifts works but is bug-prone across Discord API
versions. The helpers update when new bits are added.

## OAuth scopes (bot invite URL)

The bot invite URL has the structure:

```
https://discord.com/oauth2/authorize?client_id={APP_ID}&scope={SCOPES}&permissions={PERMS_INT}
```

| Scope                  | Use                                       |
|------------------------|-------------------------------------------|
| `bot`                  | Adds a bot to a guild (default)           |
| `applications.commands`| **Required** for slash commands to work   |
| `guilds.join`          | Allows user-install bot to auto-join server |
| `applications.builds.*`| Manage activity / app builds (rare)       |

**Minimum working invite URL** for a bot with slash commands:

```
https://discord.com/oauth2/authorize?client_id=12345&scope=bot+applications.commands&permissions=274881076736
```

The permissions integer encodes which channel-level perms the bot
gets on every channel. The dev portal generates this URL for you
when you tick "Guild Install" + check perms — **prefer the portal
version** because picking the wrong permission bitmap lays the entire
permission system apples-to-oranges.

## Privileged gateway intents

Some intents require explicit toggle in the dev portal:

- `MESSAGE_CONTENT` — read msg.text / msg.content (also requires perms)
- `PRESENCE` (presence_update, online/offline)
- `SERVER_MEMBERS` (member join/leave/update)

For each, must be (a) toggled on in dev portal, (b) embedded in
client intents array:

```javascript
const client = new Client({
  intents: [
    GatewayIntentBits.Guilds,
    GatewayIntentBits.MessageContent,  // privileged
    GatewayIntentBits.GuildMembers,     // privileged
  ],
});
```

Without the portal toggle, the client connects but events for the
privileged scope never fire. Discord returns `DisallowedIntents`.

## Slash-command permission overrides

Server admins can forbid specific users from running your slash
command, or require a role. The framework API:

```javascript
// discord.js
new SlashCommandBuilder()
  .setDefaultMemberPermissions(PermissionFlagsBits.ManageMessages)
  .setName("ban")
  // ...
```

`setDefaultMemberPermissions(0)` makes the command require admin (or
specific role) by default.

```python
# discord.py
@bot.slash_command(name="ban", default_member_permissions=discord.Permissions(ban_members=True))
async def ban(ctx, member: discord.Member, reason: str = "no reason"):
    ...
```

## Common pitfalls

- Slash commands not visible after invite → missing
  `applications.commands` scope.
- Bot online but `msg.content` empty → missing `MESSAGE_CONTENT` intent.
- Bot's role below user's → cannot kick/ban regardless of permission.
- Channel overrides silently bypass server-wide perms. Check
  `channel.permission_overwrites`.
- Permission integer in invite URL is a misinterpretation — the
  portal shows bits; pasting the wrong integer drops perms silently.
- Admin `Administrator` flag (1 bit, ~bigint) bypasses ALL checks —
  easy to accidentally grant it.

## Sources

- Permissions reference — https://discord.com/developers/docs/topics/permissions
- OAuth2 reference — https://discord.com/developers/docs/topics/oauth2
- Privileged intents — https://discord.com/developers/docs/topics/gateway#privileged-intents
- discord.js PermissionsBitField — https://discord.js.org/#/docs/
- discord.py Perms — https://discordpy.readthedocs.io/
