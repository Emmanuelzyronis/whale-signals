# WhaleSignals — Options Flow Intelligence

> Built for the **[Unusual Whales Hackathon](https://unusualwhales.substack.com/p/the-unusual-whales-hackathon)** · Deadline: October 23, 2026

A real-time options flow intelligence pipeline that ingests Unusual Whales data streams, scores anomalies across multiple dimensions, and returns plain-English explanations of why each signal is worth attention — with an honest evidence/inference/unknown breakdown per signal.

---

## Why This Is Different From a Dashboard

Most tools that wrap the UW API show you the data. WhaleSignals tells you what it means and why it thinks so.

Every signal gets:
- A **0–100 anomaly score** built from named, decomposable checks (Volume, Dark Pool Divergence, Congressional Timing, Greek Exposure)
- A **confidence breakdown**: what was observed (evidence), what is a reasonable interpretation (inference), and what is genuinely unavailable (unknown) — unknown is never collapsed into false confidence
- A **plain-English explanation** grounded in the actual signal data, not generic options commentary

---

## What It Does

1. **Real-time ingestion** — pulls live options flow, dark pool prints, congressional trades, and Greek exposure via UW REST and WebSocket APIs
2. **Anomaly scoring** — each incoming signal is scored across four dimensions:
   - *Volume* — is this trade unusual relative to open interest and recent history?
   - *Dark Pool Divergence* — does dark pool activity contradict the options flow direction?
   - *Congressional Timing* — is a congressional trade close in time to earnings, FDA decisions, or legislation?
   - *Greek Exposure* — are the Greeks (delta, gamma, vega) consistent with the stated direction?
3. **AI explanation** — Azure OpenAI generates a grounded, structured explanation for each scored signal. Deterministic fallback if the model is unavailable; the interface is identical either way.
4. **Watchlist alerts** — configure tickers, score thresholds, and signal types; receive alerts when conditions are met
5. **Congressional monitor** — dedicated feed for congressional trades with timeline context
6. **Next.js dashboard** — live flow visualization (Recharts), real-time updates (TanStack Query), signal detail drawer

---

## Architecture

```
UW API (REST + WebSocket)
        │
        ▼
  Ingestion Layer (FastAPI + asyncio)
  ├── Options flow stream
  ├── Dark pool feed
  ├── Congressional trades
  └── Greek exposure snapshots
        │
        ▼
  Scoring Engine (Python)
  ├── Volume anomaly check
  ├── Dark pool divergence check
  ├── Congressional timing check
  └── Greek consistency check
        │
        ▼
  Explanation Pipeline (LangGraph + Azure OpenAI)
  ├── Evidence / Inference / Unknown taxonomy
  ├── Plain-English signal narrative
  └── Deterministic fallback
        │
        ▼
  Storage (PostgreSQL)
        │
        ▼
  Next.js Dashboard (Recharts + TanStack Query)
  ├── Live flow feed
  ├── Signal cards with score breakdown
  ├── Congressional monitor
  └── Watchlist alert config
```

---

## Tech Stack

**Backend**
- Python 3.12, FastAPI, asyncio
- LangGraph (explanation pipeline orchestration)
- Azure OpenAI (signal narration)
- ChromaDB (embedding-based signal similarity for context)
- PostgreSQL (signal storage, watchlist config)
- Uvicorn, Pydantic, SQLAlchemy

**Frontend**
- Next.js 15, React 19, TypeScript
- Tailwind CSS v4
- Recharts (flow charts, score gauges)
- TanStack Query (live data polling + WebSocket sync)
- Zustand (local UI state)

**Data**
- [Unusual Whales API](https://api.unusualwhales.com/docs) — options flow, dark pool, congressional, Greeks, volatility
- REST (historical + snapshots) + WebSocket (live flow)

---

## UW API Endpoints Used

| Endpoint | Data | Used For |
|---|---|---|
| `GET /api/option-trades/flow-alerts` | Live options flow | Primary signal ingestion |
| `GET /api/option-trades/{ticker}` | Per-ticker flow | Watchlist monitoring |
| `GET /api/darkpool/recent` | Dark pool prints | Dark pool divergence check |
| `GET /api/darkpool/{ticker}` | Per-ticker dark pool | Ticker-level DP analysis |
| `GET /api/congress/recent` | Congressional trades | Congressional monitor feed |
| `GET /api/congress/{politician}` | Per-politician trades | Politician context |
| `GET /api/option-contract/{ticker}` | Greek exposure | Greek consistency check |
| `GET /api/market/volatility` | Volatility surface | Vega context for signals |
| `GET /api/stock/{ticker}/flow` | Historical flow | Volume baseline for scoring |
| `WS /api/option-trades/stream` | Real-time flow | Live ingestion stream |

Full endpoint docs: [api.unusualwhales.com/docs](https://api.unusualwhales.com/docs)

---

## Feature Status

### Core Pipeline
- [ ] UW API client (REST + WebSocket)
- [ ] Options flow ingestion
- [ ] Dark pool ingestion
- [ ] Congressional trades ingestion
- [ ] Greek exposure snapshots
- [ ] Volume anomaly check
- [ ] Dark pool divergence check
- [ ] Congressional timing check
- [ ] Greek consistency check
- [ ] Composite anomaly score (0–100)
- [ ] Evidence / inference / unknown taxonomy per signal
- [ ] Azure OpenAI explanation pipeline
- [ ] Deterministic fallback explanation
- [ ] PostgreSQL signal storage
- [ ] Signal deduplication

### Dashboard
- [ ] Next.js project scaffold
- [ ] Live flow feed (TanStack Query + WebSocket)
- [ ] Signal card with score breakdown + explanation
- [ ] Score gauge (Recharts)
- [ ] Volume chart per ticker (Recharts)
- [ ] Congressional monitor feed
- [ ] Watchlist config (add/remove tickers, set thresholds)
- [ ] Alert notification panel

### Submission
- [ ] Demo video (≤3 min)
- [ ] Discord post in #vibe-and-api-projects — title: `"Hackathon1: WhaleSignals"`
- [ ] Live deployment URL
- [ ] Public GitHub repo

---

## Timeline

| Date | Milestone |
|---|---|
| Sep 24 | Repo setup, UW API trial key, env config |
| Sep 25 | UW API client, REST + WebSocket ingestion working |
| Sep 26 | Scoring engine — all four checks implemented + tested |
| Sep 27 | Explanation pipeline — LangGraph + Azure OpenAI, deterministic fallback |
| Sep 28 | PostgreSQL storage, signal deduplication, watchlist config |
| Sep 29 | Next.js scaffold, live feed, signal cards |
| Sep 30 | Charts (Recharts), congressional monitor tab |
| Oct 1–5 | Polish, edge cases, alert system |
| Oct 6–20 | Watchlist alerts, deployment, testing under load |
| Oct 21 | Demo recording |
| Oct 22 | Buffer day |
| Oct 23 | **Submit to Discord by 11:59pm ET** |

---

## Getting Started

### Prerequisites
- Python 3.12+
- Node.js 20+
- PostgreSQL
- UW API key ([get trial](https://unusualwhales.com/public-api#pricing))
- Azure OpenAI deployment

### Environment
```bash
cp .env.example .env
# Fill in your keys
```

### Backend
```bash
cd backend
python -m venv .venv
source .venv/bin/activate
pip install -r requirements.txt
uvicorn main:app --reload
```

### Frontend
```bash
cd frontend
npm install
npm run dev
```

---

## Submission

Post in the Unusual Whales Discord at [discord.gg/unusualwhales](https://discord.gg/unusualwhales):
- Channel: `#vibe-and-api-projects`
- Title: `"Hackathon1: WhaleSignals"`
- Include: repo link, live URL, brief description

---

## License

MIT
