---
name: serverless-bot-deploy
description: >-
  Use when deploying a chat bot (Telegram, Discord, etc.) to a
  serverless platform - Cloudflare Workers, Vercel Functions, AWS
  Lambda, GCP Cloud Functions, Deno Deploy. Webhook-only delivery,
  cold-start math, bundling constraints, secret managers per
  platform, and the "long-lived gateway" trap that breaks CF Workers
  for bots that need persistent connections.
version: 0.1.0
user-invocable: false
author: neutralthiscrazy
license: MIT
---

# Serverless bot deployment

## Overview

Serverless works well for **webhook-mode chat bots** (Telegram +
Discord both use HTTP POST for slash-command deliveries). It does NOT
work for bots that need persistent gateway connections (Discord
on_message events with presence, some Telegram long-polling flows,
Discord voice). This skill covers the former.

Authored from first-party docs (`developers.cloudflare.com/workers`,
`vercel.com/docs/functions`, AWS Lambda docs) and the framework-
specific deployment recipes (`grammy.dev/hosting`).

## When to Use

- Bot runs on Telegram and uses webhooks (or Discord slash-only)
- Deploy target is one of: Cloudflare Workers, Vercel, AWS Lambda,
  GCP CF, Deno Deploy, Netlify
- Cost / scale / simplicity matters (no VPS management)
- Cold-start sensitive workload (avoid 10s cold starts)

## When NOT to Use

- Discord bot needs `gateway` WebSocket (presence, on_message with
  content): use VPS. CF Workers and Lambda don't support long-lived
  WebSockets.
- Voice bots: VPS / Docker required.
- Heavy database clients (Prisma, Surrealdb Rust core): bundle size
  often exceeds serverless limits.

## Routing Table

| User intent                                  | Load                                       |
|----------------------------------------------|--------------------------------------------|
| "Cloudflare Workers - HTTP bot handler"      | `references/cloudflare-workers.md`         |
| "Vercel Functions - Telegram or Discord"     | `references/vercel-serverless.md`          |
| "AWS Lambda - bot with full secret mgmt"     | `references/aws-lambda-discord-friends.md` |
| "Docker container for Telegram webhook"      | `references/docker-telegram-bot.md`        |
| "Secrets / env vars / API keys management"   | `references/secrets-and-env-management.md` |
| "Cold-start, bundle size, warm-up tactics"  | included inline below + each platform ref |

## Platform comparison

| Platform        | Cold start | Memory cap | Bundle size | WebSocket?    | Best for |
|-----------------|-----------:|-----------:|------------:|---------------|----------|
| CF Workers      | 5-50ms     | 128MB      | 1MB         | Durable Obj. only | Small TS bots, edge |
| Vercel          | 250-1000ms | ~3GB       | 250MB       | No             | Node Next.js mixed |
| AWS Lambda      | 200ms-2s   | 10GB       | 250MB       | No             | Anything Python/Node |
| Deno Deploy     | 30-150ms   | 512MB      | ~unbounded  | No             | TS-first projects |
| GCP Cloud Func  | 200ms+     | 32GB (2nd gen)| 500MB    | No             | Google ecosystem |

For *very* cold-start-sensitive: CF Workers / Deno win. For
*memory-heavy* (LLM context, large DB): Lambda or Vercel.

## Universal gotchas

1. **Webhook verification.** Cloudflare, Vercel, Lambda — all need
   a public HTTPS endpoint with valid cert. Most platforms auto-
   provision this.
2. **No long-running state.** Set up Redis / Upstash / Cloudflare
   KV / DynamoDB for any bot state (FSM context, user session).
3. **Secrets must not be in source code.** Use platform-native
   secret managers (CF env / wrangler, Vercel env, AWS SSM / Secrets
   Manager). See `references/secrets-and-env-management.md`.
4. **Timeouts.** Platform limits per-request: CF Workers default
   30s (CPU), Vercel 10-60s, Lambda up to 15min. For LLM streaming
   use deferred replies / background tasks.
5. **Bundle size.** Discord.js full bundle is ~1.6MB unminified,
   exceeding CF Workers limit. Use `discord.js` outer parts only or
   hand-rolled REST.
6. **Cold-start lag.** First request after deploy can take 200ms-2s.
   Use cached "warm-up" ping for hot path. Not always worth it.

## Common Pitfalls

1. **Bringing full Discord.js into a Worker.** Bundle: 1.6MB unminified
   > 1MB limit. Strip to REST-only or use raw fetch.
2. **WebSocket instead of webhook.** Some tutorials show discord.js
   client in CF Workers with gateway — does NOT work, Workers die
   before Discord sends heartbeat (40s).
3. **Timezones in cron-style tasks.** CF Cron Triggers / Vercel Cron
   run in UTC. Plan display-string for user-timezone in code.
4. **Missing `Content-Type: application/json` request body.** Lambda
   with proxy-integration drops body unless Content-Type set.
5. **Settings set on dev but not prod.** Use env-based config:
   platform-specific secret manager.

## Verification Checklist

- [ ] HTTP endpoint answers on HTTPS — webhook set to that URL
- [ ] `getWebhookInfo` shows expected URL with no pending errors
- [ ] Cold-start time <2s for first request after idle
- [ ] Secrets read via platform secret manager (not process.env in
     plain JSON)
- [ ] Bundle size stays under platform limit (CF Workers: `wrangler
     deploy --dry-run`)
- [ ] Logs accessible (CF `wrangler tail`, Vercel dashboard, CloudWatch)

## Related Skills

- `telegram-bot-dev` — Telegram-side framework
- `discord-bot-dev` — Discord-side framework
- `llm-streaming-reply` — LLM into chat
- `bot-async-runtime` — async-runtime gotchas

## Sources cited inline

- Cloudflare Workers — https://developers.cloudflare.com/workers/
- Vercel Functions — https://vercel.com/docs/functions
- AWS Lambda — https://docs.aws.amazon.com/lambda/
- Deno Deploy — https://docs.deno.com/deploy/
- GMP CF — https://cloud.google.com/functions/docs/
