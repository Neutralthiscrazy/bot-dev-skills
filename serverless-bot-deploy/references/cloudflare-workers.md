---
name: cloudflare-workers
description: >-
  Companion digest for Cloudflare Workers as a bot deployment. wrangler
  config, fetch handler shape, env vars / secrets, KV for bot state,
  Durable Objects for long-lived task, bundle-size constraints, and
  beginner anti-patterns (gRPC, full Discord.js, WebSocket to Discord).
version: 0.1.0
user-invocable: false
author: neutralthiscrazy
license: MIT
---

# Cloudflare Workers as bot host

## When to load

- Bot is webhook-mode (Telegram webhook or Discord slash-only)
- Edge latency matters / global distribution desired
- Node or TS source; Hono or raw fetch handler
- Tiny bot that fits in 1MB bundle

If your bot needs full Discord `gateway` (presence, voice, on_message
with content), skip Workers — VPS or AWS instead. Workers cannot
maintain the 30-40s heartbeat to Discord.

## wrangler.toml minimal

```toml
name = "my-bot"
main = "src/index.ts"
compatibility_date = "2026-04-01"

[vars]
ADMIN_USER_ID = "..."

# Secrets via wrangler secret put NAME
# wrangler secret put BOT_TOKEN
```

`compatibility_date` pins the runtime. Updating on Workers =
updating runtime features. Don't skip the date.

## Handler with grammy

```typescript
import { Bot, webhookCallback } from "grammy";
import { Hono } from "hono";

const bot = new Bot<any>(process.env.BOT_TOKEN!);

// use webhookCallback with Hono
const app = new Hono();
app.post("/webhook", webhookCallback(bot, "cloudflare"));

export default app;
```

For grammy on Workers see `grammy.dev/hosting/cloudflare-workers-nodejs`.

## Bundle size (the wall you'll hit first)

CF Workers bundle limit: **1MB minified**. Discord.js minified is
~800KB; full grammy + plugins ~600KB. Likely fits, but verify with:

```bash
wrangler deploy --dry-run --outdir=dist
ls -lh dist/index.js  # size
```

If over, candidates:
- Replace discord.js with raw fetch + minimal REST wrapper (cuts
  bundle ~70%).
- Use `esbuild` minifier with `minifyIdentifiers` and `minifySyntax`.
- Drop unused grammy plugins.

## Webhook URL routing

Set platform to send POST requests to your Workers URL:

```bash
# wrangler creates https://<name>.<subdomain>.workers.dev route
curl "https://api.telegram.org/bot$TG_TOKEN/setWebhook?url=https://my-bot.my-worker-subdomain.workers.dev/webhook"
```

For custom domain, bind route in wrangler.toml:

```toml
routes = [
  { pattern = "bot.mycompany.com/webhook", custom_domain = true }
]
```

Plus TLS termination is automatic.

## KV / Durable Objects for state

```toml
# wrangler.toml
[[kv_namespaces]]
binding = "FSM"
id = "<random-id>"
```

```typescript
// in handler
const ctx = c.env.FSM;
await ctx.put(`user:${user_id}:state`, JSON.stringify(state));
const state = JSON.parse(await ctx.get(`user:${user_id}:state`) || "{}");
```

For more complex state (FSM, conversation history), Durable Objects
give a single-threaded worker per user_id.

## Common pitfall (the big one)

**Discord gateway does NOT work in Workers.** Discord expects a long
WebSocket; Workers runtime kills it at 30s. Slash commands only —
which means: no presence, no on_message events, no reactions, no
anything beyond what slash commands can do.

If you need these, choose VPS.

## Pitfalls

- Bindings are exposed via `env.<NAME>` in the handler context, not
  process.env. Cloudflare runtime is V8 isolate — there's no
  `process` in the browser sense.
- Global state doesn't survive across requests. Always read from
  KV / DO / R2.
- Workers limits: 10ms CPU on free plan, 30s CPU on paid (per-request).
  Background tasks aren't supported; cron triggers ARE.
- Don't use `setInterval`, `setTimeout` for anything — Workers
  terminate after handler returns.

## Sources

- Cloudflare Workers docs — https://developers.cloudflare.com/workers/
- wrangler — https://developers.cloudflare.com/workers/wrangler/
- KV — https://developers.cloudflare.com/kv/
- Durable Objects — https://developers.cloudflare.com/durable-objects/
- Telegram webhook setup with Workers — https://core.telegram.org/bots/webhooks
