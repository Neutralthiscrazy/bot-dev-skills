---
name: hosting-discord-bots
description: >-
  Companion digest for hosting Discord bots — VPS (systemd, Docker,
  Docker Compose), serverless on Cloudflare Workers with a twist
  (gateway WebSocket connections), and a word on the "Discord
  connection-magic" which doesn't fit long-lived workers cleanly.
version: 0.1.0
user-invocable: false
author: neutralthiscrazy
license: MIT
---

# Hosting Discord bots

## When to load

- Picking hosting model (VPS vs Docker vs serverless)
- Bridging "Discord gateway = WebSocket" with platforms that don't
  support long-lived connections
- Writing a Dockerfile or Compose for the bot + database
- Setting up systemd for a Node or Python VPS bot

## Hosting options matrix

| Target               | Gateway (WebSocket)   | Slash commands (HTTP) | Best option |
|----------------------|-----------------------|-----------------------|-------------|
| VPS (EC2, DO, etc.)  | Easy                  | Easy                  | **Best** — full control |
| Docker / Compose     | Easy                  | Easy                  | **Common** — simple deploy |
| Cloudflare Workers   | Hard (no WS)          | Easy                  | Slash-only, no presence |
| AWS Lambda           | Hard (no WS, no long)| Easy                  | Slash-only, no presence |
| Vercel               | Hard (no WS)          | Easy                  | Same as CF Workers |

Long-lived Discord **gateway WebSocket** connections require a
runtime supporting persistent connections. CF Workers and Lambda
time out before Discord sends heartbeat. **For a full bot with
presence / message-content / member events: VPS or Docker.** For
**slash-commands-only bots with deferred replies**: serverless is fine.

## VPS — Node bot

```bash
# Install Node, clone repo, install
sudo apt install -y nodejs npm
git clone git@github.com:my-username/my-discord-bot.git
cd my-discord-bot
npm ci --omit=dev

# Run as a systemd service
sudo tee /etc/systemd/system/discord-bot.service <<EOF
[Unit]
Description=Discord bot
After=network.target

[Service]
Environment=DISCORD_TOKEN=...redacted...
ExecStart=/usr/bin/node /opt/my-bot/bot.js
Restart=on-failure
User=botuser

[Install]
WantedBy=multi-user.target
EOF
sudo systemctl daemon-reload
sudo systemctl enable --now discord-bot.service
```

Logs: `journalctl -u discord-bot -f`.

## VPS — Python bot with systemd

```ini
# /etc/systemd/system/discord-bot.service
[Service]
Environment=DISCORD_TOKEN=...redacted...
ExecStart=/opt/my-bot/.venv/bin/python /opt/my-bot/bot.py
Restart=on-failure
```

Same shape; use `python -m bot` if you have a package layout.

## Docker / Compose

```dockerfile
# Dockerfile
FROM node:22-alpine
WORKDIR /app
COPY package*.json ./
RUN npm ci --omit=dev
COPY . .
CMD ["node", "bot.js"]
```

```yaml
# docker-compose.yml
services:
  bot:
    build: .
    restart: unless-stopped
    environment:
      DISCORD_TOKEN: ${DISCORD_TOKEN}
    volumes:
      - bot-data:/data   # for cogs that persist state
volumes:
  bot-data: {}
```

`restart: unless-stopped` keeps the bot up across host reboots and
recovers from crashes.

For Python:

```dockerfile
FROM python:3.13-slim
WORKDIR /app
COPY requirements.txt ./
RUN pip install --no-cache-dir -r requirements.txt
COPY . .
CMD ["python", "bot.py"]
```

## Cloudflare Workers (slash-only)

You give up real-time presence / on-message events, but commands
that are pure slash-command-driven work fine. Store token in CF
secrets.

```typescript
import { Client, GatewayIntentBits } from "discord.js";  // gateway
// Actually discord.js doesn't fit Workers bundle limits. Use raw
// fetch + the interactions endpoint.
```

For Workers-scale bots, prefer raw fetch + Discord's REST API rather
than full discord.js (bundle > 1MB fails worker size limit). Many
companies use `itty-discord` or hand-rolled HTTP handling.

## Common pitfalls

- Bot offline in client but service says "running" — usually token
  env-var not propagated to systemd. Set `Environment=...` in the
  unit file or use `EnvironmentFile=/etc/discord-bot.env`.
- Discord.js runs single-threaded — one bad CPU-bound extension
  blocks everything. Long handler work goes to a Task / queue.
- Serverless timeout on long interactions. `deferReply()` then async
  work, but the closing edit must complete within 15 min or
  Discord kills the token.
- Heartbeat failed on CF Workers — Workers die before the
  30-40s heartbeat interval. Use VPS for any bot relying on
  persistent state, presence, etc.
- Sharing token across two running instances (e.g. dev + prod by
  accident) → Discord rate-limits both and effectively neither works.

## Sources

- discord.js hosting guide — https://discordjs.guide/miscellaneous/useful-snippets.html#hosting-on-a-vps-with-systemd
- Discord developer docs (gateway) — https://discord.com/developers/docs/topics/gateway
- Cloudflare Workers Discord bot example (raw fetch) — https://developers.cloudflare.com/workers/tutorials/
- Discord rate limits — https://discord.com/developers/docs/topics/rate-limits
