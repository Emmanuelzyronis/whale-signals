# Implementation Plan — WhaleSignals

**Deadline:** October 23, 2026 @ 11:59pm ET
**Today:** September 24, 2026
**Days remaining:** 29

---

## MVP Definition

The minimum viable submission that satisfies the hackathon requirements:

1. **Working UW API integration** — at least options flow + dark pool endpoints
2. **Anomaly scoring** — at least 2 of the 4 checks (volume + dark pool divergence)
3. **AI explanation** — plain-English signal narrative from Azure OpenAI
4. **Visible output** — either a running dashboard or a CLI output with real signals
5. **Public GitHub repo** with a README that explains what it does
6. **Discord post** in #vibe-and-api-projects before Oct 23 11:59pm ET

Everything beyond this is a stretch feature.

---

## Phase 1: Foundation (Sep 24–26)

### Day 1 — Sep 24
- [ ] Get UW API trial key at unusualwhales.com/public-api#pricing
- [ ] Test endpoints manually with curl/httpie
- [ ] Set up Python backend scaffold: `backend/` with FastAPI + uvicorn
- [ ] Write UW REST client (`uw_client.py`): auth, retries, rate-limit handling
- [ ] Pull options flow + dark pool from REST — confirm data shape matches docs
- [ ] Write `.env` + confirm Azure OpenAI endpoint responds

### Day 2 — Sep 25
- [ ] WebSocket client (`uw_ws.py`): connect, parse stream, reconnect with backoff
- [ ] Async ingestion loop: WS + REST pollers running concurrently via asyncio
- [ ] PostgreSQL schema: `signals`, `dark_pool_prints`, `congressional_trades`, `watchlist`
- [ ] SQLAlchemy models + Alembic migration
- [ ] Ingestion writes to DB — confirm rows landing with real data

### Day 3 — Sep 26
- [ ] Congressional trades poller (5-min interval)
- [ ] Greek exposure pull (on-demand per signal)
- [ ] Volatility surface poller (15-min interval)
- [ ] Historical flow baseline cache (per-ticker, 6h TTL)
- [ ] End of day: all five data feeds confirmed working

---

## Phase 2: Scoring Engine (Sep 27–28)

### Day 4 — Sep 27
- [ ] Scoring engine scaffold (`scoring/engine.py`)
- [ ] Volume anomaly check — OI ratio + 30-day baseline comparison
- [ ] Dark pool divergence check — ticker + 2-hour window join
- [ ] Unit tests for both checks (pytest, no API calls needed — fixture data)

### Day 5 — Sep 28
- [ ] Congressional timing check — transaction_date proximity to calendar events
- [ ] Greek consistency check — delta/vega evaluation per signal
- [ ] Composite score (0–100) + evidence/inference/unknown taxonomy
- [ ] Integration test: run a real signal through all four checks end-to-end
- [ ] Score stored to DB with full breakdown

---

## Phase 3: Explanation Pipeline (Sep 29–30)

### Day 6 — Sep 29
- [ ] LangGraph graph: `build_context → generate_narrative → structure_output`
- [ ] Structured output schema (four fields: what_happened, why_it_matters, confidence_note, watch_for)
- [ ] Azure OpenAI integration — structured output mode
- [ ] Deterministic fallback — template renderer from scoring data, no model call
- [ ] UI label: model-backed vs deterministic

### Day 7 — Sep 30
- [ ] End-to-end pipeline test with real signals: ingest → score → explain → store
- [ ] Explanation stored to DB alongside score
- [ ] FastAPI endpoint: `GET /signals` — returns recent signals with scores + explanations
- [ ] FastAPI endpoint: `GET /signals/{id}` — full signal detail
- [ ] FastAPI endpoint: `POST /watchlist` — add ticker + threshold
- [ ] **MVP backend complete**

---

## Phase 4: Dashboard (Oct 1–10)

### Oct 1–2
- [ ] Next.js scaffold: `frontend/` with Tailwind v4, TypeScript
- [ ] TanStack Query setup — poll `/signals` every 10s
- [ ] Signal card component: ticker, score gauge, evidence breakdown, explanation
- [ ] Live flow feed: latest 20 signals, auto-updates

### Oct 3–5
- [ ] Recharts volume chart: per-ticker 7-day options flow history
- [ ] Score gauge component (radial chart showing 0–100 with check breakdown)
- [ ] Dark pool divergence indicator (directional alignment badge)
- [ ] Congressional monitor tab: timeline view of congressional trades with timing flags

### Oct 6–8
- [ ] Watchlist config UI: add/remove tickers, set score threshold
- [ ] Alert panel: signals that crossed threshold, sorted by score
- [ ] Filter by signal type (bullish/bearish/neutral)

### Oct 9–10
- [ ] Mobile-responsive layout
- [ ] Dark/light mode
- [ ] Error states + loading skeletons

---

## Phase 5: Polish + Deploy (Oct 11–21)

### Oct 11–15
- [ ] Edge case handling: API rate limits, WS disconnects, model timeouts
- [ ] Logging + structured error reporting
- [ ] Performance: signal processing latency target < 2s from ingest to explanation
- [ ] Deploy backend to Railway or Render
- [ ] Deploy frontend to Vercel

### Oct 16–18
- [ ] Load test: simulate 100 concurrent signals
- [ ] Watchlist alerts end-to-end test
- [ ] Fix regressions

### Oct 19–20
- [ ] Final pass on explanations quality — tune prompts
- [ ] Verify all API endpoints still responding with live data
- [ ] Write submission description (3–5 sentences for Discord post)

### Oct 21 — Demo Recording
- [ ] Record demo video (3 min max)
  - Show live signal arriving from WS stream
  - Show score breakdown (all four checks)
  - Show AI explanation with evidence/inference/unknown labels
  - Show congressional monitor with timing flag
  - Show watchlist alert firing

### Oct 22 — Buffer
- [ ] Fix anything that broke during demo recording
- [ ] Confirm repo is public with clean README

### Oct 23 — Submit
- [ ] Post in Discord `#vibe-and-api-projects` before 11:59pm ET
  - Title: `"Hackathon1: WhaleSignals"`
  - Include: GitHub link, live URL, 2-sentence description

---

## Stretch Features (only if core is solid)

- **Backtester**: replay historical flow data, score it, measure whether high-score signals preceded price moves
- **Kafka consumer**: consume UW Kafka stream instead of WebSocket (if they provide Kafka access)
- **MCP server integration**: expose WhaleSignals signal data through the UW MCP server
- **Signal clustering**: ChromaDB embeddings to find signals similar to historical ones
- **Politician-level analysis**: score a politician's track record (timing accuracy, trade size patterns)
- **Sector rotation detection**: aggregate signals by sector, flag when unusual activity clusters

---

## Risk Register

| Risk | Likelihood | Mitigation |
|---|---|---|
| UW API trial key has limited endpoints | Medium | Check docs on day 1; upgrade if needed |
| Azure OpenAI quota exceeded | Low | Deterministic fallback is already built in |
| WebSocket stream unstable | Medium | REST fallback + reconnect logic |
| Dashboard too slow for demo | Low | Pre-load demo data if live stream is thin |
| Run out of time before dashboard | Medium | CLI output is acceptable for MVP submission |
