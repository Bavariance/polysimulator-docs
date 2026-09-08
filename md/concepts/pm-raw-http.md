# Polymarket Raw-HTTP Endpoints

Source: /concepts/pm-raw-http

# Polymarket Raw-HTTP Endpoints

PolySimulator exposes the **exact path + payload shapes** Polymarket's
public CLOB API uses. SDK clients written against
`https://clob.polymarket.com` (e.g. `py-clob-client`,
`@polymarket/clob-client`) can be pointed at PolySimulator by changing
**only the base URL and credentials** — no payload rewrites, no
field renames.

  This page covers the **raw-HTTP** shape only. PolySimulator also
  offers two simpler shapes for new code:
  - `POST /v1/orders` — native (`market_id` + `outcome` + `quantity` +
    `price` + `time_in_force`). See [Placing Orders](/trading/placing-orders).
  - `POST /v1/clob/order` — legacy SDK convenience (`token_id` + `side`
    + `size` + `price` + `order_type`). See [CLOB Compatibility](/concepts/clob-compatibility).

  All three paths funnel through the same matching engine; pick by
  integration shape, not by behaviour.

---

## Order Placement

### `POST /v1/order` — single order

Mirror of Polymarket's [`POST /order`](https://docs.polymarket.com/clob/post-order).

**Request body:**

```json
{
  "order": {
    "tokenId": "54533043819946592547517511176940999955633860128497669742211153063842200957669",
    "makerAmount": "500000",
    "takerAmount": "1000000",
    "side": "BUY",
    "expiration": "0",
    "salt": 1234567890,
    "maker": "0x0000000000000000000000000000000000000000",
    "signer": "0x0000000000000000000000000000000000000000",
    "taker": "0x0000000000000000000000000000000000000000",
    "nonce": 0,
    "feeRateBps": 0,
    "signatureType": 0,
    "signature": "0x..."
  },
  "owner": "your-api-key-uuid",
  "orderType": "GTC",
  "deferExec": false
}
```

**Maker/taker amount conversion (load-bearing — get this wrong and
notional sizing is silently broken):**

PolySimulator follows Polymarket's 6-decimal fixed-point convention
for both USDC and outcome-token amounts:

| Side | `makerAmount` | `takerAmount` | Implied price | Implied quantity |
|------|---------------|---------------|---------------|------------------|
| `BUY`  | `round(price × qty × 1e6)` USDC | `round(qty × 1e6)` shares | `makerAmount / takerAmount` | `takerAmount / 1e6` |
| `SELL` | `round(qty × 1e6)` shares | `round(price × qty × 1e6)` USDC | `takerAmount / makerAmount` | `makerAmount / 1e6` |

Example: BUY 1 share at $0.50 ⇒ `makerAmount=500000`,
`takerAmount=1000000`.

**On-chain fields are accepted and ignored:** `salt`, `maker`, `signer`,
`taker`, `nonce`, `feeRateBps`, `signatureType`, `signature`,
`timestamp`, `metadata`, `builder`. PolySimulator is paper trading —
there is no chain to settle against.

**`orderType` enum:** `GTC | FOK | FAK | GTD`. We don't support
good-til-date natively; `GTD` coerces to `GTC` so SDK callers get a
working order rather than a 422.

**Response:**

```json
{
  "success": true,
  "orderID": "6391",
  "status": "live",
  "makingAmount": "500000",
  "takingAmount": "1000000",
  "transactionsHashes": null,
  "tradeIDs": null,
  "errorMsg": ""
}
```

`status` enum on **write** (POST /v1/order, POST /v1/orders): `live`
(resting on book) / `matched` (filled immediately) / `unmatched`
(rejected). PolySimulator does **not** return PM's `delayed` status —
we don't delay matching.

  **Status enum split between write and read endpoints.** The
  write-side returns lowercase short forms (`live` / `matched` /
  `unmatched`) — that's PM's `SendOrderResponse` contract.

  The **read-side** (`GET /v1/data/orders`, `GET /v1/order/{id}`)
  returns PM's `OpenOrder.status` enum — PM's **exact** members:

  | Read endpoint | Write endpoint | Meaning |
  |---|---|---|
  | `ORDER_STATUS_LIVE` | `live` | resting on book (default for new GTC orders) |
  | `ORDER_STATUS_MATCHED` | `matched` | fully filled |
  | `ORDER_STATUS_CANCELED` | (n/a — write side never returns this) | cancelled or expired (single-L spelling, per PM) |
  | `ORDER_STATUS_INVALID` | `unmatched` | rejected / never accepted |

  The `?status=` filter accepts the single-L `ORDER_STATUS_CANCELED`
  spelling as well as the double-L `ORDER_STATUS_CANCELLED` alias.

  This mirrors Polymarket's real CLOB exactly — the two enums come
  from different protobufs in PM's internal codebase. SDK clients
  should treat them as separate types and map between them at the
  application layer (`@polymarket/clob-client` does this internally;
  raw `fetch` users need to do it themselves).

### TypeScript / Node example

```typescript
const BASE = "https://api.polysimulator.com";
const KEY = process.env.POLYSIM_API_KEY!;

// Token IDs are 77-digit decimal strings — DO NOT cast to Number.
// They overflow Number.MAX_SAFE_INTEGER (2^53). Keep them as strings
// throughout your codebase; use BigInt only if you need to do math.
const tokenId = "54533043819946592547517511176940999955633860128497669742211153063842200957669";

// BUY 1 share at $0.50:
//   makerAmount = 0.50 * 1 * 1e6 = 500_000  (USDC, 6-decimal fixed-point)
//   takerAmount = 1   * 1e6     = 1_000_000 (shares, also 6-decimal)
const sendOrder = await fetch(`${BASE}/v1/order`, {
  method: "POST",
  headers: {
    "X-API-Key": KEY,
    "Content-Type": "application/json",
  },
  body: JSON.stringify({
    order: {
      tokenId,                  // string, NOT number
      makerAmount: "500000",
      takerAmount: "1000000",
      side: "BUY",
    },
    owner: "your-api-key-uuid",
    orderType: "GTC",
  }),
});
const result = await sendOrder.json();
console.log(result.orderID, result.status); // "12345" "live"

// Listing orders later: PM envelope, status uses ORDER_STATUS_* prefix
const orders = await fetch(
  `${BASE}/v1/data/orders?limit=10`,
  { headers: { "X-API-Key": KEY } },
).then(r => r.json());

for (const o of orders.data) {
  // o.status is "ORDER_STATUS_LIVE" / "ORDER_STATUS_MATCHED" /
  // "ORDER_STATUS_CANCELED" / "ORDER_STATUS_INVALID" — PM's exact
  // members, a different enum from the write-side response above.
  // See the Warning callout.
  console.log(o.id, o.status, o.original_size, o.size_matched);
}
```

  **TypeScript users**: the [PolySimulator API](https://docs.polysimulator.com)
  Mintlify spec is auto-generated from the live OpenAPI; if you're
  using `openapi-typescript` to generate types, run it against
  `https://docs.polysimulator.com/openapi.json` (auto-rebuilds on every
  deploy).

---

### `POST /v1/orders` — single object **or** PM batch array

`POST /v1/orders` dispatches on the JSON body type:

- A **JSON object** is the **native** single order:
  `{market_id, side, outcome, quantity, order_type, price, time_in_force}`.
- A **JSON array** is a **Polymarket-shape batch** of up to **15**
  `PmSendOrderRequest` entries — each entry is validated and executed
  independently (per-entry failure isolation, matching PM's "failures in
  one entry do not abort the others" contract). A malformed entry
  surfaces as a failed result at its array index rather than rejecting
  the whole batch. More than 15 entries returns `400`.

```bash
# Native single order (object body)
curl -X POST https://api.polysimulator.com/v1/orders \
  -H "X-API-Key: $API_KEY" -H "Content-Type: application/json" \
  -d '{"market_id": "0x...", "side": "BUY", "outcome": "Yes", "quantity": "10", "order_type": "GTC", "price": "0.65"}'

# PM-shape batch (array body, ≤15 entries)
curl -X POST https://api.polysimulator.com/v1/orders \
  -H "X-API-Key: $API_KEY" -H "Content-Type: application/json" \
  -d '[{"order": {...}, "owner": "...", "orderType": "GTC"}, {"order": {...}, "owner": "...", "orderType": "FAK"}]'
```

For a PM-shape **single** order you can also use `POST /v1/order`
(singular):

```bash
curl -X POST https://api.polysimulator.com/v1/order \
  -H "X-API-Key: $API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"order": {...}, "owner": "...", "orderType": "GTC"}'
```

---

## Order Lifecycle

### `GET /v1/data/orders` — list user's orders, PM envelope

Polymarket-shaped `OrdersResponse`:

```json
{
  "limit": 100,
  "count": 5,
  "next_cursor": "MjAyNi0wNS0wNVQwMTozNjoxNy40MTY5NDcrMDA6MDA",
  "data": [
    {
      "id": "6390",
      "status": "ORDER_STATUS_MATCHED",
      "owner": "your-api-key-uuid",
      "maker_address": null,
      "market": "0x0f49db97...",
      "asset_id": null,
      "side": "BUY",
      "original_size": "1.0000",
      "size_matched": "0.2480",
      "price": "0.5000",
      "outcome": "Yes",
      "expiration": "0",
      "order_type": "GTC",
      "associate_trades": [],
      "created_at": 1778103749
    }
  ]
}
```

Pass `next_cursor` back as `?next_cursor=...` (PM's request param —
what py-clob-client sends) or the `?cursor=...` alias to paginate.
Cursor is urlsafe-base64 and opaque; the SDK's initial `MA==` seed and
the `LTE=` terminal sentinel both behave exactly as on real PM.

**Default scope matches PM: OPEN orders only.**
Pass the polysim-extension `?status=` param (`ORDER_STATUS_MATCHED`,
`ORDER_STATUS_CANCELED`, `ALL`, or friendly aliases like `matched` /
`canceled`) to read history.

**Filters:** `id`, `market`, `asset_id` (token id — resolves to the
market+outcome and filters both), `status`, `before` / `after`
(unix-seconds timestamps, PM convention; ISO accepted as an extra;
garbage returns 400 instead of being silently dropped), `limit`
(max 500).

### `GET /v1/order/{orderID}` — single order

Returns a single `OpenOrder` with the same shape as `data[]` above.
Hex order IDs (PM uses on-chain hashes; we use integer DB ids
serialised as strings) cleanly 404 with `error: ORDER_NOT_FOUND`
instead of crashing.

### `DELETE /v1/order` — cancel by body

```bash
curl -X DELETE https://api.polysimulator.com/v1/order \
  -H "X-API-Key: $API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"orderID": "6391"}'
```

Returns `{canceled: ["6391"], not_canceled: {}}` — PM's exact shape.

### `DELETE /v1/orders` — bulk cancel (array body)

```bash
curl -X DELETE https://api.polysimulator.com/v1/orders \
  -H "X-API-Key: $API_KEY" \
  -H "Content-Type: application/json" \
  -d '["6391", "6392", "6393"]'
```

Up to **100** ids per request (PM's documented cap is 3,000; the
PolySimulator cap is lower during the beta period). Invalid ids appear
in `not_canceled` with an explanation.

### `POST /v1/cancel-all` — cancel all open orders

No body. Returns the same `{canceled, not_canceled}` envelope.

---

## Read Endpoints (public — no auth)

These mirror Polymarket's public market-data endpoints. All return
the exact shape PM emits.

| Endpoint | Purpose | PM equivalent |
|---|---|---|
| `GET /v1/markets-by-token/{token_id}` | Resolve `tokenId` → `(condition_id, outcome)` | (PolySim convenience; PM does not expose this directly) |
| `GET /v1/midpoint?token_id=` | Midpoint price | `GET /midpoint` |
| `POST /v1/midpoints` | Batch midpoints | `POST /midpoints` |
| `GET /v1/spread?token_id=` | Bid/ask spread (capped at $0.10 per PM parity) | `GET /spread` |
| `POST /v1/spreads` | Batch spreads | `POST /spreads` |
| `GET /v1/book?token_id=` | L2 order book | `GET /book` |
| `POST /v1/books` | Batch books | `POST /books` |
| `GET /v1/last-trade-price?token_id=` | Last trade `{price, side}` | `GET /last-trade-price` |
| `GET /v1/tick-size/{token_id}` | Minimum tick size: `{minimum_tick_size: number}` ∈ `{0.1, 0.01, 0.001, 0.0001}` | `GET /tick-size/{token_id}` |
| `GET /v1/neg-risk/{token_id}` | `{neg_risk: bool}` for multi-outcome markets | `GET /neg-risk/{token_id}` |
| `GET /v1/time` | Bare integer Unix-epoch seconds (e.g. `1779147906`), **not** an object | `GET /time` |

  **Field-type details:**
  - `GET /v1/tick-size/{token_id}` returns `minimum_tick_size` as a JSON
    **number** (`0.01`), matching live Polymarket. Note that the
    `tick_size` field *inside* a `/v1/book` snapshot is a **string** —
    see [String Numerics](/concepts/string-numerics).
  - `GET /v1/time` returns a **bare integer** (`1779147906`), not a
    `{"server_time": ...}` object — this matches Polymarket's wire.

**Spread cap:** PolySim mirrors Polymarket's documented behaviour of
capping reported spread at $0.10. Markets with a `bid=0.001 ask=0.999`
book report `spread="0.10"`, not `0.998`. This stops arbitrage
filters from treating every illiquid book as a free-money signal.

---

## Tick-Size Enforcement

PolySimulator rejects limit orders whose `price` doesn't conform to
the market's minimum tick size — exactly as Polymarket does.

```json
HTTP 400
{
  "error": "INVALID_ORDER_MIN_TICK_SIZE",
  "message": "Price 0.575 breaks minimum tick size rule: 0.01. Round to the nearest multiple.",
  "request_id": "..."
}
```

Always quantise client-side using the result of
`GET /v1/tick-size/{token_id}` before submitting. SDKs that already
implement PM's quantisation (e.g. `py-clob-client.utilities.price_valid`)
work without modification.

Market orders use `price` as a **slippage bound**, not a strict price,
so off-grid `price` values are **not** tick-rejected on market orders.

---

## Beta Cohort Header

Beta-issued keys carry `X-API-Beta-Cutoff: expired` on every response
once the cohort cutoff date passes. SDKs can treat this header as the
signal to enter read-only mode:

```python
resp = client.get_orders()
if resp.headers.get("x-api-beta-cutoff") == "expired":
    print("Beta cohort ended — switch to a paid Pro key for trade access")
```

Public cohort status (no auth required):

```bash
curl https://api.polysimulator.com/api/beta/cohort-status
# → {"cohort_label": "beta-2026-05", "available": true, "active": 0,
#    "cap": 100, "cutoff": "2026-05-22T23:59:59Z", "reopens_at": null}
```

---

## SDK Migration Checklist

Porting a `py-clob-client` bot? Three lines of config:

```python
from py_clob_client.client import ClobClient

client = ClobClient(
    host="https://api.polysimulator.com",   # was clob.polymarket.com
    chain_id=137,                            # leave as-is (ignored)
    key="",                # leave as-is (ignored)
)
client.set_api_creds({"X-API-Key": "ps_live_..."})
```

`@polymarket/clob-client` (TypeScript) is the same: change `host`,
swap auth headers. `salt`, `signer`, `signature` continue to be
generated client-side and ignored server-side — no SDK changes
needed.

---

## Next Steps

- [Placing Orders](/trading/placing-orders) — Native `/v1/orders` shape
- [CLOB Compatibility](/concepts/clob-compatibility) — Three trading paths compared
- [Error Handling](/bots/error-handling) — Full error code reference
- [Authentication](/authentication) — `X-API-Key` and the `POLY_API_KEY` alias
