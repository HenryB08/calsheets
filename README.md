# Kalshi Edge

A self-hosted dashboard for analyzing and (with manual approval) trading Kalshi
sports/World Cup markets. Demo/paper mode is the default; live trading is opt-in.

See **ARCHITECTURE.md** for the full design (architecture, schema, routes, Kalshi
integration plan, risk rules, file layout).

## What this is — and isn't
- **Is:** an edge calculator + risk-managed execution + auditable logging tool. It
  sizes bets off *edge* (true probability − price), enforces stop-loss / daily-loss /
  per-market caps, and defaults to `NO BET`.
- **Isn't:** an alpha machine. Kalshi's sports markets are efficiently priced, so the
  honest, normal output of the recommendation engine is `NO BET`. The probability model
  in `app/research` is the limiting factor; until something genuinely beats the sharp
  consensus, edges are rare and small. The platform's value is discipline and execution,
  not a money printer. Sizing never keys off raw win-probability, so a 94%-to-win
  favorite with no edge is staked at $0.

## Backend setup
```bash
cd backend
python -m venv .venv && source .venv/bin/activate
pip install -r requirements.txt
cp .env.example .env          # fill in KALSHI_KEY_ID + key path; keep KALSHI_ENV=demo
uvicorn app.main:app --reload
```
Generate a Kalshi API key in your account, download the RSA private key, and point
`KALSHI_PRIVATE_KEY_PATH` at it. The key never touches the frontend or the database.

## Try the quant core (no Kalshi account needed)
```python
from app.quant.edge import evaluate_outcome
r = evaluate_outcome("Germany", price=0.94, true_prob=0.94,
                     bankroll_cents=50000, kelly_frac=0.25,
                     min_tier="C", max_stake_per_market_cents=10000)
print(r.decision, r.tier, r.stake_cents)   # NO_BET F 0  -> no edge, no bet
```

## Delivered in this pass
- Full design doc (ARCHITECTURE.md)
- `config.py` — env-var settings + secrets handling
- `kalshi/client.py` — RSA-PSS signed REST client (demo|prod), reads + guarded orders, paper mode
- `quant/probability.py` — implied prob, de-vig, blending
- `quant/edge.py` — edge, EV, fractional Kelly, edge-based tiering, ranking
- `risk/manager.py` — per-market cap, daily-loss halt, drawdown stop-loss, tier gate, live-mode gate
- `research/base.py` — research adapter interface + sharp-consensus baseline
- `db/models.py` — SQLAlchemy schema
- `main.py` — FastAPI skeleton with `/api/orders/preview` and guarded `/api/orders/approve`

## Next passes
1. DB session + Alembic migrations + auth (login, session cookies)
2. Full route wiring (markets, recommendations, positions, bankroll, log, alerts)
3. Jobs: market refresh, edge recompute, settlement, T-90 reminders, price-move alerts (Celery beat)
4. WebSocket live orderbook feed
5. Next.js + Tailwind dashboard (best bet today / avoid / positions / P&L / upcoming / alerts)
6. Real `research` adapters (odds API + model) behind the `ResearchAdapter` interface
