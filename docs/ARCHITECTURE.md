# Architecture — WhaleSignals

## System Design

WhaleSignals is a four-layer system: ingestion, scoring, explanation, presentation. Each layer is independently testable. The explanation layer has a deterministic fallback — if Azure OpenAI is unavailable, the system produces grounded guidance from the structured scoring output without a model call. The interface is identical either way.

---

## Data Flow

```
┌─────────────────────────────────────────────┐
│           Unusual Whales API                │
│  REST endpoints + WebSocket stream          │
└───────────────┬─────────────────────────────┘
                │
    ┌───────────▼───────────┐
    │    Ingestion Layer     │
    │  (FastAPI + asyncio)  │
    │                       │
    │  ┌─────────────────┐  │
    │  │ WebSocket client│  │  ← live options flow
    │  └────────┬────────┘  │
    │           │           │
    │  ┌────────▼────────┐  │
    │  │  REST pollers   │  │  ← dark pool, congress, Greeks
    │  └────────┬────────┘  │
    └───────────┼───────────┘
                │
    ┌───────────▼───────────┐
    │    Scoring Engine     │
    │                       │
    │  Volume check         │  0–25 pts
    │  Dark pool divergence │  0–25 pts
    │  Congressional timing │  0–25 pts
    │  Greek consistency    │  0–25 pts
    │  ─────────────────    │
    │  Composite: 0–100     │
    │  + evidence taxonomy  │
    └───────────┬───────────┘
                │
    ┌───────────▼───────────┐
    │  Explanation Pipeline │
    │  (LangGraph)          │
    │                       │
    │  1. Build context     │  raw score + evidence
    │  2. Generate narr.    │  Azure OpenAI or fallback
    │  3. Structured output │  fields, not free prose
    └───────────┬───────────┘
                │
    ┌───────────▼───────────┐
    │   PostgreSQL storage  │
    │  signals, watchlists  │
    └───────────┬───────────┘
                │
    ┌───────────▼───────────┐
    │   Next.js Dashboard   │
    │                       │
    │  TanStack Query       │  REST polling + WS sync
    │  Recharts             │  flow charts, gauges
    │  Zustand              │  local UI state
    └───────────────────────┘
```

---

## Scoring Model

Each incoming signal is evaluated against four independent checks. Each check resolves to one of three states: **pass**, **warn**, or **unknown**. Unknown is never collapsed into a default score — it surfaces explicitly in the breakdown.

### Check 1: Volume Anomaly (0–25 pts)

Compares the trade's size against:
- Open interest ratio (trade size / OI)
- 30-day average daily volume for the contract
- Relative to sector peers on the same day

| State | Condition |
|---|---|
| pass (20–25) | Trade > 5× average volume AND > 10% of OI |
| warn (10–19) | Trade > 2× average, or elevated OI ratio |
| unknown | Insufficient historical volume data |

### Check 2: Dark Pool Divergence (0–25 pts)

Compares dark pool print direction against options flow direction on the same ticker within a 2-hour window.

| State | Condition |
|---|---|
| pass (20–25) | Dark pool and options flow directionally aligned (both bullish or both bearish) |
| warn (10–19) | One is neutral; direction unclear |
| unknown (0) | No dark pool activity for this ticker in the window |

Note: **contradictory** dark pool + options flow gets flagged separately as a divergence alert, not a pass — a DP buy against a large put sweep is a different signal entirely, not a scoring failure.

### Check 3: Congressional Timing (0–25 pts)

Evaluates whether a congressional trade landed within a sensitive window:
- ±5 days of earnings announcement
- ±10 days of relevant committee vote or bill passage
- ±14 days of FDA/regulatory decision on the ticker

| State | Condition |
|---|---|
| pass (20–25) | Trade within a sensitive window with committee relevance |
| warn (5–19) | Trade in window, no direct committee link |
| unknown (0) | No calendar context available for this ticker |

### Check 4: Greek Consistency (0–25 pts)

Evaluates whether the Greeks at the time of the trade are consistent with the implied direction:
- High delta call sweep: is delta > 0.6?
- Unusual vega: is vega elevated vs. 30-day average (volatility event play)?
- Gamma exposure: does the strike sit near a known GEX wall?

| State | Condition |
|---|---|
| pass (20–25) | Greeks strongly support the implied trade thesis |
| warn (8–19) | Greeks partially support; some inconsistency |
| unknown (0) | Greek data unavailable for this contract |

---

## Explanation Pipeline

The explanation pipeline is a three-node LangGraph graph:

```
build_context → generate_narrative → structure_output
```

**build_context**: assembles the scoring result, evidence strings, and inference strings into a context block. No model call.

**generate_narrative**: sends the context to Azure OpenAI with a structured output schema. The model fills four fields — `what_happened`, `why_it_matters`, `confidence_note`, `watch_for` — rather than generating free prose. Structured output makes the next step tractable.

**structure_output**: validates the model output against the schema and produces the final signal object. If the model call fails, the deterministic fallback runs `build_context` output through a template renderer that produces the same four fields from the scoring data directly.

The UI labels each explanation as `model-backed` or `deterministic` — the human can see which path ran.

---

## Evidence Taxonomy

Every signal carries a three-way breakdown:

- **Evidence**: what was directly observed from the API (trade size, strike, expiry, DP volume, congressional filing date)
- **Inference**: reasonable interpretation of observed facts (e.g. "elevated OI ratio suggests institutional size")
- **Unknown**: what is genuinely unavailable (e.g. "no dark pool data for this ticker in the window")

Unknown is a first-class field. It is never collapsed into evidence or inference. A signal with three evidence points and two unknowns presents differently — and more honestly — than one with five resolved checks.

---

## WebSocket vs REST

| | WebSocket | REST |
|---|---|---|
| Options flow | ✅ Primary (live stream) | Fallback if WS drops |
| Dark pool | ❌ | ✅ Polled every 30s |
| Congressional | ❌ | ✅ Polled every 5 min |
| Greeks | ❌ | ✅ Polled per-signal on demand |

The WebSocket client reconnects automatically with exponential backoff. REST pollers run on independent schedules via asyncio tasks, not threads. A failure in any poller does not affect the others or the WebSocket stream.
