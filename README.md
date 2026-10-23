# TradingVerse

A self-hosted quantitative research and signal-delivery platform for **crypto, gold and FX**.

**134 orchestrated services** on a single host under macOS `launchd` — 17 always-on daemons, 63 interval jobs and 54 calendar-scheduled jobs. ~126,000 lines of Python 3.13 across ~710 modules, 43 local SQLite databases, 105 automated guard tests, and an MQL5 Expert Advisor on the execution side.

> **About this repository.** This is an architecture and design overview. The implementation is private — no strategy logic, signal rules, model parameters, performance figures or credentials are published here. The purpose is to document how the system is built, how changes are decided, and why.
>
> *Last updated: October 2026.*

---

## What it does

1. **Ingests** real-time market data and microstructure — bars, trade tape, L2 order-book depth, market-by-order, liquidations, open interest, funding, macro and rates context.
2. **Stores** it across 43 local SQLite databases.
3. **Researches** it — signal engines, regime detection and an autonomous strategy-validation pipeline run on staggered schedules.
4. **Validates** every idea against history before it can reach a member — and keeps measuring it after.
5. **Delivers** signals and trade-management updates to a paid Discord community, seconds after a candle closes.

**Markets:** 9 crypto majors, gold, and 3 FX majors (2 more FX pairs in paper tracking).
**Timeframes:** 5-minute gold scalps through 15m, 1H, 4H and daily signals.

---

## Architecture

```
                    ┌─────────────────────────────────────────┐
                    │  launchd  (134 services, KeepAlive)     │
                    └────────────────────┬────────────────────┘
                                         │
        ┌────────────────────────────────┼────────────────────────────────┐
        │                                │                                │
┌───────▼────────┐              ┌────────▼────────┐             ┌─────────▼────────┐
│  COLLECTORS    │              │    RESEARCH     │             │    DELIVERY      │
│  17 daemons    │              │  interval jobs  │             │                  │
│                │              │                 │             │                  │
│ • bars / WS    │─── SQLite ──▶│ • signal engines│────────────▶│ • real-time      │
│ • trade tape   │   (43 DBs)   │ • regime detect │             │   bar-close cards│
│ • L2 depth     │              │ • research agent│             │ • trade updates  │
│ • MBO          │              │ • backtests     │             │ • Discord bot    │
│ • liquidations │              │ • Monte Carlo   │             │   19 commands    │
│ • open interest│              │ • outcome track │             │ • subscriptions  │
│ • macro / rates│              │ • shadow books  │             │ • live dashboard │
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
        ┌──────────────────────────────────────────────────────────┐
        │  CHANGE CONTROL — kill flags, fail-open hooks, guard     │
        │  tests, self-retiring pilots, admin approval for live    │
        └──────────────────────────────────────────────────────────┘
```

---

## Components

| Layer | Count | What it does |
|---|---|---|
| **Always-on daemons** | 17 | Real-time collectors (crypto, FX, gold, trade tape, L2 depth, MBO, liquidations), real-time bar-close signal listeners, Discord gateway, dashboard server, hedge monitor |
| **Interval jobs** | 63 | Signal engines, the gold scalp publisher, running-trade health checks, health watchdogs, news pipeline, scan-and-broadcast cycle, research agents — from 60 s to 3-day cadences |
| **Scheduled jobs** | 54 | Nightly maintenance, log rotation, cache pruning, Monte Carlo risk, outcome tracking, market-hours notifications, dashboard rebuild |

### Data collection
WebSocket bar listeners; trade tape, L2 order-book depth and market-by-order collectors; liquidation, open-interest and funding feeds; COT positioning and gamma-wall snapshots; macro and rates context for FX; a live broker-price check so gold cards match the price members actually see. Venues and sources include Bybit, Binance, Capital.com, cTrader and Alpaca.

### Research
Momentum, regime detection, market-structure, absorption, order-flow, support/resistance, range and liquidity studies — each on its own schedule. Gold has its own intraday desk, including a family of 5-minute scalp setups published under member-facing names. An autonomous research agent and a strategy-validation pipeline run unattended.

### Delivery
- **Real-time cards** posted within seconds of a candle closing, each with a daily signal ID, entry, stop, target and a management plan.
- **Trade management updates** — when a new signal points the same way as one already running, members get a short update first (how the running trade is doing, and whether to add or not). When a running trade stalls or starts to turn before its target, a **precaution card** reminds members to protect it.
- **Context labels** on cards — market regime, order-book confirmation, counter-trend flags — so members know what kind of trade they are looking at, without strategy names being revealed.
- A `discord.py` gateway bot (~2,270 lines, 19 slash commands) with full subscription lifecycle handling — onboarding, premium roles, expiry tracking — across 10+ channels, plus a profit-history archive and a live dashboard refreshed every 60 seconds.

---

## How changes are decided

The research layer is only as good as the rules about what is allowed to go live. These are the rules.

1. **History first, not opinions.** Every proposed change is re-traded on the actual historical candles — the same entries, stops and exits members would have had — and scored by one standard rule shared by every report, card and review.
2. **Robustness, not a single number.** A result has to hold in both halves of the history, survive a bootstrap confidence interval, beat a random-entry null, and — where it applies — hold pair by pair (leave-one-out). When many variants are tried at once, false-discovery control (Benjamini–Hochberg) is applied.
3. **Confluence over isolation.** New ideas are tested *together with* the existing setups they would join, not only on their own.
4. **Every live change has an off switch.** Each feature ships behind a kill flag, fails open (an error leaves the signal exactly as it was built), and is covered by a guard test.
5. **Pilots retire themselves.** New setup types carry a self-retire rule: if their live record turns negative after a set number of trades, they switch off and alert the admin.
6. **Admin approval for anything members see.** No change to live signals ships without an explicit decision from the admin.
7. **Look back honestly.** When results dip, every change from the period is re-traded against the version before it. In October 2026 this found that one cross-timeframe rule was costing results; it was switched to information-only the same day, and an unrelated setup type that had been added the same week was removed from the code entirely.

---

## Execution is deliberately disabled

The broker execution bridge is written and tested. It filters high-conviction signals, computes position size at a fixed fraction of account risk, and submits bracket orders (entry, stop, take-profit) to Bybit for crypto and Capital.com for FX and gold.

**It is switched off, on purpose.**

1. Signals are delivered for members to trade at their own discretion. The system tracks every outcome; it does not trade them.
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
| **Source control** | Git / GitHub as the single source of truth for every live change |

---

## Known limitations

Documented deliberately — these are real, and they are the roadmap.

- **Single host.** Everything runs on one machine. No redundancy, no failover. The host is a single point of failure.
- **No containerisation.** Services run directly under `launchd`. Portability is limited; moving hosts means reconfiguring.
- **Tests, but no CI gate.** 105 guard tests exist and are run before changes, but nothing yet blocks a deploy automatically.
- **Local storage.** 43 SQLite databases on local disk. Fine at current volume, will not scale horizontally.
- **macOS-specific.** `launchd` orchestration ties the system to macOS. A Linux move means reworking the service layer.

Next steps: containerise the collectors, move to a cloud host, replace `launchd` with a portable scheduler, and turn the guard tests into an automatic gate before deploys.

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
