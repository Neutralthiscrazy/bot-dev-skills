---
name: vercel-serverless
description: >-
  Companion digest for Vercel Functions as bot host. Default Node
  / Edge runtime config, env vars, Next.js API routes, the API/edge
  distinction, and bundle-size sweet spots for grammy.
version: 0.1.0
user-invocable: false
author: neutralthiscrazy
license: MIT
---

# Vercel Functions as bot host

## When to load

- Existing Next.js / Vercel app where you add a /api bot route
- Webhook-mode bot with no long-lived state
- Cool if you want to share deploy with web UI

If your bot is pure and not attached to a web app, consider Workers
(cheaper edge) or Lambda (more control) instead.

## Setup — minimal webhook route

```typescript
// app/api/webhook/route.ts (Next.js 13+)
import { Bot, webhookCallback } from "grammy";

const bot = new Bot<any>(process.env.BOT_TOKEN!);
bot.command("start", (ctx) => ctx.reply("hi"));

export const POST = webhookCallback(bot, "https");
```

Edge runtime (CF-edge-like, fast cold start):

```typescript
// app/api/webhook/route.ts
export const runtime = "edge";
// ...
```

For Edge runtime, use `webhookCallback(bot, "https", 300_000)` (pass
the timeout as 3rd arg in ms) since Edge has shorter ceiling than
Node runtime.

## Env vars / secrets

```bash
# local dev
vercel env pull .env.local  # syncs project env to local file

# set production
vercel env add BOT_TOKEN production
vercel env add OPENAI_API_KEY production
```

For datasets (KV-like state):

```typescript
import { kv } from "@vercel/kv";
await kv.set(`user:${id}`, JSON.stringify(state));
```

Vercel KV = Upstash Redis under the hood. Free tier has limits.

## Pitfalls

- Vercel Hobby plan = serverless functions 10s timeout (10s = too
  short for most LLM). **Paid plan** extends to 60s @ Node runtime.
  Use Edge runtime for 25s ceiling — usually enough.
- Cold starts ~250-1000ms. Tighter than CF Workers. Use background
  tasks for long streams.
- `runtime = 'edge'` → no Node API; pure Web Standard APIs. Most
  LLM SDKs (openai, anthropic) work fine in edge.
- `runtime = 'nodejs'` → full Node, but slower.

## Costs

| Plan          | Function | Cold start budget | Cap         |
|---------------|---------:|-------------------|-------------|
| Hobby         | Up to 12 | Per-function 10s  | 100GB-hr/mo |
| Pro           | Unlimited| Per-function 60s (Node) | 1TB-hr/mo |
| Enterprise    | Unlimited| Per-function 300s | Custom      |

For deeper cost analysis: https://vercel.com/docs/functions/limits

## Sources

- Vercel Functions — https://vercel.com/docs/functions
- Edge runtime — https://vercel.com/docs/functions/edge-functions
- KV — https://vercel.com/docs/storage/vercel-kv
- Environment variables — https://vercel.com/docs/projects/environment-variables
