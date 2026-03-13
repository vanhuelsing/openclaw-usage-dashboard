# OpenClaw Usage Dashboard

Interactive HTML dashboard for [OpenClaw](https://github.com/openclaw/openclaw) API usage, costs, and rate-limit quotas across multiple LLM providers.

![Dashboard Preview](https://img.shields.io/badge/providers-7-blue) ![License](https://img.shields.io/badge/license-MIT-green) ![Dependencies](https://img.shields.io/badge/dependencies-0-brightgreen)

## Features

- **Cost KPIs** — Total spend, today's cost, API calls, cache hit rate, monthly projection
- **Model config** — Active / default / fallback models from `openclaw.json`
- **Anthropic quota gauges** — 5h and 7d rate limit utilization
- **Donut chart** — Cost distribution by model (SVG)
- **Hourly sparkline** — Today's call volume
- **Stacked bars** — Cost by period (daily / weekly / monthly / yearly)
- **Model table** — Per-model breakdown with token counts
- **Budget tracker** — Monthly limit with localStorage persistence
- **CSV export** — Download any period view
- **Provider filter** — Dropdown to filter by provider
- **Demo mode** — Generate realistic mock data for testing

## Supported Providers

| Provider | Models |
|---|---|
| Anthropic | Claude Opus, Sonnet, Haiku |
| OpenAI | GPT-4o, GPT-4o-mini, o1, o3-mini |
| Google | Gemini 2.0 Flash/Pro, Gemini 1.5 |
| Moonshot | Kimi K2.5 |
| Mistral | Large, Medium, Small, Codestral |
| Cohere | Command-R, Command-R-Plus |
| Groq | Llama 3.3, Llama 3.1, Gemma2 |

## Quick Start

```bash
# Real data (reads ~/.openclaw/agents/main/sessions/)
python3 scripts/usage-dashboard-generic.py

# Demo mode
python3 scripts/usage-dashboard-generic.py dashboard.html --demo

# Open in browser
open ~/openclaw-usage-dashboard.html
```

## Requirements

- Python 3.8+ (stdlib only — zero dependencies)
- `ANTHROPIC_API_KEY` in env (optional, for live quota gauges)

## Install as OpenClaw Skill

```bash
# Copy to your skills directory
cp -r . ~/.openclaw/workspace/skills/openclaw-usage-dashboard/
```

Or install the `.skill` package directly in OpenClaw.

## Customization

Add models to the `PRICES` dict in the script (price per 1M tokens):

```python
"provider/model-name": {"input": 2.50, "output": 10.00, "cache_read": 1.25, "cache_write": 2.50}
```

See `references/prices.md` for current pricing across all providers.

## License

MIT
