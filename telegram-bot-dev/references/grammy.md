---
name: grammy
description: >-
  Companion digest for grammy (TypeScript-first Telegram bot framework).
  Setup, Composer modularity, Conversations plugin for FSM, Menu plugin
  for repliable UI, serverless-friendly webhookCallback adapter, and
  Vercel/Cloudflare Workers deployment patterns.
version: 0.1.0
user-invocable: false
author: neutralthiscrazy
license: MIT
---

# grammy — TypeScript-first Telegram bot framework

## When to load

- Bot written in TypeScript or JavaScript (not Python)
- Deploy target is serverless (Vercel, CF Workers, AWS Lambda, Deno)
- Want conversations / FSM as a plugin rather than ad-hoc states
- Porting an aiogram bot and want similar async API feel

If your project is Python-flavored, use `aiogram-3x.md` instead.

## Install + minimal bot

```bash
npm init -y
npm install grammy
```

```typescript
import { Bot } from "grammy";

const bot = new Bot(process.env.TG_TOKEN!);

bot.command("start", (ctx) => ctx.reply("hello, " + ctx.from?.first_name));

bot.start();
```

See https://grammy.dev/ for full quick-start.

## Composer (modular handlers)

`Composer` groups handlers; then you sandwich composers into the bot:

```typescript
import { Bot, Composer } from "grammy";

const adminComposer = new Composer();
adminComposer.command("ban", (ctx) => /* ... */);

const userComposer = new Composer();
userComposer.command("start", (ctx) => ctx.reply("Welcome!"));

const bot = new Bot(process.env.TG_TOKEN!);
bot.use(adminComposer);
bot.use(userComposer);
```

Order matters: list most-specific composers first (auth gate before user
handler). Each composer can have its own middleware via `composer.use(...)`.

## Conversations plugin (built-in FSM)

```typescript
import { Bot, Context, session } from "grammy";
import { conversations, createConversation } from "@grammyjs/conversations";

type MyContext = Context & session.FlushData<{ step?: number }>;

const bot = new Bot<MyContext>(process.env.TG_TOKEN!);
bot.use(session({ initial: () => ({}) }));
bot.use(conversations());

async function feedback(conversation: any, ctx: MyContext) {
  await ctx.reply("name?");
  const nameCtx = await conversation.waitFor("message:text");
  await ctx.reply(`Hi ${nameCtx.msg.text}. email?`);
  const emailCtx = await conversation.waitFor("message:text");
  await ctx.reply(`Send to ${emailCtx.msg.text}?`);
  await conversation.waitFor("callback_query:data");
  await ctx.reply("Done.");
}

bot.use(createConversation(feedback));
bot.command("feedback", (ctx) => ctx.conversation.enter("feedback"));
```

For FSM design concepts (state split, validation, timeout) —
see `telegram-fsm-patterns.md`.

## Serverless deployment (the killer feature)

`webhookCallback` adapts the bot to any handler interface.

**Vercel** (`api/webhook.ts`):

```typescript
import { Bot, webhookCallback } from "grammy";

const bot = new Bot(process.env.TG_TOKEN!);
bot.command("start", (ctx) => ctx.reply("hi"));

export const POST = webhookCallback(bot, "https", 300_000);
```

**Cloudflare Workers** (`src/worker.ts`):

```typescript
import { Bot, webhookCallback } from "grammy";

const bot = new Bot(process.env.TG_TOKEN!);
bot.command("start", (ctx) => ctx.reply("hi"));

addEventListener("fetch", (event) => {
  event.respondWith(webhookCallback(bot, "cloudflare")(event.request));
});
```

CF Workers 1MB bundle limit — grammy + conversations + menu fits;
heavy Prisma clients may not.

## Menu plugin (button-driven menu trees)

```typescript
import { Menu } from "@grammyjs/menu";

const main = new Menu("main")
  .text("Help", (ctx) => ctx.reply("..."))
  .row()
  .submenu("More", "more");

const more = new Menu("more")
  .text("Contact", (ctx) => ctx.reply("..."))
  .back("Back");

bot.use(main, more);
bot.command("menu", (ctx) => ctx.reply("Pick:", { reply_markup: main }));
```

## Common gotchas

- Webhook **must return 200 within ~10s**. Long LLM replies → send a
  placeholder, then edit in background.
- `bot.catch((err) => ...)` is a global error trap. Without it exceptions
  die silently — always install one.
- Strict TS + grammy can fight on `ctx.session` typing. Cast when needed.
- CF Workers + `Buffer` — use platform-specific adapter; CF expects
  `Uint8Array`.
- Conversations timeout — long forms idle and expire. Configure
  `idleTimeout` per conversation.

## Sources

- grammy docs — https://grammy.dev/
- GitHub — https://github.com/grammyjs/grammY
- Hosting — https://grammy.dev/hosting/
- Conversations — https://grammy.dev/plugins/conversations
- Menu — https://grammy.dev/plugins/menu
