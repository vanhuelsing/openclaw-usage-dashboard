# OpenClaw Usage Dashboard

Interactive HTML dashboard for [OpenClaw](https://github.com/openclaw/openclaw) API usage, costs, and rate-limit quotas across multiple LLM providers.

![Providers](https://img.shields.io/badge/providers-7-blue) ![License](https://img.shields.io/badge/license-MIT-green) ![Dependencies](https://img.shields.io/badge/dependencies-0-brightgreen)

## Features

- **Cost KPIs** — Total spend, today's cost, API calls, cache hit rate, monthly projection
- **Provider & Limits** — Per-provider plan selection with spend tracking against plan limits
  - Auto-detect active providers from usage data
  - Plan dropdown per provider (stored in localStorage)
  - Progress bars per model class with color-coded status (🟢 <70%, 🟡 70-89%, 🔴 ≥90%)
  - Multiple time windows (5h rolling, 7d weekly, monthly, daily)
  - Countdown timers for reset times
  - Direct links to each provider's status page
- **Model config** — Active / default / fallback models from `openclaw.json`
- **Donut chart** — Cost distribution by model (SVG)
- **Hourly sparkline** — Today's call volume
- **Stacked bars** — Cost by period (daily / weekly / monthly / yearly)
- **Model table** — Per-model breakdown with token counts
- **Budget tracker** — Monthly limit with localStorage persistence
- **CSV export** — Download any period view
- **Provider filter** — Dropdown to filter by provider
- **Demo mode** — Generate realistic multi-provider mock data

## Supported Providers

| Provider | Models | Plans |
|---|---|---|
| Anthropic | Claude Opus, Sonnet, Haiku | Pro, Max 5x/20x, Team, API Tier 1-4 |
| OpenAI | GPT-4o, GPT-4o Mini, o-Series | Tier 1-5 |
| Google | Gemini Flash, Gemini Pro | Free, Pay-as-you-go |
| Moonshot | Kimi K2.5 | Standard |
| Mistral | Large, Medium, Small, Codestral | Experiment, Production |
| Cohere | Command-R, Command-R-Plus | Trial, Production |
| Groq | Llama 3.3, Llama 3.1, Gemma2 | Free, Developer |

## Quick Start

```bash
# Real data (reads ~/.openclaw/agents/main/sessions/)
python3 scripts/usage-dashboard-generic.py

# Custom output path
python3 scripts/usage-dashboard-generic.py dashboard.html

# Demo mode
python3 scripts/usage-dashboard-generic.py dashboard.html --demo

# Open in browser
open ~/openclaw-usage-dashboard.html
```

## Requirements

- Python 3.8+ (stdlib only — zero dependencies)
- `ANTHROPIC_API_KEY` in env (optional, for live Anthropic quota gauges)

## Install as OpenClaw Skill

Copy to your OpenClaw skills directory:

```bash
cp -r . ~/.openclaw/workspace/skills/openclaw-usage-dashboard/
```

## Customization

### Add models

Add to the `PRICES` dict in the script (price per 1M tokens):

```python
"provider/model-name": {"input": 2.50, "output": 10.00, "cache_read": 1.25, "cache_write": 2.50}
```

### Configure plan limits

Edit `PROVIDER_REGISTRY` in the script to adjust plan budgets. Consumer plan limits (Pro, Max, Team) are estimates — providers don't always publish exact budgets.

See `references/prices.md` for current pricing across all providers.

## License

MIT
