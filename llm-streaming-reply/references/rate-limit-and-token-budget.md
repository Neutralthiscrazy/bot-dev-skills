---
name: rate-limit-and-token-budget
description: >-
  Companion digest for managing LLM and chat-platform rate limits in
  a bot. Tokens-per-second targets, prompt caching math, user-side
  rate limit / flood control in chat APIs, and warning thresholds
  before rate limits hit.
version: 0.1.0
user-invocable: false
author: neutralthiscrazy
license: MIT
---

# Rate limits and token budget — bot-side guard rail

## When to load

- Bot uses paid LLM API → cost and rate-limit matters
- Multiple concurrent users in chat → user-side rate-limit needed
- Telegram / Discord / Slack rate-limit response (429 / 5xx) → need
  to know how to back off

## LLM side

### OpenAI rate limits

OpenAI enforces token-per-minute (TPM) and request-per-minute (RPM)
limits per organization and per model. Specific numbers depend on
usage tier (free vs tier 1-4). For chat bots:

- `gpt-4o-mini`: high RPM/TPM, cheap.
- `gpt-4`: lower RPM/TPM, expensive.
- Self-tiers in dashboard — https://platform.openai.com/settings/organization/limits

If a stream is expected to use 2000 input + 2000 output tokens, and
RPM is 60, you can fit ~30 such requests per minute.

### Anthropic rate limits

Per `ITPM` (input tokens per minute) and `OTPM` (output tokens per
minute). Generally lower than OpenAI on equivalent models. Cache
hits don't count against OTPM, only ITPM for the new portion.

### Three-tier avoidance

1. **Per-user rate limit** — max N requests per minute per user.
   Prevents one user flooding.
2. **Per-bot token budget** — running counter of input + output tokens
   consumed. Once a daily budget is reached, return a friendly
   "try tomorrow" message instead of calling the API.
3. **Backoff on 429** — if LLM returns 429, exponential back off.

```python
import time

async def call_llm_with_backoff(prompt):
    for attempt in range(3):
        try:
            return await llm_call(prompt)
        except RateLimitError as e:
            await asyncio.sleep(2 ** attempt)
    raise  # give up after 3
```

## Chat-side (Telegram)

Telegram overall limits:
- 30 messages per second per bot overall
- 20 messages per minute per group

The group limit is the painful one. If your bot is in 1000 groups
with active users, your bot can saturate quickly.

Mitigation:
- Cache the LLM answer keyed on user question similarity (embedding
  match) — different users asking similar questions share response.
- Per-message debounce in groups (don't auto-reply to every message
  in a busy group; require explicit `@botname` or `/command`).

## Chat-side (Discord)

Discord limits:
- 50 messages per second global per bot
- 5 messages per 5 seconds per channel per bot
- Webhook (no global rate limit, but channel still capped)

`interaction.reply` counts toward the channel rate limit. Edit
spam (`editReply` 5/sec/channel) is technically allowed but produces
visible flicker.

Mitigation: `buffer + 1s flush` pattern as in the platform reference.

## Token budget math

L1 cost-aware bot:
- Daily budget: 100k tokens @ $0.005/1k = $5/day cap
- Per-user budget: 4k tokens (request) + 2k tokens (response) per hour
- If exceeded → friendly message + log

Track with simple in-memory counter:

```python
DAILY_BUDGET_TOKENS = 100_000
PER_USER_TPM = 6_000  # 4k input + 2k output

daily_used = 0
user_window = defaultdict(list)  # user_id -> [(timestamp, tokens)]

def check_budget(user_id, tokens_to_use):
    global daily_used
    if daily_used + tokens_to_use > DAILY_BUDGET_TOKENS:
        return False
    now = time.monotonic()
    user_window[user_id] = [
        (t, tk) for (t, tk) in user_window[user_id] if now - t < 3600
    ]
    used = sum(tk for _, tk in user_window[user_id])
    if used + tokens_to_use > PER_USER_TPM:
        return False
    user_window[user_id].append((now, tokens_to_use))
    daily_used += tokens_to_use
    return True
```

## Pitfalls

- Backoff doesn't reset across process restarts → use day-bucket
  persistent storage (SQLite, Redis).
- Discord's 429 has a `retry_after` field — always read it, don't
  guess. Telegram's flood-wait also provides `retry_after` seconds.
- API quotas reset at midnight UTC. Plan accordingly.
- Caching aggressively without invalidation: same answer to all
  "what's the weather" questions is wrong in some regions.

## Sources

- OpenAI rate limits — https://platform.openai.com/docs/guides/rate-limits
- Anthropic rate limits — https://docs.anthropic.com/en/api/rate-limits
- Telegram bot API limits — https://core.telegram.org/bots/faq#how-do-i-avoid-rate-limits-in-groups
- Discord rate limits — https://discord.com/developers/docs/topics/rate-limits
