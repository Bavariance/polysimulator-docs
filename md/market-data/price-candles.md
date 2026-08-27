# Price Candles

Source: /market-data/price-candles

# Price Candles

```http
GET /v1/markets/{condition_id}/candles
```

Returns OHLCV candlesticks. The price ticks come from Polymarket's CLOB
`/prices-history` endpoint — those arrive as `{t, p}` (single price per
tick, not OHLC), so PolySimulator buckets them server-side into the
requested interval and aggregates `open` (first tick), `high` (max),
`low` (min), `close` (last tick) per bucket. Volume is sourced from
**your internal fills** on PolySimulator, not Polymarket's chain volume.

  Each bucket aggregates every tick that falls within it, so candles show
  proper OHLC variation whenever the underlying tick stream has it. A bucket
  with a single tick reports `O = H = L = C` — see the note under
  [Response](#response).

---

## Query Parameters

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `outcome` | string | first available outcome | Outcome label (e.g. `Yes`, `No`, `Up`, `Down`) |
| `interval` | string | `1h` | Bucket interval — see table below |

### Available Intervals

| Interval | Bucket size |
|----------|-------------|
| `1h`     | 1 hour (default) |
| `6h`     | 6 hours     |
| `1d`     | 1 day       |
| `1w`     | 1 week      |
| `max`    | Treated as `1d` for bucketing — gives the longest meaningful intra-day signal |

  **Only `1h`, `6h`, `1d`, `1w`, and `max` are supported.** Sub-hour
  intervals (`1m`, `5m`, `15m`) and any other unrecognised value return
  **HTTP 400** with `{"error": "INVALID_INTERVAL", "message": "...",
  "supported_intervals": ["1h", "6h", "1d", "1w", "max"]}`. There is no
  silent fall-back to `1h` — the request fails loudly so you don't
  render an empty or mis-bucketed chart.

  Sub-hour granularity is unavailable because the upstream Polymarket
  CLOB `/prices-history` feed is hourly-granular — we can't reconstruct
  5-minute buckets from 1-hour samples. (Polymarket's own
  `/prices-history` enum does include `1m` and `all`, which
  PolySimulator does not support.)

---

## Example

```bash
curl -H "X-API-Key: $API_KEY" \
  "https://api.polysimulator.com/v1/markets/0x0f49db97f71c68b1e42a6d16e3de93d85dbf7d4148e3f018eb79e88554be9f75/candles?outcome=Yes&interval=1d"
```

---

## Response

```json
[
  {
    "t": 1777939200,
    "o": "0.2615",
    "h": "0.2680",
    "l": "0.2595",
    "c": "0.2625",
    "v": "98.0000"
  },
  {
    "t": 1778025600,
    "o": "0.2605",
    "h": "0.2605",
    "l": "0.2475",
    "c": "0.2475",
    "v": "3.0000"
  }
]
```

| Field | Type | Description |
|-------|------|-------------|
| `t`   | integer | Unix-second timestamp of the **start** of the bucket. For `1h` every value is a multiple of 3600. |
| `o`   | string  | Open — first tick price in the bucket |
| `h`   | string  | High — max tick price |
| `l`   | string  | Low — min tick price |
| `c`   | string  | Close — last tick price |
| `v`   | string  | Volume — sum of share-quantity from your internal FILLED orders within the bucket. `"0"` when no internal fills. Polymarket's chain volume is **not** included. |

  The `v` field reflects your own simulated trade flow only. To measure
  Polymarket-wide volume on a market, query the underlying tokens via
  the Polymarket Gamma API directly. PolySimulator's volume is meant
  for "did my own backtest fill?" sanity, not for liquidity proxies.

If a bucket has only a single tick, `o == h == l == c` — that's correct
behaviour for a slow-moving market, not a bug.

---

## Backtesting Example

```python
import requests
from decimal import Decimal

BASE = "https://api.polysimulator.com"
headers = {"X-API-Key": "YOUR_API_KEY"}

# Fetch hourly candles for the past day. Pass outcome explicitly when
# you want a specific side; default is the first outcome.
candles = requests.get(
    f"{BASE}/v1/markets/0x0f49.../candles",
    headers=headers,
    params={"interval": "1h", "outcome": "Yes"},
).json()

# Simple moving average crossover (close prices)
prices = [Decimal(c["c"]) for c in candles]
if len(prices) >= 48:
    sma_short = sum(prices[-12:]) / 12   # last 12 hours
    sma_long  = sum(prices[-48:]) / 48   # last 48 hours
    if sma_short > sma_long:
        print("Bullish crossover — consider BUY")
    else:
        print("Bearish crossover — consider SELL")
```

---

## Migrating from Polymarket

Polymarket's CLOB `/prices-history` returns the raw `{t, p}` tick stream
without bucketing. If you're porting a bot that does its own bucketing
client-side, you can either:

1. **Trust ours** — drop your bucketing code and use the `t/o/h/l/c/v`
   shape directly. The interval parameter behaves identically to a
   pandas `resample("1H").agg({"o": "first", "h": "max", ...})`.
2. **Keep yours** — fetch raw ticks via Polymarket's Gamma API, since
   PolySimulator does not (yet) expose the un-bucketed feed. Cross-host
   strategies that compare the two should bucket identically client-side.

---

## Next Steps

- [Batch Prices](/market-data/batch-prices) — Multi-market price lookup
- [Order Book](/market-data/order-book) — L2 depth for fill-price modelling
- [Equity Curve](/account/equity-curve) — Track your portfolio over time
