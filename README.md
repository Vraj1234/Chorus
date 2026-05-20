# Chorus

**AI-powered correlation discovery, visualization, and backtesting for prediction markets.**

Chorus turns Polymarket's live order book into a navigable map of which markets move together — and lets you test trading strategies built on those relationships before risking capital.

---

## Overview

Prediction markets contain thousands of overlapping bets. When a question about a U.S. election shifts, related markets on geopolitics, commodities, and economic outcomes often shift in lockstep. Chorus surfaces those relationships in three views:

- **Network** — a live force-directed graph of every Polymarket market over $50k volume, with edges drawn between markets whose log-returns are statistically correlated.
- **Discover** — pick any market and let an LLM-guided search find its semantic *followers*: markets that have historically moved with it, ranked by correlation and inefficiency.
- **Backtest** — pick a resolved market, replay history, and measure how a correlation-following strategy would have performed at a given threshold and holding period.

Live deployment runs on Fly.io and refreshes data on demand.

---

## Features

### Live network visualization
- Force-directed D3 graph of correlated markets across Politics, Sports, Finance, Crypto, Geopolitics, Earnings, Tech, Culture, World, Economy, Elections, and Mentions.
- Markets categorized via `gpt-4o-mini`.
- Edges weighted by Pearson correlation of **log returns** (not raw prices) to avoid spurious trend correlation. See [`docs/CORRELATION.md`](docs/CORRELATION.md) for the full methodology and changelog.
- Volume-filtered (>$50k) to suppress low-liquidity noise.

### Semantic follower discovery
- Given a target market, an OpenAI-driven worker proposes candidate markets, fetches CLOB price history, computes timestamp-aligned correlations, and streams results to the client over NDJSON.
- Keepalive pings every 15s keep long-running streams alive behind Fly.io / browser idle timers.

### Strategy backtester
- Searches resolved-to-Yes markets from Polymarket's Gamma API (2023+ with CLOB data, $10k+ volume).
- Replays the price history of the chosen anchor market and its correlated followers.
- Reports realized P&L under a configurable correlation threshold and holding period (1d / 7d / 30d).

---

## Architecture

```
┌──────────────────┐         ┌─────────────────────────────┐
│   Browser UI     │ ──────► │      Python HTTP server     │
│  (D3 + Tailwind) │ ◄────── │      (server.py)            │
└──────────────────┘ NDJSON  └─────────────────────────────┘
                                  │           │           │
                          ┌───────┘           │           └───────┐
                          ▼                   ▼                   ▼
                    ┌──────────┐      ┌──────────────┐     ┌──────────────┐
                    │ Polymarket│      │   OpenAI    │     │  Supabase    │
                    │ Gamma+CLOB│      │ gpt-4o-mini │     │  (Postgres)  │
                    └──────────┘      └──────────────┘     └──────────────┘
```

| Component             | Purpose                                                                  |
| --------------------- | ------------------------------------------------------------------------ |
| `server.py`           | Stdlib HTTP server, REST + NDJSON streaming endpoints, CORS, static files |
| `data_worker.py`      | Periodic fetch of markets + price history from Polymarket APIs           |
| `discover_worker.py`  | Semantic follower discovery (LLM candidate generation + correlation)     |
| `backtest_worker.py`  | Historical replay and P&L calculation                                    |
| `market_refresh_job.py` | Long-running scheduled refresh job                                     |
| `database.py`         | Supabase / Postgres data layer                                           |
| `scheduler.py`        | Cron-style job scheduler                                                 |
| `analyze_api.py`      | Ad-hoc analysis helpers                                                  |
| `index.html`          | Network visualization                                                    |
| `discover.html`       | Semantic discovery UI                                                    |
| `backtest.html`       | Backtester UI                                                            |
| `methodology.html`    | Plain-English explanation of the correlation algorithm                   |

---

## Tech stack

- **Backend**: Python 3.11, stdlib `http.server` (no framework), `requests`, `openai`, `python-dotenv`, `schedule`
- **Frontend**: Vanilla JS modules, [D3.js v7](https://d3js.org/), [Tailwind CSS](https://tailwindcss.com/) (CDN)
- **Data sources**: [Polymarket Gamma API](https://gamma-api.polymarket.com/) (market metadata), [Polymarket CLOB API](https://clob.polymarket.com/) (price history)
- **AI**: OpenAI `gpt-4o-mini` for market classification and semantic candidate generation
- **Storage**: Supabase (Postgres) — see [`supabase/migrations/`](supabase/migrations/)
- **Deployment**: Fly.io, Docker, GitHub Actions ([`.github/workflows/fly-deploy.yml`](.github/workflows/fly-deploy.yml))

---

## Getting started

### Prerequisites

- Python 3.11+
- An OpenAI API key
- (Optional) Supabase project for persistent storage

### 1. Clone and install

```bash
git clone https://github.com/Vraj1234/Chorus.git
cd Chorus
pip install -r requirements.txt
```

### 2. Configure environment

```bash
cp .env.example .env
# then edit .env:
#   OPENAI_API_KEY=sk-...
```

### 3. Run the server

```bash
python server.py
```

The app is now live at [http://localhost:8000](http://localhost:8000).

Routes:

| Path                | View                                  |
| ------------------- | ------------------------------------- |
| `/`                 | Network correlation graph             |
| `/discover.html`    | Semantic follower discovery           |
| `/backtest.html`    | Historical strategy backtester        |
| `/methodology.html` | How correlations are calculated       |

---

## API

The server exposes a small REST + NDJSON streaming API.

| Method | Endpoint                  | Description                                                  |
| ------ | ------------------------- | ------------------------------------------------------------ |
| GET    | `/api/data`               | Complete dataset (markets + correlations) for the network    |
| GET    | `/api/data/markets`       | Markets only                                                 |
| GET    | `/api/data/correlations`  | Correlation edges only                                       |
| GET    | `/api/data/status`        | Last refresh time, totals, readiness                         |
| GET    | `/api/backtest/search?q=` | Search resolved markets by question text                     |
| POST   | `/api/classify`           | Classify a market into a category via `gpt-4o-mini`          |
| POST   | `/api/refresh`            | Manually trigger a data refresh                              |
| POST   | `/api/discover`           | Stream NDJSON discovery events for a target market           |
| POST   | `/api/backtest`           | Stream NDJSON backtest events for a resolved market          |
| GET    | `/api/gamma/*`            | Pass-through proxy to Polymarket Gamma API (CORS)            |
| GET    | `/api/clob/*`             | Pass-through proxy to Polymarket CLOB API (CORS)             |

Long-running endpoints (`/api/discover`, `/api/backtest`) stream `application/x-ndjson` and emit a `{"type":"keepalive"}` event every 15 seconds to prevent idle-connection timeouts.

---

## Deployment (Fly.io)

The repo ships with a working [`Dockerfile`](Dockerfile) and [`fly.toml`](fly.toml).

```bash
fly launch          # one-time
fly secrets set OPENAI_API_KEY=sk-...
fly deploy
```

Fly config highlights:

- Persistent volume `polymarket_data` mounted at `/app/data`
- Auto-start machines, `min_machines_running = 1` to keep workers warm
- `REFRESH_INTERVAL=600` (10 min) — data refreshes lazily on user requests

CI/CD: pushes to `main` deploy automatically via [`.github/workflows/fly-deploy.yml`](.github/workflows/fly-deploy.yml).

---

## Methodology

The short version: edges are drawn when two markets' **log returns** show a Pearson correlation above a threshold, with timestamp-aligned data points and a minimum-variance filter to reject stagnant markets.

For the full algorithm, including version history, known limitations, and the API contract for CLOB price history, see [`docs/CORRELATION.md`](docs/CORRELATION.md) — or visit `/methodology.html` in the running app.

---

## Project structure

```
.
├── server.py                # HTTP server + REST/NDJSON endpoints
├── data_worker.py           # Polymarket data fetcher
├── discover_worker.py       # LLM-driven follower discovery
├── backtest_worker.py       # Historical strategy replay
├── market_refresh_job.py    # Scheduled refresh job
├── scheduler.py             # Cron-style scheduling
├── database.py              # Supabase / Postgres data access
├── analyze_api.py           # Analysis helpers
├── index.html               # Network visualization
├── discover.html            # Discovery UI
├── backtest.html            # Backtester UI
├── methodology.html         # Algorithm write-up
├── css/                     # Styles
├── js/                      # Frontend modules (D3 graph, math, API, UI)
├── docs/                    # Algorithm documentation
├── supabase/migrations/     # Database schema
├── .github/workflows/       # CI/CD (Fly.io deploy)
├── Dockerfile
├── fly.toml
└── requirements.txt
```

---

## Roadmap

Known gaps tracked in [`docs/CORRELATION.md`](docs/CORRELATION.md):

- Spearman / mutual-information correlation for non-linear relationships
- Same-event grouping detection (sibling markets that correlate mechanically)
- Lagged cross-correlation for lead/follow pairs
- Resolution-date exclusion windows
- Higher-resolution (hourly) price data

---

## License

No license has been declared yet. Until one is added, all rights are reserved by the author.
