---
name: webhook-vs-polling
description: >-
  Companion digest for polling-vs-webhook in Telegram bots. Trade-offs
  by hosting model (VPS, serverless, edge), TLS/cert pain, the
  "setWebhook returns True but updates don't arrive" gotcha, and
  recommended pattern for local dev with ngrok.
version: 0.1.0
user-invocable: false
author: neutralthiscrazy
license: MIT
---

# Polling vs Webhook for Telegram bots

Telegram offers two ways for your bot to receive updates:

- **Long polling**: bot opens a connection to Telegram; new updates
  arrive over the connection. Bot initiates.
- **Webhook**: bot registers a URL with Telegram; Telegram pushes new
  updates over HTTPS to that URL. Telegram initiates.

Both work. The choice is mostly about hosting.

## Decision tree

```
Where does the bot code live?
├─ VPS / dedicated container (DO, EC2, your hardware, Docker):
│  either works. Webhook saves egress; polling avoids HTTPS + cert.
├─ Serverless / FaaS (Vercel, AWS Lambda, GCP CF, CF Workers):
│  WEBHOOK ONLY. No long-running process to poll.
├─ Local dev (laptop): polling, unless ngrok / cloudflared already
│  running with a stable URL.
└─ Edge / multi-region: polling more forgiving.
```

## When polling wins

- Local dev without HTTPS tunnel
- Networks with aggressive NAT or unreliable inbound
- Rapid prototyping ("just run and see")
- Behind reverse proxies with cert issues

## When webhook wins

- Serverless (mandatory)
- Want zero polling lag (immediate delivery)
- Egress metered on VPS
- Multi-bot where each bot has its own cert/URL

## Polling setup

**aiogram:**

```python
await bot.delete_webhook(drop_pending_updates=True)
await dp.start_polling(bot)
```

**grammy:**

```typescript
bot.start();  // long polling by default
```

## Webhook setup (serverless / VPS)

Need public HTTPS endpoint with valid cert (Let's Encrypt is fine).
Then:

```bash
curl "https://api.telegram.org/bot$TG_TOKEN/setWebhook?url=https://your.host/webhook&drop_pending_updates=true"
```

Verify health:

```bash
curl "https://api.telegram.org/bot$TG_TOKEN/getWebhookInfo"
```

Healthy response: `last_error_date: null`, `pending_update_count: 0`.

## Local dev with webhook — ngrok

```bash
ngrok http 8080
# copy https URL, then
curl "https://api.telegram.org/bot$TG_TOKEN/setWebhook?url=https://<random>.ngrok.io/webhook"
```

Free tier URL changes on restart — annoying for dev. Paid tier pins a
subdomain. `cloudflared` (free) is an alternative.

## Pitfalls unique to webhooks

1. **setWebhook returns True but updates don't arrive.** The response
   means "Telegram accepted the request", not "we can reach your URL".
   Always check `getWebhookInfo().last_error_message`. Common causes:
   404 on your route, cert mismatch, 500 from your handler (Telegram retries).
2. **Shared-IP cert issues (SNI).** The cert Telegram sees must match
   your hostname. SNI vhost matters. Certbot + nginx handles this when
   vhost block is set correctly. Test with `curl --resolve`.
3. **Webhook timeout ~10s.** Telegram retries if handler doesn't
   respond. For long-running handlers (LLM, heavy DB): reply 200 fast,
   queue work to background.
4. **CIDR to whitelist.** Telegram webhook POSTs come from
   `149.154.160.0/20` and `91.108.4.0/22`. If filtering by source IP,
   whitelist those.
5. **Drop pending updates when switching modes.** Polling→webhook
   without `drop_pending_updates` leaves a stale backlog gated by
   the old interval timing.

## Sources

- setWebhook — https://core.telegram.org/bots/api#setwebhook
- getWebhookInfo — https://core.telegram.org/bots/api#getwebhookinfo
- aiogram webhook — https://aiogram.dev/
- grammy hosting — https://grammy.dev/hosting/
- Telegram server IPs — https://core.telegram.org/bots/webhooks
