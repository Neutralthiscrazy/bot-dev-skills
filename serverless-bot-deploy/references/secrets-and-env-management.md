---
name: secrets-and-env-management
description: >-
  Companion digest for bot-side secrets. Env vars vs secrets manager,
  per-platform options (CF wrangler, Vercel, AWS, local dev),
  .env vs .env.local vs .env.production, and the "secret leaked
  in git" emergency response.
version: 0.1.0
user-invocable: false
author: neutralthiscrazy
license: MIT
---

# Secrets and env vars

## When to load

- Bot has API keys (OpenAI, Anthropic, Discord/Telegram bots) that
  must NOT live in source code
- Multiple environments (dev / staging / production)
- Setting up a new hosting target (CF / Vercel / AWS)
- Just leaked a key and need to revoke fast

## Three-tier secret model

| Tier | Examples                       | Storage                  |
|------|--------------------------------|--------------------------|
| BUILD | NPM token, GitHub PAT for CI  | Build-time env, never runtime |
| RUNTIME_PUBLIC | API URL, project ID | Process env, safe |
| RUNTIME_SECRET | BOT_TOKEN, OPENAI_API_KEY | Platform secret manager |

The disaster scenario: putting the RUNTIME_SECRET tier in any of:
- Code committed
- Process env printed in logs
- DSN in URL query string visible in browser dev tools
- Front-end readable config

→ all visible on a public GitHub repo. Always use secret manager.

## Per-platform secrets

### CF Workers

```bash
wrangler secret put BOT_TOKEN
```

Reads from stdin, never disk-stored. Available in handler:

```typescript
const token = c.env.BOT_TOKEN;  // c.env, not process.env
```

### Vercel

```bash
vercel env add BOT_TOKEN production
# or per all environments
vercel env add BOT_TOKEN
```

Inside Next.js route: `process.env.BOT_TOKEN`. In Vercel, access is
restricted per env (production, preview, development).

### AWS

```bash
aws secretsmanager create-secret \
  --name telegram/bot-token \
  --secret-string "..."
```

In Lambda: `secrets.get_secret_value(SecretId="...")`. Cache client
across invocations to avoid connect-tax on every cold start.

### Local dev

```bash
echo "BOT_TOKEN=..." > .env.local
echo ".env.local" >> .gitignore
```

`.env` (committed) can have non-sensitive config. `.env.local` (not
committed) has secrets.

## Pitfalls

- `process.env` in Cloudflare Workers doesn't exist — use `c.env.X`.
- `process.env` in Edge runtime (Vercel/Next) is a thin wrapper, but
  some keys aren't available there without configuration.
- Bot not running because `.env.local` not present locally AND
  platform secret not set in deploy → confusing.
- Front-end code can see `NEXT_PUBLIC_*` envs ONLY. Don't name
  secrets with that prefix.

## Emergency: leaked key response

1. **Stop the bleed.** Revoke the compromised token immediately:
   - Telegram: `@BotFather` → `/revoke`
   - Discord: dev portal → reset token
   - OpenAI: dashboard → revoke key
   - Anthropic: dashboard → rotate
2. **Rotate to a new key.** Update your platform secret.
3. **Audit damage.** Most providers show usage logs — check for
   unexpected callers / sources.
4. **Remove from history.** If committed to public git, the token
   stays in history. Even `git rm` doesn't un-commit. BFG Repo-
   Cleaner can rewrite history but requires force-push coordination
   with collaborators.
5. **Add to `.gitignore` and a pre-commit hook** so future leaks
   fail at commit time.

## Sources

- AWS Secrets Manager — https://docs.aws.amazon.com/secretsmanager/
- Vercel env — https://vercel.com/docs/projects/environment-variables
- wrangler secret — https://developers.cloudflare.com/workers/wrangler/commands/#secret
- BFG Repo-Cleaner — https://rtyley.github.io/bfg-repo-cleaner/
