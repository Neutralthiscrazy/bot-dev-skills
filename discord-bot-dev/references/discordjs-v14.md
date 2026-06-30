---
name: discordjs-v14
description: >-
  Companion digest for discord.js v14 (Node/TypeScript). Client setup,
  event handling, SlashCommandBuilder pattern, interactions (buttons,
  selects, modals), deploying commands per-guild and globally, the v14
  breaking-change list from v13, and common gotchas around tokens +
  intents.
version: 0.1.0
user-invocable: false
author: neutralthiscrazy
license: MIT
---

# discord.js v14 — Node/TypeScript Discord bot

## When to load

- Bot in Node.js or TypeScript (not Python)
- Slash-commands-first design (default for new bots in 2026)
- Want mature, well-documented framework with full feature coverage
  (voice, threads, stage channels, scheduled events, …)

If your project is Python-flavored, use `discord-py.md` instead.

## Install + minimal bot

```bash
mkdir my-bot && cd my-bot
npm init -y
npm install discord.js
```

Minimal bot (`bot.js`):

```javascript
import { Client, GatewayIntentBits, Events } from "discord.js";

const client = new Client({
  intents: [
    GatewayIntentBits.Guilds,
    GatewayIntentBits.GuildMessages,
  ],
});

client.once(Events.ClientReady, (c) => {
  console.log(`Logged in as ${c.user.tag}`);
});

client.on(Events.MessageCreate, (msg) => {
  if (msg.author.bot) return;
  if (msg.content === "!ping") msg.reply("pong");
});

client.login(process.env.DISCORD_TOKEN);
```

Run: `DISCORD_TOKEN=... node bot.js`.

See https://discordjs.guide/ for the full quick-start.

## SlashCommandBuilder pattern (the default)

```javascript
import { SlashCommandBuilder } from "discord.js";

export const data = new SlashCommandBuilder()
  .setName("ping")
  .setDescription("Replies with Pong!");

export async function execute(interaction) {
  await interaction.reply("Pong!");
}
```

Then in your command-loading loop at startup:

```javascript
import { REST, Routes } from "discord.js";

const commands = [];
for (const cmdFile of commandFiles) {
  const cmd = await import(`./commands/${cmdFile}`);
  commands.push(cmd.data.toJSON());
}

const rest = new REST({ version: "10" }).setToken(process.env.DISCORD_TOKEN);

// Register guild-scoped (instant, good for dev)
await rest.put(
  Routes.applicationGuildCommands(APP_ID, GUILD_ID),
  { body: commands }
);

// Register globally (slow, up to 1h to propagate, for prod)
// await rest.put(Routes.applicationCommands(APP_ID), { body: commands });
```

Pitfall #1 from parent SKILL.md: bot must be invited with
`applications.commands` scope for slash to work. See
`references/discord-permissions-and-scopes.md`.

## v14 breaking changes from v13

If your code is older or tutorial was v13:

- `Intents` → `GatewayIntentBits` (bit flags enum, not integer 1 << N)
- `client.ws.on(...)` → `client.on(Events.X, ...)` (enum string keys)
- `MessageEmbed` → `EmbedBuilder` (returns builder, not class)
- All builders (SlashCommandBuilder, etc.) used `add*` now: `addStringOption`
  for SlashCommand, `addSubcommand`, etc.
- Voice @discordjs/voice is now a separate package; v13 imported from core
- `client.user.tag` replaced with structured `client.user.username#discriminator`
  (or just `username` if no discriminator)

## Buttons + Select Menus + Modals (interaction-rich UI)

```javascript
// Button on a component
import { ActionRowBuilder, ButtonBuilder, ButtonStyle } from "discord.js";

const row = new ActionRowBuilder().addComponents(
  new ButtonBuilder()
    .setCustomId("vote_yes")
    .setLabel("Yes")
    .setStyle(ButtonStyle.Success),
);

await interaction.reply({ content: "Vote?", components: [row] });

// Handler (filter on custom_id)
client.on(Events.InteractionCreate, async (interaction) => {
  if (!interaction.isButton()) return;
  if (interaction.customId === "vote_yes") {
    await interaction.update({ content: "Vote: yes", components: [] });
  }
});
```

For deeper interaction patterns (multiple kinds, routing, permissions),
see `references/interactions-and-modals.md`.

## Defer-then-edit pattern (LLM streaming)

```javascript
// ack immediately to get ~15 minutes
await interaction.deferReply();

// later…
await interaction.editReply("Here is the answer…");
```

If your work exceeds 3 seconds, ALWAYS `deferReply` first. After the
interaction token expires (~3s without defer, ~15 min with), no edits
or followups can succeed.

## Common gotchas

- Token in source code → revoke in dev portal immediately + rotate.
- Privileged intents (`MESSAGE_CONTENT`, `PRESENCE`, `SERVER_MEMBERS`):
  requires dev portal toggle, including `GatewayIntentBits.X` in client
  intents array. Without portal toggle, the bot will be alerted but
  events won't fire.
- Slash not appearing after deploy — wait up to 1h for global. For
  dev, deploy guild-scoped to a private guild for instant feedback.
- `interaction.deferReply({ ephemeral: true })` for messages only the
  invoker sees (e.g. error/info).
- `interaction.followUp()` for additional messages after the original
  reply; `interaction.editReply()` only edits the original.

## Sources

- discord.js docs — https://discordjs.guide/
- GitHub — https://github.com/discordjs/discord.js
- API reference — https://discord.js.org/#/docs/
- Discord developer docs — https://discord.com/developers/docs
- Migration guide v13 → v14 — https://discordjs.guide/additional-info/changes-in-v14.html
