# Unusual Whales API — Endpoints Reference

Full docs: [api.unusualwhales.com/docs](https://api.unusualwhales.com/docs)
MCP server: [unusualwhales.com/public-api/mcp](https://unusualwhales.com/public-api/mcp)

All REST requests:
```
Authorization: Bearer {UW_API_KEY}
Base URL: https://api.unusualwhales.com
```

---

## Options Flow

### GET /api/option-trades/flow-alerts
Live options flow alerts — the primary signal source.

**Query params:**
| Param | Type | Description |
|---|---|---|
| `limit` | int | Max results (default 50) |
| `ticker` | string | Filter by ticker |
| `date` | string | YYYY-MM-DD |

**Key response fields:**
```json
{
  "ticker": "AAPL",
  "strike": 195,
  "expiry": "2026-10-17",
  "put_call": "CALL",
  "volume": 12500,
  "open_interest": 8300,
  "premium": 2150000,
  "sentiment": "BULLISH",
  "unusual_score": 87,
  "timestamp": "2026-09-24T14:32:00Z"
}
```

**Used for:** Volume anomaly check (volume / OI ratio), primary signal ingestion, WebSocket stream source.

---

### GET /api/option-trades/{ticker}
Per-ticker options flow history.

**Used for:** Building 30-day volume baseline for the volume anomaly check. Called once per ticker when it first enters the watchlist, then cached with a 1-hour TTL.

---

### WebSocket: wss://api.unusualwhales.com/api/option-trades/stream
Real-time options flow. Emits one JSON object per trade as it lands.

**Connection:**
```python
ws_url = "wss://api.unusualwhales.com/api/option-trades/stream"
headers = {"Authorization": f"Bearer {UW_API_KEY}"}
```

**Message format:** Same schema as the REST flow-alerts endpoint.

**Reconnection strategy:** Exponential backoff starting at 1s, cap at 60s. On reconnect, pull the last 5 minutes of REST flow-alerts to fill the gap.

---

## Dark Pool

### GET /api/darkpool/recent
Recent dark pool prints across all tickers.

**Key response fields:**
```json
{
  "ticker": "AAPL",
  "price": 193.40,
  "size": 250000,
  "side": "BUY",
  "premium": 48350000,
  "timestamp": "2026-09-24T14:28:00Z"
}
```

**Used for:** Dark pool divergence check. Polled every 30s. Joined against options flow by ticker + 2-hour window.

---

### GET /api/darkpool/{ticker}
Per-ticker dark pool history.

**Used for:** Building the DP baseline for a ticker. Called on demand when a ticker appears in options flow.

---

## Congressional Trading

### GET /api/congress/recent
Recent congressional trade filings.

**Key response fields:**
```json
{
  "politician": "Jane Smith",
  "ticker": "NVDA",
  "transaction_type": "Purchase",
  "amount": "100001-250000",
  "filed_date": "2026-09-20",
  "transaction_date": "2026-09-15",
  "committee": "Committee on Science, Space, and Technology"
}
```

**Used for:** Congressional timing check. Polled every 5 minutes. Joined against calendar events (earnings, FDA) to assess timing sensitivity.

**Note:** `filed_date` vs `transaction_date` matters — the trade may have happened days before filing. Both dates are stored; the timing check uses `transaction_date` for proximity assessment.

---

### GET /api/congress/{politician}
All trades for a specific politician.

**Used for:** Building politician-level context for the congressional monitor feed.

---

## Greek Exposure

### GET /api/option-contract/{ticker}
Greek snapshot for all active contracts on a ticker.

**Key response fields:**
```json
{
  "ticker": "AAPL",
  "strike": 195,
  "expiry": "2026-10-17",
  "put_call": "CALL",
  "delta": 0.62,
  "gamma": 0.08,
  "vega": 0.34,
  "theta": -0.15,
  "implied_volatility": 0.28,
  "timestamp": "2026-09-24T14:30:00Z"
}
```

**Used for:** Greek consistency check. Called per-signal on demand (not polled), since Greeks change constantly and need to be current at signal time.

---

### GET /api/market/volatility
Market-wide volatility surface.

**Used for:** Vega context. A high-vega trade in a low-IV environment scores differently than the same trade heading into earnings. Polled every 15 minutes.

---

## Stock Context

### GET /api/stock/{ticker}/flow
Historical options flow summary for a ticker.

**Key response fields:**
```json
{
  "ticker": "AAPL",
  "calls_volume": 185000,
  "puts_volume": 92000,
  "net_premium": 12500000,
  "avg_daily_volume_30d": 145000
}
```

**Used for:** Volume baseline in the volume anomaly check. Cached per ticker with a 6-hour TTL.

---

## Polling Schedule

| Endpoint | Interval | Reason |
|---|---|---|
| WS stream | Continuous | Primary live signal source |
| `/api/darkpool/recent` | 30s | Near-real-time DP divergence |
| `/api/congress/recent` | 5 min | Filings don't arrive faster |
| `/api/market/volatility` | 15 min | Surface changes slowly |
| `/api/stock/{ticker}/flow` | 6h | Volume baseline is stable |
| `/api/option-contract/{ticker}` | On demand | Greeks pulled per signal |
