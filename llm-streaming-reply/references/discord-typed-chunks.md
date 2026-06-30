---
name: discord-typed-chunks
description: >-
  Companion digest for the "streaming into Discord" pattern. deferReply
  + editReply loop + followup.send at 2000-char cap + ephemeral
  fallbacks. Discord's 15-minute edit window is the largest of any
  platform — chunking is rare.
version: 0.1.0
user-invocable: false
author: neutralthiscrazy
license: MIT
---

# Discord — typed chunks via deferReply + editReply

## When to load

- Streaming an LLM reply into Discord
- Slash command with deferred reply (3s window → 15min window)
- Hitting 2000-char limit and using `followup.send` for additional
- Hitting message-component / embed limits

## Setup (discord.js)

```javascript
import { ApplicationCommandType } from "discord.js";

// 1) Slash command interceptor — deferReply immediately
export const data = new SlashCommandBuilder()
  .setName("ask")
  .setDescription("Ask Claude anything")
  .addStringOption((o) => o.setName("question").setRequired(true));

export async function execute(interaction) {
  const question = interaction.options.getString("question");
  await interaction.deferReply();
  await streamIntoReply(interaction, question);
}

async function streamIntoReply(interaction, question) {
  let buffer = "";
  let lastEdit = Date.now();
  for await (const delta of llmStream(question)) {
    buffer += delta;
    if (Date.now() - lastEdit > 1000) {
      if (buffer.length <= 2000) {
        await interaction.editReply(buffer);
      } else {
        // hit cap, push remainder via followup
        const remainder = buffer.slice(2000);
        await interaction.followUp(remainder);
        buffer = buffer.slice(0, 2000);
        await interaction.editReply(buffer);
      }
      lastEdit = Date.now();
    }
  }
  // final flush
  await interaction.editReply(buffer);
}
```

## Setup (discord.py)

```python
@bot.slash_command(name="ask")
async def ask(ctx: discord.ApplicationContext, question: str):
    await ctx.defer()
    buffer = ""
    async for delta in llm_stream(question):
        buffer += delta
        if len(buffer) > 2000:
            await ctx.followup.send(buffer[2000:])
            buffer = buffer[:2000]
            await ctx.edit(content=buffer)
    await ctx.edit(content=buffer)
```

## Discord vs Telegram — the 15-minute window

After `deferReply()`, you have 15 minutes to edit the same reply.
Telegram allows no formal edit window but messages get old.
Discord's generous window supports truly streaming inputs (full
3-5 minute LLM responses are fine).

## 2000-char cap (Discord Standard Message)

A normal text message caps at 2000 chars. Beyond:

- Send a `followup.send` for each extra chunk
- Use an Embed (`discord.Embed`, `description` field) for up to
  4096 chars
- For really long responses, switch to an attached `.md` or `.txt`
  file via `ctx.respond(file=discord.File(...))`

## Pitfalls

- `interaction.editReply(text)` ONLY edits the first reply, never a
  followup. Use `interaction.editFollowup(messageId, text)` after a
  followup.
- `followup.send` after stream end is fine; rate-limit is one per
  second-ish on average.
- `interaction.editReply` does NOT accept `embeds` while the original
  reply has components. Component-then-embed pattern: send an update
  with both, or send followup with embeds.
- "Can't edit ephemeral message" — ephemeral messages have different
  rules. Don't try to edit them in a stream.

## Sources

- Interactions reply deferred — https://discord.com/developers/docs/interactions/slash-commands#responding-to-an-interaction
- Message limits — https://discord.com/developers/docs/resources/channel#message-limits
- webhooks/followups — https://discord.com/developers/docs/resources/webhook#execute-webhook
