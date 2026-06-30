---
name: docker-telegram-bot
description: >-
  Companion digest for Docker-based bot deployment (Dockerfile,
  docker-compose, systemd-fed Docker, bot + Redis + Postgres
  combinations). When you don't want serverless, a Docker container
  is the typical "permanent" host.
version: 0.1.0
user-invocable: false
author: neutralthiscrazy
license: MIT
---

# Docker hosting (Telegram webhook bot)

## When to load

- Want a single, simple deploy target (Docker on a VPS)
- Need long-running state (FSM, sessions, queues)
- Sidecars like Redis or Postgres required

If pure serverless works, it's cheaper and simpler.

## Minimal Dockerfile (Python)

```dockerfile
FROM python:3.13-slim

WORKDIR /app
RUN apt-get update && apt-get install -y --no-install-recommends \
        ca-certificates curl && rm -rf /var/lib/apt/lists/*

COPY requirements.txt ./
RUN pip install --no-cache-dir -r requirements.txt

COPY . .

# Run as non-root
RUN useradd -m bot && chown -R bot:bot /app
USER bot

CMD ["python", "-m", "bot"]
```

Healthcheck:

```dockerfile
HEALTHCHECK --interval=30s --timeout=5s \
  CMD curl -fs http://localhost:8080/health || exit 1
```

`--no-install-recommends` keeps image small. Multi-stage for build
deps:

```dockerfile
FROM python:3.13-slim AS build
RUN pip wheel --wheel-dir=/wheels -r requirements.txt

FROM python:3.13-slim
COPY --from=build /wheels /wheels
RUN pip install --no-index --find-links=/wheels /wheels/*
```

## docker-compose with sidecars

```yaml
# docker-compose.yml
services:
  bot:
    build: .
    restart: unless-stopped
    env_file: .env
    depends_on:
      - redis
      - postgres
    ports:
      - "8080:8080"  # webhook server

  redis:
    image: redis:7-alpine
    restart: unless-stopped
    volumes:
      - redis-data:/data

  postgres:
    image: postgres:16-alpine
    restart: unless-stopped
    environment:
      POSTGRES_PASSWORD: ${PG_PASSWORD}
    volumes:
      - pg-data:/var/lib/postgresql/data

volumes:
  redis-data:
  pg-data:
```

Webhook mode (vs polling): this Dockerfile exposes port 8080, has
healthcheck, ready for ALB / Traefik / nginx in front.

## Polling mode (no port, simpler)

Drop the port binding; polling needs no public endpoint but
requires the bot process to keep running.

## Sources

- docker docs — https://docs.docker.com/
- docker-compose — https://docs.docker.com/compose/
- Python slim images — https://hub.docker.com/_/python
- aiogram aiohttp server — https://aiogram.dev/
