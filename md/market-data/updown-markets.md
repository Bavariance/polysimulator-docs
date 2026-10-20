# Up/Down Markets

Source: /market-data/updown-markets

# Up/Down Markets

```http
GET /markets/updown
```

  Both `/markets/updown` and `/v1/markets/updown` work — they return the
  identical payload. The `/v1/markets/updown` alias is not listed in the
  OpenAPI schema, but it is a live, fully-functional route, so you can keep
  your `/v1/...` base path everywhere. The companion interval-discovery
  endpoint `GET /v1/markets/updown/intervals` is a regular versioned route.

Returns active **up/down interval markets** — short-term binary markets that resolve based on whether an asset's price goes up or down within a specific time window.

---

## Query Parameters

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `asset` | string | _all_ | Filter by asset: `BTC`, `ETH`, `SOL`, `XRP`, `SPX`, `NDX` |
| `interval` | string | _all_ | Filter by interval: `5M`, `15M`, `1H`, `4H`, `daily` |
| `limit` | int | 200 | Max results (1–500) |

  `5M` and `15M` are subject to availability — see
  `GET /v1/markets/updown/intervals` to discover which intervals are
  currently active.

---

## Request

```bash
# All active up/down markets
curl -H "X-API-Key: $API_KEY" \
  https://api.polysimulator.com/v1/markets/updown

# Filter by asset and interval
curl -H "X-API-Key: $API_KEY" \
  "https://api.polysimulator.com/v1/markets/updown?asset=BTC&interval=1H"
```

  The un-versioned `https://api.polysimulator.com/markets/updown` form
  works identically if your existing code already uses it — same
  handler, same response.

---

## Response

```json
{
  "markets": [
    {
      "event_id": "evt_abc123",
      "condition_id": "0xabc123...",
      "slug": "btc-updown-1h-1770706800",
      "title": "BTC 1H Up/Down — March 15",
      "description": "Will BTC go up or down in the next hour?",
      "asset": "BTC",
      "interval": "1H",
      "time_range": "1:00 PM-2:00 PM ET",
      "date_display": "March 15",
      "start_date": "2026-03-15T13:00:00+00:00",
      "end_date": "2026-03-15T14:00:00Z",
      "active": true,
      "closed": false,
      "resolved": false,
      "tags": ["up-or-down", "bitcoin", "1h"],
      "markets": [
        {
          "id": "12345",
          "condition_id": "0xabc123...",
          "question": "Will Bitcoin go up or down in the next hour?",
          "outcomes": "[\"Up\", \"Down\"]",
          "outcome_prices": "[\"0.55\", \"0.45\"]",
          "token_ids": "[\"71321045...\", \"82432156...\"]"
        }
      ],
      "live_price": {
        "buy": 0.55,
        "sell": 0.45,
        "updated_at": "2026-03-15T13:44:58Z"
      },
      "group_item_threshold": null,
      "image": "https://polymarket.com/images/btc.png",
      "icon": "https://polymarket.com/icons/btc.svg"
    }
  ],
  "total": 42,
  "interval_counts": { "5M": 0, "15M": 0, "1H": 12, "4H": 8, "daily": 6 },
  "asset_counts": { "BTC": 10, "ETH": 8, "SOL": 6 },
  "available_intervals": ["1H", "4H", "daily"],
  "available_assets": ["BTC", "ETH", "SOL", "XRP"],
  "crypto_prices": {
    "BTC": { "price": 98500.50, "change_24h": 2.3 },
    "ETH": { "price": 3850.25, "change_24h": -1.1 }
  },
  "timestamp": "2026-03-15T13:45:00Z",
  "cache_hit": true
}
```

| Field | Description |
|-------|-------------|
| `markets` | Array of market entries (see fields below) |
| `total` | Total number of markets returned |
| `interval_counts` | Count of active markets per interval |
| `asset_counts` | Count of active markets per asset |
| `available_intervals` | Intervals that have at least one active market |
| `available_assets` | Assets that have at least one active market |
| `crypto_prices` | Current reference prices for tracked assets |
| `cache_hit` | Whether the result was served from cache |

  `markets` is the canonical market list. Group it client-side by each entry's
  `asset` and `interval` fields; the API does not duplicate full market objects
  into precomputed grouping trees.

### Market Entry Fields

| Field | Description |
|-------|-------------|
| `event_id` | Polymarket event ID |
| `condition_id` | Convenience copy of the **first nested market's** `condition_id`. For trading, read the per-market `markets[].condition_id` you intend to trade rather than this top-level alias. |
| `slug` | URL-safe slug for this market |
| `title` | Display title (e.g., "BTC 1H Up/Down — March 15") |
| `description` | Event description |
| `asset` | Asset symbol: `BTC`, `ETH`, `SOL`, `XRP`, `SPX`, `NDX` |
| `interval` | Time interval: `5M`, `15M`, `1H`, `4H`, `daily` |
| `start_date` | Calculated interval start (ISO 8601) |
| `end_date` | Resolution time (ISO 8601) |
| `markets` | Nested array with `condition_id`, `question`, `outcomes`, `outcome_prices`, `token_ids`. Note: `outcomes`, `outcome_prices` **and `token_ids`** are JSON-encoded **strings** (e.g. `"[\"Up\", \"Down\"]"`), as in Polymarket's Gamma API — parse each with a second `json.loads`. Do not split `token_ids` on commas: the brackets and quotes stay attached. `token_ids[i]` is the CLOB token of `outcomes[i]`. This differs from `live_price.buy`/`sell`, which are bare JSON numbers. |
| `live_price` | Object with `buy`, `sell`, `updated_at` — updated in real time. **Note:** `buy`/`sell` here are JSON **numbers** (floats), unlike the string prices used everywhere else — see the caveat below. |
| `group_item_threshold` | Strike price parsed from the event title — **usually `null` on intraday markets.** Measured 2026-09-06: 66 of 69 live BTC up/down markets returned `null`. The `5m` and `15m` intervals settle against the price at interval start, so there is no strike in the title to parse; only the daily `...-up-or-down-` markets carry one. **Do not build a strike-distance signal on this field without a null check** — it is absent on exactly the markets a 5-minute bot trades. |

---

## Trading Up/Down Markets

Up/down markets trade like any other market through `POST /v1/orders`. The
part bots get wrong is picking the round: the list holds the current round
**and** upcoming ones, so select by time rather than taking `markets[0]`, and
trade the nested `markets[0].condition_id`.

```python
import json, os, datetime as dt
import requests

BASE = os.environ.get("POLYSIM_BASE_URL", "https://api.polysimulator.com")
HEADERS = {"X-API-Key": os.environ["POLYSIM_API_KEY"], "Content-Type": "application/json"}

def ts(s):
    return dt.datetime.fromisoformat(s.replace("Z", "+00:00"))

# 1. The BTC 5-minute round that is open right now
rounds = requests.get(f"{BASE}/v1/markets/updown", headers=HEADERS,
                      params={"asset": "BTC", "interval": "5M"}).json()["markets"]
now = dt.datetime.now(dt.timezone.utc)
current = next(r for r in rounds if ts(r["start_date"]) <= now < ts(r["end_date"]))
market = current["markets"][0]
cid = market["condition_id"]
outcomes = json.loads(market["outcomes"])       # ["Up", "Down"]
token_ids = json.loads(market["token_ids"])     # same order as outcomes
print(current["slug"], "ends", current["end_date"])

# 2. The Up book (the same levels Polymarket shows for that token)
book = requests.get(f"{BASE}/v1/markets/{cid}/book", headers=HEADERS,
                    params={"outcome": "Up"}).json()
best_ask = min(float(level["price"]) for level in book["asks"])

# 3. Buy 10 Up with a worst-price cap one tick above the ask.
#    Market orders REQUIRE `price` (Polymarket-faithful slippage cap); without
#    it the API returns 400 "Market orders require a 'price' field".
order = requests.post(f"{BASE}/v1/orders", headers=HEADERS, json={
    "market_id": cid, "side": "BUY", "outcome": "Up", "quantity": "10",
    "order_type": "market", "price": f"{min(0.99, best_ask + 0.01):.2f}",
}).json()
print(order["order_id"], order["status"], order["price"], "fee", order.get("fee"))
```

  Prices from `/book` and order responses are strings (see
  [String Numerics](/concepts/string-numerics)); `live_price.buy` / `sell`
  on this endpoint are floats and are a display summary, not a quote. Price
  a bot off the book.

With the SDK the same steps are `client.list_updown(asset="BTC", interval="5M")`,
`client.get_book(cid, outcome="Up")` and `client.place_order(...)`. A ported
`py-clob-client` bot can keep `get_order_book(token_id)` and
`create_order` / `post_order` against `token_ids[outcomes.index("Up")]`
(see [CLOB compatibility](/concepts/clob-compatibility)).

  Up/down markets have short lifespans (5 minutes to 24 hours). After expiry, prices may
  show invalid values (both outcomes at 100%). The API blocks BUY orders on expired markets
  but allows SELL orders for emergency position exit.

---

## Next Steps

- [Placing Orders](/trading/placing-orders) — Full order placement guide
- [Markets](/market-data/markets) — Browse all active markets
- [Slippage Protection](/trading/slippage-protection) — Protect against price movement
