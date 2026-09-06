# Batch Prices

Source: /market-data/batch-prices

# Batch Prices

```http
POST /v1/prices/batch
```

Returns live prices for multiple markets in a single request. Much more efficient than calling `GET /v1/markets/{id}` individually.

---

## Request

```bash
curl -X POST https://api.polysimulator.com/v1/prices/batch \
  -H "X-API-Key: $API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"market_ids": ["0x1a2b3c...", "0x4d5e6f..."]}'
```

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `market_ids` | string[] | Yes | Array of condition_ids. Max = **your tier's `max_batch_size`** (see table below); the absolute hard ceiling enforced by the schema is **50**, but no documented tier reaches it. |

### Batch Size Limits

The maximum number of markets per request depends on your API key's rate limit tier:

| Tier | Max `market_ids` per request |
|------|:---------------------------:|
| `free` | 1 |
| `pro` | 5 |
| `pro_plus` | 10 |
| `enterprise` | 25 |

Exceeding your tier's limit returns a `400 BATCH_LIMIT_EXCEEDED` error.
The authoritative per-tier value is on the wire via `GET /v1/keys/tiers`
(the `max_batch_size` field). The schema-level hard ceiling is 50, but
since the highest tier (`enterprise`) caps at 25, you will hit your
tier limit long before the ceiling.

---

## Response

```json
[
  {
    "condition_id": "0x1a2b3c...",
    "buy": "0.65",
    "sell": "0.35",
    "outcomes": [
      {"label": "Yes", "price": "0.65", "token_id": "71321..."},
      {"label": "No", "price": "0.35", "token_id": "71322..."}
    ]
  },
  {
    "condition_id": "0x4d5e6f..."
  }
]
```

| Field | Type | Description |
|-------|------|-------------|
| `condition_id` | string | The requested market ID |
| `buy` | string \| null | Yes outcome price (0–1). Absent when price unavailable. |
| `sell` | string \| null | No outcome price (0–1). Absent when price unavailable. |
| `outcomes` | array | Per-outcome `{label, price, token_id}` breakdown |
| `source` | string \| null | An opaque provenance label that may accompany a price. Treat it as informational; **do not branch on its value** — the set is not a stable contract. |

  Markets with unavailable prices return only the `condition_id` field —
  all price fields will be absent. This can happen when a market is newly
  listed and hasn't been cached yet. Always check for the presence of
  `buy`/`sell` before using them.

---

## Use Cases

- **Portfolio valuation**: Fetch current prices for all your positions at once
- **Watchlist monitoring**: Track prices for markets you're interested in
- **Batch strategies**: Evaluate multiple markets before submitting batch orders

```python
import requests

API_KEY = "ps_live_..."
BASE = "https://api.polysimulator.com"
headers = {"X-API-Key": API_KEY}

# Fetch prices for all open positions
positions = requests.get(
    f"{BASE}/v1/account/positions",
    headers=headers,
    params={"status": "OPEN"},
).json()

market_ids = [p["market_id"] for p in positions]

prices = requests.post(
    f"{BASE}/v1/prices/batch",
    headers=headers,
    json={"market_ids": market_ids},
).json()

for p in prices:
    if p.get("buy") is not None:
        print(f"{p['condition_id'][:16]}: Yes={p['buy']}, No={p['sell']}")
    else:
        print(f"{p['condition_id'][:16]}: price unavailable")
```

---

## Error Handling

| Status | Meaning |
|--------|---------|
| `400` | `market_ids` missing, empty, or exceeds tier batch size limit |
| `401` | Invalid or expired API key |
| `429` | Rate limit exceeded — check `Retry-After` header |

```python
import time

resp = requests.post(
    f"{BASE}/v1/prices/batch",
    headers=headers,
    json={"market_ids": market_ids},
)

if resp.status_code == 200:
    prices = resp.json()
    for p in prices:
        if p.get("buy") is not None:
            print(f"{p['condition_id'][:16]}: {p['buy']}")
elif resp.status_code == 400:
    print(f"Bad request: {resp.json().get('message', resp.json())}")
elif resp.status_code == 429:
    retry_after = int(resp.headers.get("Retry-After", 1))
    time.sleep(retry_after)
```

---

## Next Steps

- [Markets](/market-data/markets) — Full market metadata
- [Order Book](/market-data/order-book) — Detailed liquidity depth
