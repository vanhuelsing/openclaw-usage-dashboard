---
name: openclaw-usage-dashboard
description: Generate an interactive local HTML dashboard showing OpenClaw API usage, costs, and rate-limit quotas across multiple LLM providers (Anthropic, OpenAI, Google, Moonshot, Mistral, Cohere, Groq). Reads session transcripts directly — no external service needed. Use when a user asks about API costs, token usage, model spend, quota/rate limits, usage by model or time period, or wants a cost overview dashboard. Triggers on "usage dashboard", "API kosten", "wie viel kostet", "kontingent", "rate limit", "token verbrauch", "cost dashboard", "openclaw costs", "spending overview".
---

# OpenClaw Usage Dashboard

Generate a self-contained HTML dashboard from OpenClaw session transcripts. Zero dependencies — Python 3.8+ stdlib only.

## Usage

```bash
# Real data (reads ~/.openclaw/agents/main/sessions/)
python3 scripts/usage-dashboard-generic.py [output.html]

# Demo mode (generates realistic multi-provider mock data)
python3 scripts/usage-dashboard-generic.py [output.html] --demo
```

Default output: `~/openclaw-usage-dashboard.html`

Override sessions path: set `OPENCLAW_ROOT` env var.

## What It Shows

- **Cost KPIs**: Total spend, today's cost, API calls, active model, cache hit rate, monthly projection
- **Model config**: Active / default / fallback models from `openclaw.json`
- **Provider & Limits**: Per-provider plan selection with usage tracking against plan limits
  - Toggle active providers (auto-detected from usage data)
  - Plan dropdown per provider (Pro, Max, Team, API Tiers, Custom, etc.)
  - Spend progress bars per model class with color-coded status
  - Multiple time windows (5h, 7d, monthly, daily) with countdown timers
  - Direct links to each provider's status/limits page
  - All settings persisted in localStorage
- **Anthropic quota gauges**: Live 5h and 7d rate limit utilization (when API key supports it)
- **Donut chart**: Cost distribution by model
- **Hourly sparkline**: Today's call volume by hour
- **Stacked bars**: Cost by period (daily / weekly / monthly / yearly)
- **Model table**: Per-model breakdown with token counts and cost
- **Detail table + chart**: Toggle between table and bar chart view per period
- **Budget tracker**: Set monthly limit, progress bar, localStorage persistence
- **CSV export**: Download any period view as CSV
- **Provider dropdown**: Filter by provider

## Supported Providers

| Provider | Models | Plans |
|---|---|---|
| Anthropic | Opus, Sonnet, Haiku | Pro, Max 5x/20x, Team, API Tier 1-4 |
| OpenAI | GPT-4o, GPT-4o Mini, o-Series | Tier 1-5 |
| Google (Gemini) | Flash, Pro | Free, Pay-as-you-go |
| Moonshot/Kimi | Kimi K2.5 | Standard |
| Mistral | Large, Medium, Small, Codestral | Experiment, Production |
| Cohere | Command-R, Command-R-Plus | Trial, Production |
| Groq | Llama, Gemma | Free, Developer |

## Customization

### Add models

Edit `PRICES` dict in the script (price per 1M tokens):

```python
"provider/model-name": {"input": 2.50, "output": 10.00, "cache_read": 1.25, "cache_write": 2.50}
```

Add to `MODEL_COLORS` and `ALIAS_MAP` as needed.

### Configure plan limits

Edit `PROVIDER_REGISTRY` in the script to adjust plan budgets per provider. Consumer plan limits (Pro, Max, Team) are estimates — Anthropic doesn't publish exact budgets.

### Pricing reference

See `references/prices.md` for current prices across all providers.

## Requirements

- Python 3.8+ (stdlib only)
- `ANTHROPIC_API_KEY` in env (optional, for live quota gauges)
