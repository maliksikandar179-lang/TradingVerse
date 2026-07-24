# TradingVerse

A self-hosted quantitative research and signal-delivery platform for crypto and FX markets.

**71 orchestrated services** running on a single host under macOS `launchd` — 15 always-on daemons, 27 interval jobs and 29 calendar-scheduled jobs. Python 3.13, with an MQL5 Expert Advisor on the execution side.

> **About this repository.** This is an architecture and design overview. The implementation is private — no strategy logic, signal rules, model parameters or credentials are published here. The purpose is to document how the system is built and why.

---

## What it does

1. **Ingests** real-time market microstructure from crypto and FX venues — bars, trade tape, L2 order-book depth, market-by-order, liquidations, open interest.
2. **Stores** it across ~22 local SQLite databases.
3. **Researches** it — signal engines and an autonomous strategy-validation pipeline run over the data on staggered schedules.
4. **Validates** — Monte Carlo risk simulation, signal outcome tracking, shadow-mode pilots for anything unproven.
5. **Delivers** — a Discord gateway bot with subscription and role management, plus a live animated dashboard.

---

## Architecture

```
                    ┌─────────────────────────────────────────┐
                    │  launchd  (71 services, KeepAlive)      │
                    └────────────────────┬────────────────────┘
                                         │
        ┌────────────────────────────────┼────────────────────────────────┐
        │                                │                                │
┌───────▼────────┐              ┌────────▼────────┐             ┌─────────▼────────┐
│  COLLECTORS    │              │    RESEARCH     │             │    DELIVERY      │
│  15 daemons    │              │  interval jobs  │             │                  │
│                │              │                 │             │                  │
│ • bars / WS    │─── SQLite ──▶│ • signal engines│────────────▶│ • Discord bot    │
│ • trade tape   │   (~22 DBs)  │ • regime detect │             │   18 commands    │
│ • L2 depth     │              │ • research agent│             │   8 channels     │
│ • MBO          │              │ • backtests     │             │ • subscriptions  │
│ • liquidations │              │ • Monte Carlo   │             │ • live dashboard │
│ • open interest│              │ • outcome track │             │                  │
└────────────────┘              └────────┬────────┘             └──────────────────┘
                                         │
                                ┌────────▼────────┐
                                │ EXECUTION BRIDGE│
                                │   ✗ DISABLED    │
                                └─────────────────┘

        ┌──────────────────────────────────────────────────────────┐
        │  RELIABILITY LAYER — watchdogs, DNS fallback, re-auth,   │
        │  429 backoff, stale-price gating, log rotation, pruning  │
        └──────────────────────────────────────────────────────────┘
```

---

## Components

| Layer | Count | What it does |
|---|---|---|
| **Always-on daemons** | 15 | Real-time collectors (crypto bars, FX pairs, gold, trade tape, L2 depth, MBO, liquidations), Discord gateway, dashboard server, hedge monitor |
| **Interval jobs** | 27 | Signal engines, health watchdogs, news pipeline, scan-and-broadcast cycle, research agents — from 30s to 3-day cadences |
| **Scheduled jobs** | 29 | Nightly maintenance, log rotation, cache pruning, Monte Carlo risk, outcome tracking, market-hours notifications, dashboard rebuild |

### Data collection
Bars and tick data via WebSocket listeners; trade tape, L2 order-book depth and market-by-order collectors; liquidation and open-interest feeds; COT positioning and gamma-wall snapshots. Venues: Bybit, Binance, Capital.com, cTrader, Alpaca.

### Research
Momentum (volatility-scaled TSMOM), regime detection, market-structure shift, absorption entry, dip engine, order-flow analysis and support/resistance fade — each on its own interval. An autonomous research agent and a strategy validation pipeline run unattended on three-day cycles.

### Risk and validation
Weekly Monte Carlo risk simulation. Signal outcome tracking against realised results. Take-profit recomputation per pair and confidence tier. Unproven strategies run in shadow mode — logged and reviewed, never broadcast.

### Reliability
Health watchdogs at 30s, 300s and 900s intervals. Signal-drought alarms. DNS resilience with fallback resolution. Exchange re-authentication with 429 rate-limit backoff. Stale-price gating so nothing trades on dead data. Nightly maintenance, log rotation and cache pruning. `launchd` `KeepAlive` restarts any daemon that exits.

### Delivery
A `discord.py` gateway bot (~1,650 lines) with 18 slash commands, auto-posting across 8 channels, and full subscription lifecycle handling — member onboarding, premium role assignment, expiry tracking. A live dashboard refreshed every 60 seconds with animated component status, served over local HTTP.

---

## Execution is deliberately disabled

The broker execution bridge is written and tested. It filters high-conviction signals, computes position size at 1% account risk, and submits bracket orders (entry, stop, take-profit) to Bybit for crypto and Capital.com for FX and gold.

**It is switched off, on purpose.**

Three reasons:

1. The system is at research and paper-trading stage. There is no validated live edge yet.
2. Governance: real-money execution requires explicit approval, not a config flag someone forgets about.
3. On testnet, computed order sizes fell below exchange minimum notionals — a real constraint, not a bug to paper over.

The `launchd` job is set to `RunAtLoad=false` and is not loaded. It cannot fire.

Writing execution code is easy. Deciding not to run it is the part that matters.

---

## Tech stack

| | |
|---|---|
| **Language** | Python 3.13, MQL5, Bash |
| **Data** | pandas, numpy, scipy, SQLite |
| **ML** | scikit-learn, LightGBM; PyTorch + Transformers (experimental) |
| **Async / networking** | asyncio, aiohttp, websockets, exchange REST and WebSocket APIs |
| **Discord** | discord.py 2.7 |
| **Visualisation** | matplotlib, custom HTML/CSS dashboards |
| **Orchestration** | macOS `launchd` |

---

## Known limitations

Documented deliberately — these are real, and they are the roadmap.

- **Single host.** Everything runs on one machine. No redundancy, no failover. The host is a single point of failure.
- **No containerisation.** Services run directly under `launchd`. Portability is limited; moving hosts means reconfiguring.
- **No CI/CD.** Changes are deployed manually. No automated test gate before a service restarts.
- **Local storage.** ~22 SQLite databases on local disk. Fine at current volume, will not scale horizontally.
- **macOS-specific.** `launchd` orchestration ties the system to macOS. A Linux move means reworking the service layer.

Next steps: containerise the collectors, move to a cloud host, replace `launchd` with a portable scheduler, and add a test gate before deploys.

---

## Security

- No credentials, keys or tokens in this repository or in the implementation repo.
- All secrets loaded from environment variables; `.env.example` contains variable names only.
- Local databases and logs are untracked.

---

## Contact

**Malik Sikandar Sohail** — Rawalpindi, Pakistan
[LinkedIn](https://www.linkedin.com/in/malik-sikandar-400baa144) · maliksikandar179@gmail.com

*All published market content is educational and reflects personal analysis. Nothing constitutes financial advice.*
