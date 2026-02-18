# Hyperliquid Funding Rate Dashboard — Technical Spec

*Version 1.0 — Feb 18, 2026*

---

## Product Vision

A real-time funding rate dashboard and strategy platform for Hyperliquid traders. Shows funding rates across HL, Binance, Bybit, and OKX side-by-side, highlights spread opportunities, sends alerts, and lets users backtest and paper trade funding arb strategies.

**Tagline:** *See the edge before you trade it.*

---

## Market Gap

- Hyperliquid has a basic `/fundingComparison` page — no alerts, no sorting by spread, no history
- Velo.xyz covers it but is expensive and not HL-centric
- Coinalyze doesn't prioritize HL
- Nobody does HL-specific spread alerts or strategy backtesting

---

## API Reference

### Hyperliquid
- **REST:** `POST https://api.hyperliquid.xyz/info`
  - Body: `{"type": "metaAndAssetCtxs"}` → returns all assets with current funding rate
  - Body: `{"type": "fundingHistory", "coin": "BTC", "startTime": <ms>}` → historical
- **WebSocket:** `wss://api.hyperliquid.xyz/ws`
- **Auth:** None required
- **Rate limits:** No hard limits documented
- **Note:** HL uses hourly funding (not 8h like CEXs) — normalize to 8h-equivalent for comparison

### Binance
- **Endpoint:** `GET https://fapi.binance.com/fapi/v1/premiumIndex`
- Batch fetches all symbols in one call
- Rate limit: 500 weight / 5 min
- No auth for public data

### Bybit
- **Endpoint:** `GET https://api.bybit.com/v5/market/tickers?category=linear`
- Batch fetches all linear perps
- Rate limit: 10 req/sec
- No auth for public data

### OKX
- **Endpoint:** `GET https://www.okx.com/api/v5/public/funding-rate?instId=BTC-USD-SWAP`
- Per-symbol calls (annoying — need to batch manually)
- Rate limit: 20 req / 2 sec
- No auth for public data

---

## Tech Stack

| Layer | Choice | Reasoning |
|-------|--------|-----------|
| Runtime | Bun | Fast, modern, TypeScript native |
| Backend framework | Hono | Lightweight, Bun-compatible |
| Frontend | Next.js | React ecosystem, easy deployment |
| Cache | Upstash Redis | Serverless, cheap, perfect for rate data |
| Database | Postgres (Supabase) | Free tier, historical data storage |
| Hosting | Vercel (frontend) + Fly.io (backend) | Cheap, scalable |
| Payments | Stripe | Standard |
| Alerts | Telegram Bot API | Free, users already have Telegram |

**Estimated monthly infra cost: $5-27/mo** at MVP scale

---

## MVP — Phase 1 (3 Weeks)

Ruthlessly scoped. Ship this, nothing more.

### Features
- [ ] Sortable funding rate table — all HL markets vs Binance/Bybit/OKX
- [ ] Spread column — HL rate minus best CEX rate, color-coded
- [ ] Normalize all rates to 8h-equivalent
- [ ] 30-second auto-refresh (free tier)
- [ ] Telegram alert bot — notify when spread crosses user-defined threshold
- [ ] 1 alert per free user, unlimited for Pro

### Explicitly NOT in MVP
- No user accounts / auth
- No historical charts
- No WebSocket (polling is fine for MVP)
- No backtesting
- No paper trading
- No mobile optimization (desktop first)

### Pages
1. `/` — Main dashboard table
2. `/alerts` — Set up Telegram alerts (enter chat ID + threshold)
3. `/pricing` — Free vs Pro vs API

---

## Phase 2 — Backtesting & Paper Trading (Weeks 4-10)

This is what turns a $15/mo dashboard into a $99-299/mo platform.

### Historical Data (Start Day 1)
- Archive all funding rate snapshots to Postgres from launch
- Even during Phase 1, store everything — the backtest engine needs this data
- Schema: `(timestamp, exchange, asset, rate_8h_equiv, open_interest)`
- Store every poll (every 30s) — cheap at this volume

### Backtesting Engine
Define a strategy with simple rules:
```
ENTER when: HL rate > 0.05%/8h AND spread vs best CEX > 0.02%
EXIT when: HL rate < 0.01%/8h OR spread inverts
POSITION SIZE: $X notional (configurable)
```

Engine replays historical data through the strategy and outputs:
- Total P&L
- Sharpe ratio
- Max drawdown
- Win rate per asset
- Best/worst performing assets
- Funding collected vs slippage/fees estimate

### Paper Trading Mode
- User defines a strategy
- System runs it in real-time with simulated capital (default $10K)
- Tracks: simulated entries/exits, funding collected, current P&L
- No real money, fully realistic conditions
- Dashboard shows live paper portfolio alongside the rate table

### Why This Matters
- De-risks real trading — backtest → paper trade → go live
- Dramatically increases WTP ($15/mo → $99-299/mo)
- Creates stickiness — users with 6 months of paper trading data don't churn
- Natural upsell: "Your paper strategy made 34% — go live?"

---

## Monetization

| Tier | Price | Features |
|------|-------|---------|
| **Free** | $0 | 30s refresh, top 20 assets, 1 Telegram alert |
| **Pro** | $15/mo | 5s refresh, all assets, unlimited alerts, historical charts, backtesting, paper trading |
| **API** | $49/mo | REST API access for bot operators, 1s refresh, webhook alerts |

**Break-even: 20 Pro subscribers** (~$300 MRR covers all infra)

Stripe for payments. Supabase Auth for user accounts (Phase 2 when we need auth).

---

## Development Roadmap

### Week 1
- Set up repo, Bun + Hono backend
- Connect all 4 exchange APIs, normalize to 8h
- Basic table UI in Next.js
- Deploy to Fly.io + Vercel
- **Start archiving data to Postgres on day 1**

### Week 2
- Spread calculation and sorting
- Color-coded opportunity highlighting
- Telegram bot alert system
- Free vs Pro gating (simple API key for now)

### Week 3
- Polish UI
- Pricing page
- Stripe integration
- Launch on Twitter/crypto communities

### Week 4-6
- Historical charts (using archived data)
- Backtesting engine v1
- Paper trading mode

### Week 7-10
- Refine backtesting (better fee modeling, slippage estimates)
- Strategy sharing (users publish their backtested strategies)
- API tier
- HIP-4 integration (HL outcomes trading — add when mainnet launches)

---

## Launch Strategy

1. Ship free tier → post on HL Discord, CT, r/hyperliquid
2. "Here's the funding arb opportunity right now" thread with screenshots — drives organic traffic
3. Apply to Hyperliquid ecosystem grants
4. Upsell to Pro once users are hooked on the free data

---

## Future: HIP-4 Integration

When Hyperliquid's outcomes trading (HIP-4) goes mainnet:
- Add HIP-4 market prices alongside HL perp funding rates
- Cross-venue spread monitor: HIP-4 BTC outcome prices vs Polymarket BTC prices
- This becomes the arbitrage tool between prediction markets and perp markets
- Natural expansion of the platform with no new infrastructure needed

---

*This is a living spec. Update as we build and learn.*
