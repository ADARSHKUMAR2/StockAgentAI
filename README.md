# StockTraderAI

A small simulation where four AI “traders” (Warren, George, Ray, and Cathie) each run with a distinct investing persona. They use the [OpenAI Agents SDK](https://openai.github.io/openai-agents-python/) with MCP tools for accounts, market prices, research (web fetch, search, memory), and optional Pushover notifications. A Gradio dashboard shows portfolios, trades, and live-style logs backed by SQLite.

## What runs where

| Entry point | Role |
|-------------|------|
| `trading_floor.py` | Async scheduler: wakes every **N** minutes (default 60), checks US market status via Polygon when configured, then runs all traders in parallel. Alternates trading vs. rebalancing each cycle. |
| `app.py` | Gradio UI: one column per trader—value, chart, logs, holdings, transactions. Timers refresh data periodically. |
| `reset.py` | Resets the four accounts to default strategies and starting capital ($10,000 each). |

MCP servers (`accounts_server.py`, `market_server.py`, `push_server.py`) are started as subprocesses by the agent runner (`uv run …`) or by `uvx` / `npx` as configured in `mcp_params.py`.

## Prerequisites

- **Python** 3.10+ recommended (project uses stdlib + listed packages in `requirements.txt`).
- **[uv](https://docs.astral.sh/uv/)** — used to run local MCP servers (`uv run …`) and some external MCP packages (`uvx`).
- **Node.js** with **npx** — required for the researcher stack: `@modelcontextprotocol/server-brave-search` and `mcp-memory-libsql` (see `mcp_params.py`).
- API keys as below (`.env` in the project root is loaded via `python-dotenv`).

## Setup

```bash
cd StockTraderAI
python -m venv .venv
source .venv/bin/activate   # Windows: .venv\Scripts\activate
pip install -r requirements.txt
```

Create a `.env` file (see variables below). For first-time use you may want to seed strategies and balances:

```bash
uv run reset.py
```

## Environment variables

| Variable | Required for | Notes |
|----------|----------------|-------|
| `OPENAI_API_KEY` | Default OpenAI models (`gpt-4o-mini`, `gpt-4.1-mini`, etc.) | Used by the Agents SDK for OpenAI-hosted models. |
| `POLYGON_API_KEY` | Real market hours and real prices | If missing, `market.py` falls back to random prices and market-open checks may not reflect reality. |
| `POLYGON_PLAN` | Polygon pricing behavior | Omit or non-`paid`/`realtime`: EOD-style data via local `market_server.py`. `paid` or `realtime`: uses Polygon’s hosted MCP (`uvx` + git install in `mcp_params.py`) and minute-level paths where applicable. |
| `DEEPSEEK_API_KEY` | DeepSeek models | When `USE_MANY_MODELS=true`. |
| `GOOGLE_API_KEY` | Gemini-via-OpenAI-compat | When `USE_MANY_MODELS=true`. |
| `GROK_API_KEY` | xAI Grok | When `USE_MANY_MODELS=true`. |
| `OPENROUTER_API_KEY` | OpenRouter model IDs (names containing `/`) | When `USE_MANY_MODELS=true`. |
| `BRAVE_API_KEY` | Researcher Brave Search MCP | Passed into `npx @modelcontextprotocol/server-brave-search`. |
| `PUSHOVER_USER`, `PUSHOVER_TOKEN` | Push tool in `push_server.py` | Optional; without them pushes may no-op or fail at runtime. |

### Scheduler / demo toggles

| Variable | Default | Meaning |
|----------|---------|---------|
| `RUN_EVERY_N_MINUTES` | `60` | Minutes between trader runs. |
| `RUN_EVEN_WHEN_MARKET_IS_CLOSED` | `false` | Set to `true` to run when Polygon reports the market closed. |
| `USE_MANY_MODELS` | `false` | `false`: all four traders use `gpt-4o-mini`. `true`: distinct models per trader (see `trading_floor.py`). |

## Running

**Trading loop** (keep this process running):

```bash
uv run trading_floor.py
```

**Dashboard**:

```bash
python app.py
# or: uv run app.py
```

**Reset accounts**:

```bash
uv run reset.py
```

## Data on disk

- **`accounts.db`** — SQLite: accounts, append-only logs, cached daily market snapshots (`database.py`).
- **`memory/`** — Per-trader LibSQL files for the researcher memory MCP (`memory/{TraderName}.db`).

## Architecture (short)

- **`traders.py`** — Builds each `Agent` with MCP stdio servers (accounts, push, market; researcher gets fetch + Brave + memory).
- **`accounts.py` / `accounts_server.py`** — Pydantic account model, buy/sell, persisted in SQLite.
- **`market.py`** — Polygon REST client for status and prices; optional fallback random prices.
- **`tracers.py`** — Custom trace processor writing agent traces into the log table for the UI.

This project is for experimentation and learning, not financial advice. Model outputs and simulated trades are not a substitute for professional guidance.
