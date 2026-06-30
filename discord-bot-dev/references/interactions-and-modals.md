---
name: interactions-and-modals
description: >-
  Companion digest for Discord interaction types — buttons, select
  menus, modals, context menus — and the routing patterns to handle
  them. Covers the four interaction response types (Pong, Channel,
  DeferredChannel, DeferredUpdate), interaction tokens, custom_id
  collision avoidance, and component v2 (Container, Section, TextDisplay).
version: 0.1.0
user-invocable: false
author: neutralthiscrazy
license: MIT
---

# Discord interactions — buttons, modals, and the routing layer

## When to load

- You're designing slash commands with rich UI replies
- Need to ask for follow-up input (modals = pop-up forms)
- Need multi-action surfaces (rows of buttons, drop-down selects)
- Hit "InteractionNotFound" / "Unknown Interaction" errors

This is the routing-layer reference. Pair it with the framework
reference for the API surface (`discordjs-v14.md` or `discord-py.md`).

## The four interaction-response types

Discord returns all interactions with **one of four response types**.
Choosing the right one is the difference between a working interaction
and a 3-second timeout nightmare.

| Type                       | Use                                            | API call                  |
|----------------------------|------------------------------------------------|---------------------------|
| **PONG**                   | You're not going to respond with a message     | (rare; no actual call)    |
| **CHANNEL_MESSAGE_WITH_SOURCE** | Reply with a message (the default)       | `interaction.reply(...)`  |
| **DEFERRED_CHANNEL_MESSAGE_WITH_SOURCE** | Will reply but it might take >3s | `interaction.deferReply()` |
| **DEFERRED_UPDATE_MESSAGE** | Will update the original ("thinking…" buttons)| `interaction.deferUpdate()` |
| **UPDATE_MESSAGE**         | Update the original message immediately        | `interaction.update(...)` |
| **MODAL**                  | Open a modal popup                             | `interaction.showModal(...)` |

The **15-minute edit window** is your friend: `deferReply()` then
`editReply()` later. The **3-second** no-action window is your enemy:
if your handler exceeds 3s before any of the above, the interaction
token expires and you get `10062: Unknown Interaction`.

## Buttons

```javascript
// discord.js
import { ActionRowBuilder, ButtonBuilder, ButtonStyle, Events } from "discord.js";

const row = new ActionRowBuilder().addComponents(
  new ButtonBuilder().setCustomId("vote_yes").setLabel("Yes").setStyle(ButtonStyle.Success),
  new ButtonBuilder().setCustomId("vote_no").setLabel("No").setStyle(ButtonStyle.Danger),
);

await interaction.reply({ content: "Vote?", components: [row] });

client.on(Events.InteractionCreate, async (interaction) => {
  if (!interaction.isButton()) return;
  switch (interaction.customId) {
    case "vote_yes":
      await interaction.update({ content: "Voted yes.", components: [] });
      break;
    case "vote_no":
      await interaction.update({ content: "Voted no.", components: [] });
      break;
  }
});
```

```python
# discord.py
class VoteView(discord.ui.View):
    @discord.ui.button(label="Yes", style=discord.ButtonStyle.success, custom_id="vote_yes")
    async def yes(self, button, interaction):
        await interaction.response.edit_message(content="Voted yes.", view=None)

    @discord.ui.button(label="No", style=discord.ButtonStyle.danger, custom_id="vote_no")
    async def no(self, button, interaction):
        await interaction.response.edit_message(content="Voted no.", view=None)

view = VoteView(timeout=300)  # expire after 5min
await ctx.respond("Vote?", view=view)
```

`timeout=300` makes the view auto-expire after 5min — interactions on
expired buttons silently fail with `MessageComponentNotFound`.

## Select Menus (drop-down)

```javascript
import { StringSelectMenuBuilder, StringSelectMenuOptionBuilder } from "discord.js";

const sel = new StringSelectMenuBuilder()
  .setCustomId("color_pick")
  .setPlaceholder("Choose a color")
  .addOptions(
    new StringSelectMenuOptionBuilder().setLabel("Red").setValue("red"),
    new StringSelectMenuOptionBuilder().setLabel("Green").setValue("green"),
    new StringSelectMenuOptionBuilder().setLabel("Blue").setValue("blue"),
  );

const row = new ActionRowBuilder().addComponents(sel);
await interaction.reply({ content: "Pick:", components: [row] });

// Handler
client.on(Events.InteractionCreate, async (interaction) => {
  if (!interaction.isStringSelectMenu()) return;
  const choice = interaction.values[0];  // "red" | "green" | "blue"
  await interaction.update({ content: `Picked ${choice}.`, components: [] });
});
```

Discord allows max 25 options per StringSelectMenu, max 10 select
menus per message (rare in practice).

## Modals (pop-up forms)

```javascript
import { ModalBuilder, TextInputBuilder, TextInputStyle } from "discord.js";

const modal = new ModalBuilder()
  .setCustomId("feedback_modal")
  .setTitle("Feedback form");

const nameInput = new TextInputBuilder()
  .setCustomId("name")
  .setLabel("Your name")
  .setStyle(TextInputStyle.Short)
  .setRequired(true);

const feedbackInput = new TextInputBuilder()
  .setCustomId("feedback")
  .setLabel("Your feedback")
  .setStyle(TextInputStyle.Paragraph)
  .setMinLength(20)
  .setMaxLength(2000);

modal.addComponents(
  new ActionRowBuilder().addComponents(nameInput),
  new ActionRowBuilder().addComponents(feedbackInput),
);

await interaction.showModal(modal);

// Modal submit handler (separate event)
client.on(Events.InteractionCreate, async (interaction) => {
  if (!interaction.isModalSubmit()) return;
  const name = interaction.fields.getTextInputValue("name");
  const feedback = interaction.fields.getTextInputValue("feedback");
  await interaction.reply({ content: `Got feedback from ${name}.`, ephemeral: true });
});
```

Max 5 components per modal (each row = 1 component).

## Context Menus (right-click)

Two kinds: user (right-click a user) and message (right-click a message).
Register separately on the application:

```javascript
const userCmd = new ContextMenuCommandBuilder()
  .setName("User info")
  .setType(ApplicationCommandType.User);

const msgCmd = new ContextMenuCommandBuilder()
  .setName("Report")
  .setType(ApplicationCommandType.Message);
```

Handle:

```javascript
client.on(Events.InteractionCreate, async (interaction) => {
  if (!interaction.isUserContextMenuCommand()) return;
  // interaction.targetUser, .targetMember
});
```

## Custom ID collisions

`custom_id` length max 100 chars. Use a `prefix:suffix` pattern to
namespace and route:

```javascript
// Instead of:
data.setCustomId("accept")  // generic, collides across views

// Use prefix:
.setCustomId("poll:42:accept")   // poll #42, accept button
```

In dispatcher, split by `:`:

```javascript
const [type, id, action] = interaction.customId.split(":");
```

This lets one handler route across many `Views` and survives reuse
across multiple messages.

## Pitfalls

- Some interaction responses are **deferred then require a followup** —
  not editable directly. Use `editReply` for the original; `followUp`
  for additional channels.
- `ephemeral: true` makes the reply only visible to the invoker.
  Use for sensitive info (errors, personal data).
- Disabled components in an old message can still be clicked by users
  that pulled up the message before disable — Discord re-verifies on
  click. The "race" is benign.
- `MessageComponentNotFound` = the message was deleted or the view
  timed out. Don't crash — log and ack.
- Component v2 (Container, Section, TextDisplay) is in beta as of
  2026-Q2 — Discord has toggled it on for some bots. Avoid if you
  must support non-beta clients.

## Sources

- Interactions — https://discord.com/developers/docs/interactions/
- Components v1 — https://discord.com/developers/docs/interactions/message-components
- Modals — https://discord.com/developers/docs/interactions/modals
- Component v2 (beta) — https://discord.com/developers/docs/interactions/message-components#component-types
- discord.js components — https://discordjs.guide/interactions/buttons.html
