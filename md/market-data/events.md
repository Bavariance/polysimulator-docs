# Events

Source: /market-data/events

# Events

`/v1/events` is the event-first counterpart to `/v1/markets`. Each
response row is a parent event (e.g. *"2026 U.S. midterm elections"*)
with its child markets nested inside, plus live prices on each child.
For most algorithmic-trading workflows the flatter `/v1/markets` shape
is easier to iterate over — use `/v1/events` when you need the
event-level grouping (event title, end date, category) without making
a second request per market.

---

## List Events

```http
GET /v1/events
```

### Query Parameters

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `limit` | int | 50 | Max events to return (1–200). |
| `offset` | int | 0 | Pagination offset. |
| `hot_only` | bool | false | Only return events with high-volume markets. |
| `markets_per_event` | int | 20 | Cap on child markets returned per event (1–100). |
| `category` | string | — | Filter to a single category (e.g. `crypto`, `sports`, `politics`). Case-insensitive substring match against the event's classified category, which is derived from the event's tags. Use **unquoted** values: `?category=crypto`, NOT `?category="crypto"`. Leading/trailing single or double quotes are stripped defensively for users pasting from Postman. |
| `search` | string | — | Free-text search over event titles and market questions (case-insensitive substring). |
| `include_sports` | bool | true | When `false`, suppress sports events from the response. |

### Example

```bash
# All events, first page
curl -H "X-API-Key: $API_KEY" \
  "https://api.polysimulator.com/v1/events?limit=10"

# Crypto events only
curl -H "X-API-Key: $API_KEY" \
  "https://api.polysimulator.com/v1/events?category=crypto&limit=10"
```

### Response

```json
{
  "events": [
    {
      "id": "12345",
      "slug": "will-bitcoin-reach-100k-by-2026",
      "title": "Will Bitcoin reach $100K by end of 2026?",
      "image": "https://polymarket-upload.s3.us-east-2.amazonaws.com/...",
      "end_date": "2026-12-31T23:59:59Z",
      "is_hot": true,
      "category": "crypto",
      "markets": [
        {
          "condition_id": "0x1a2b3c...",
          "slug": "btc-100k-2026",
          "question": "Bitcoin price > $100,000 by 2026-12-31?",
          "outcomes": ["Yes", "No"],
          "live_price": {
            "buy": "0.65",
            "sell": "0.35"
          }
        }
      ]
    }
  ],
  "total": 9740,
  "limit": 10,
  "offset": 0,
  "has_sports": false
}
```

### Envelope fields

| Field | Type | Description |
|-------|------|-------------|
| `events` | `Event[]` | Events on this page — one object per parent event, each with its child markets capped at `markets_per_event`. |
| `total` | `int` | Total event count matching the filter across **all** pages (not just this page). |
| `limit` | `int` | Echo of the requested `limit`. |
| `offset` | `int` | Echo of the requested `offset`. |
| `cache_source` | `string` | An opaque provenance label that may accompany the response. Informational only — **do not branch on its value**; the set is not a stable contract. |
| `has_sports` | `bool \| null` | `true` when sports events were filtered out (i.e. you passed `include_sports=false` and some sports events matched). `false`/`null` otherwise. |
| `message` | `string \| null` | Present only on `503` (cache warming) and other operational states — carries a human-readable status. Absent on a normal `200`. |

  While the cache is warming the endpoint may return HTTP 503 with a
  `Retry-After: 5` header (and a `message` field in the body) — wait the
  suggested interval and retry. The cache is typically ready within
  ~30 seconds.

---

## Pairing with `/v1/markets`

The two endpoints draw from the same underlying data — every market
returned by `/v1/markets?category=crypto` belongs to one of the events
returned by `/v1/events?category=crypto`. Pick the shape that minimises
client-side work:

- **`/v1/markets`** — flat, one row per `condition_id`. Best for bots
  that iterate over individual markets.
- **`/v1/events`** — nested, one row per parent event. Best for UI
  rendering, category browsing, or any workflow where the event group
  matters.

---

## Next Steps

- [Markets](/market-data/markets) — flat market-first shape
- [Order Book](/market-data/order-book) — L2 depth for a specific market
- [Batch Prices](/market-data/batch-prices) — Multi-market price lookup
