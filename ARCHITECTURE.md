# Kalshi Edge — Architecture & Design

A self-hosted trading dashboard for Kalshi sports/World Cup markets. It pulls markets,
estimates true probabilities, computes edge / EV / Kelly, ranks bets, enforces risk limits,
and places orders **only after manual approval**. Demo (paper) mode is the default.

## Design reality (read once)
The profitability of this system lives entirely in the probability-estimation module
(`app/research` + `app/quant/probability`). Kalshi's marquee sports markets are deep and
sharply priced, so a positive edge is rare. The platform is therefore built so that
**`NO BET` is the normal, expected output**. Its durable value is execution discipline,
risk control, position tracking, and an auditable log — not generating alpha. The stake
engine sizes off *edge*, never off raw win-probability, so a 94%-to-win favorite with no
edge is sized at $0, while a genuine +10% edge on a 30% outcome is sized up.

---

## 1. Full architecture

```
                         ┌──────────────────────────────┐
                         │  Next.js + Tailwind frontend │  (read-only view + Approve button)
                         │  dashboard / positions / log │
                         └───────────────┬──────────────┘
                                         │  HTTPS (session cookie auth)
                                         ▼
        ┌────────────────────────────────────────────────────────────┐
        │                  FastAPI backend (Python)                   │
        │                                                            │
        │  /api/markets     /api/recommendations    /api/positions    │
        │  /api/orders      /api/bankroll           /api/log          │
        │                                                            │
        │  ┌──────────┐  ┌───────────┐  ┌─────────┐  ┌────────────┐   │
        │  │ kalshi   │  │ research  │  │ quant   │  │ risk       │   │
        │  │ client   │  │ adapters  │  │ engine  │  │ manager    │   │
        │  └────┬─────┘  └─────┬─────┘  └────┬────┘  └─────┬──────┘   │
        └───────┼──────────────┼─────────────┼─────────────┼─────────┘
                │              │             │             │
                ▼              ▼             ▼             ▼
        Kalshi REST/WS   Odds/stats APIs  pure-python   PostgreSQL
        (demo|prod)      (pluggable)      math           (state + log)
                ▲
                │  Celery beat / cron
        ┌───────┴────────────────────────────────────────────────┐
        │ jobs: refresh markets, recompute edges, T-90 reminders, │
        │       price-move alerts, settle results, P/L rollup     │
        └────────────────────────────────────────────────────────┘
```

Data flow per cycle: scheduler → pull chosen markets via Kalshi client → research adapters
fetch sharp odds + stats → quant engine de-vigs, estimates true prob, computes edge/EV/Kelly →
risk manager clamps stake against limits → recommendation written to DB → dashboard renders →
user clicks **Approve** → backend re-validates risk → order sent to Kalshi (paper or live) →
result logged → settlement job reconciles P/L.

---

## 2. Database schema (PostgreSQL)

```sql
-- accounts / auth
users(id PK, email UNIQUE, password_hash, created_at)

-- environment + bankroll config (one active row)
settings(id PK, env TEXT CHECK(env IN ('demo','prod')) DEFAULT 'demo',
         bankroll_cents BIGINT, kelly_fraction NUMERIC DEFAULT 0.25,
         min_tier_to_bet TEXT DEFAULT 'C',
         max_stake_per_market_cents BIGINT DEFAULT 10000,
         max_daily_loss_cents BIGINT DEFAULT 15000,
         stop_loss_drawdown_pct NUMERIC DEFAULT 0.20,
         updated_at)

-- snapshot of a Kalshi market we track
markets(id PK, ticker TEXT UNIQUE, event_ticker, series_ticker,
        title, sport TEXT, kickoff_ts, status,
        yes_price_cents INT, no_price_cents INT,
        implied_prob NUMERIC, volume_cents BIGINT, open_interest BIGINT,
        liquidity_cents BIGINT, last_seen_ts)

-- order-book snapshots (for liquidity + movement)
orderbook_snaps(id PK, market_id FK, ts, yes_bids JSONB, yes_asks JSONB,
                spread_cents INT, top_liquidity_cents BIGINT)

-- research output per market/outcome
estimates(id PK, market_id FK, outcome TEXT, true_prob NUMERIC,
          model_inputs JSONB, sharp_prob NUMERIC, source_notes TEXT, ts)

-- a ranked recommendation
recommendations(id PK, market_id FK, outcome TEXT,
                market_prob NUMERIC, true_prob NUMERIC,
                edge NUMERIC, ev_per_dollar NUMERIC,
                kelly_fraction NUMERIC, conservative_stake_cents BIGINT,
                tier TEXT, confidence INT, rank INT,
                decision TEXT CHECK(decision IN ('BET','NO_BET')),
                reason TEXT, created_at)

-- orders we actually placed (paper or live)
orders(id PK, recommendation_id FK, market_id FK, mode TEXT CHECK(mode IN ('paper','live')),
       side TEXT, outcome TEXT, count INT, limit_price_cents INT,
       kalshi_order_id TEXT, status TEXT, placed_at, filled_at)

-- open + settled positions
positions(id PK, market_id FK, outcome TEXT, count INT,
          avg_cost_cents INT, mode TEXT, opened_at, closed_at,
          settle_result TEXT, pnl_cents BIGINT)

-- immutable audit log of every recommendation + action
audit_log(id PK, ts, kind TEXT, payload JSONB)

-- daily P/L + risk rollup
daily_ledger(id PK, day DATE UNIQUE, mode TEXT, realized_pnl_cents BIGINT,
             unrealized_pnl_cents BIGINT, bets_placed INT,
             daily_loss_cents BIGINT, halted BOOL DEFAULT false)
```

All money is stored as integer cents. Probabilities are 0–1 numerics.

---

## 3. API routes (FastAPI)

```
Auth
  POST   /api/auth/login                 -> session cookie
  POST   /api/auth/logout

Markets
  GET    /api/markets?sport=soccer       -> tracked markets + price/implied/liquidity
  POST   /api/markets/track              -> add tickers to watchlist
  GET    /api/markets/{ticker}/orderbook -> latest book + movement

Research / quant
  POST   /api/recommendations/run        -> (re)compute edges for tracked markets
  GET    /api/recommendations            -> ranked list (best..worst) for today
  GET    /api/recommendations/best       -> BEST BET OF THE DAY (or NO BET)
  GET    /api/markets/avoid              -> markets flagged no-edge / illiquid

Trading (guarded)
  POST   /api/orders/preview             -> dry-run: stake, fees, risk checks, NO live call
  POST   /api/orders/approve             -> places order; body must echo recommendation_id
                                            + idempotency key; re-runs risk gate server-side
  GET    /api/positions                  -> open positions + unrealized P/L
  POST   /api/positions/{id}/close       -> sell/exit (guarded)

Bankroll / log
  GET    /api/bankroll                   -> balance, daily ledger, drawdown, halt state
  GET    /api/log                        -> audit log + recommendation history
  GET    /api/alerts                     -> price-move + T-90 reminders
```

The frontend never holds keys or signs requests; it only calls these routes with a session
cookie. Only `/api/orders/approve` and `/api/positions/{id}/close` can move real money, and
both re-validate every risk rule server-side before touching Kalshi.

---

## 4. Kalshi API integration plan

- **Environments:** demo `https://demo-api.kalshi.co/trade-api/v2` (default), prod
  `https://api.elections.kalshi.com/trade-api/v2`. `env` lives in `settings`; prod requires an
  explicit flag flip.
- **Auth:** API key ID + RSA private key. Every request signs `timestamp(ms)+METHOD+path`
  with RSA-PSS / SHA-256 and sends `KALSHI-ACCESS-KEY`, `KALSHI-ACCESS-SIGNATURE`,
  `KALSHI-ACCESS-TIMESTAMP`. Private key is read from an env-var path, never logged, never
  sent to the frontend. (Implemented in `app/kalshi/client.py`.)
- **Reads:** `GET /markets`, `GET /markets/{ticker}`, `GET /markets/{ticker}/orderbook`,
  `GET /events`, `GET /portfolio/balance`, `GET /portfolio/positions`.
- **Writes (guarded):** `POST /portfolio/orders` (limit orders only, IOC/GTC), `DELETE` to
  cancel. Paper mode short-circuits before this call and records a simulated fill at the
  current best price.
- **Streaming:** WebSocket `wss://…/trade-api/ws/v2` for live orderbook/price deltas feeding
  the movement + alert engine (added with the jobs layer).
- **Rate limiting + retries:** token-bucket limiter, exponential backoff, and a hard
  circuit-breaker that flips the system to read-only if the order endpoint errors repeatedly.

---

## 5. Risk-management rules (enforced server-side, not advisory)

1. **Edge-based sizing only.** Stake derives from fractional Kelly on the *edge*
   (`true_prob − price`), then clamped to the tier dollar band. Win-probability never sizes a bet.
2. **Tier gate.** Default `min_tier_to_bet = C`. Anything below C tier → `NO BET`, no override
   in the normal path.
3. **Per-market cap.** `max_stake_per_market_cents` (default $100) hard-caps any single ticket.
4. **Daily loss limit.** When realized loss for the day ≥ `max_daily_loss_cents` (default $150),
   the day is **halted** — `/api/orders/approve` rejects until next day.
5. **Drawdown stop-loss.** If bankroll falls ≥ `stop_loss_drawdown_pct` (default 20%) from its
   high-water mark, the system goes read-only until manually reset.
6. **Manual approval required.** No order is ever placed without an explicit approve call that
   echoes the exact `recommendation_id` it is acting on; stale recommendations are rejected.
7. **Paper-first.** `env=demo` + `mode=paper` is the default. Going live requires flipping
   `settings.env` to `prod` *and* passing `mode=live` per order.
8. **Idempotency.** Every approve carries a client key; duplicates are no-ops (no double-fills).

Tier → stake band (sized within band by conservative Kelly, then risk-clamped):

| Tier | Edge (true − price) | Stake band |
|------|---------------------|------------|
| A    | ≥ +10%              | $75–100    |
| B    | +7% to +10%         | $50–75     |
| C    | +4% to +7%          | $25–50     |
| D    | +2% to +4%          | $5–25 (off by default) |
| F    | < +2%               | NO BET     |

---

## 6. Exact file structure

```
kalshi-edge/
├── ARCHITECTURE.md
├── README.md
├── docker-compose.yml                  # postgres + backend + worker (later pass)
├── backend/
│   ├── requirements.txt
│   ├── .env.example
│   └── app/
│       ├── __init__.py
│       ├── main.py                     # FastAPI app + routers
│       ├── config.py                   # env-var settings, secrets handling
│       ├── kalshi/
│       │   ├── __init__.py
│       │   ├── client.py               # signed REST client (demo|prod)
│       │   └── models.py               # response models  (later pass)
│       ├── research/
│       │   ├── __init__.py
│       │   └── base.py                 # ResearchAdapter interface + sharp-odds stub
│       ├── quant/
│       │   ├── __init__.py
│       │   ├── probability.py          # implied prob, de-vig
│       │   └── edge.py                  # edge, EV, Kelly, tiering, stake
│       ├── risk/
│       │   ├── __init__.py
│       │   └── manager.py              # all risk rules
│       ├── db/
│       │   ├── __init__.py
│       │   ├── models.py               # SQLAlchemy models == schema above
│       │   └── session.py              # engine/session  (later pass)
│       └── api/
│           ├── __init__.py
│           └── routes.py               # route handlers  (later pass)
└── frontend/                           # Next.js + Tailwind dashboard (next pass)
    └── (scaffold)
```

This pass delivers the design above plus the correctness-critical modules: `config.py`,
`kalshi/client.py`, `quant/probability.py`, `quant/edge.py`, `risk/manager.py`,
`db/models.py`, `research/base.py`, and a runnable `main.py` skeleton. Next passes add the
jobs layer, WebSocket alerts, full route wiring, and the Next.js dashboard.
