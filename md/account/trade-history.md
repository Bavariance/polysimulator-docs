# Trade History

Source: /account/trade-history

# Trade History

```
GET /v1/account/history
```

Returns your filled orders (trade history) with filtering and pagination.

---

## Authentication

API key only — `X-API-Key: ` (or the PM-compat `POLY_API_KEY` /
`Authorization: Bearer ps_live_...` aliases). A Supabase Bearer JWT is **not**
accepted here.

---

## Query Parameters

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `market_id` | string | — | Filter by `condition_id` |
| `side` | string | — | Filter: `BUY` or `SELL` |
| `limit` | int | 50 | Max results (1–200) |
| `offset` | int | 0 | Pagination offset |
| `wallet_id` | int \| `"all"` \| `"api"` | `api` | Wallet scope. An integer scopes to a single wallet you own (404 `WALLET_NOT_FOUND` otherwise). `api` scopes to your API wallet (including legacy rows recorded before per-wallet attribution). `all` returns filled orders across **all** your wallets, UI MAIN/SANDBOX included. Keywords are case-insensitive; any other value returns 422 `VALIDATION_FAILED`. **Omitted = `api`.** |
| `envelope` | bool | `false` | When `true`, wrap the response in the Polymarket-shape `{ "data": [...], "next_cursor": "" }` envelope. `next_cursor` carries the next `offset` value as a string when a full page was returned (more rows likely exist), else `""`. Default `false` returns the bare array. |

  **Default changed on 2026-06-10.** Before 2026-06-10 the default (param
  omitted) returned filled orders across **all** wallets — UI MAIN/SANDBOX
  included. The default is now the **API wallet**, consistent with
  [Balance](/account/balance) / [Portfolio](/account/portfolio) /
  [Equity Curve](/account/equity-curve). Pass `wallet_id=all` if you depended
  on the old cross-wallet behaviour.

---

## Request

```bash
# All recent trades
curl -H "X-API-Key: $API_KEY" \
  "https://api.polysimulator.com/v1/account/history?limit=20"

# Only sells for a specific market
curl -H "X-API-Key: $API_KEY" \
  "https://api.polysimulator.com/v1/account/history?market_id=0x1a2b3c&side=SELL"
```

---

## Response

Orders are returned **most-recent first** (descending `filled_at`).

```json
[
  {
    "order_id": 42,
    "market_id": "0x1a2b3c...",
    "side": "BUY",
    "outcome": "Yes",
    "price": "0.65",
    "quantity": "10.0",
    "notional": "6.50",
    "fee": "0.09",
    "status": "FILLED",
    "price_source": "clob_midpoint_live",
    "price_age_ms": 12,
    "filled_at": "2026-02-06T12:00:45Z",
    "created_at": "2026-02-06T12:00:45Z"
  }
]
```

| Field | Type | Description |
|-------|------|-------------|
| `order_id` | int | Order ID |
| `market_id` | string | Market `condition_id` |
| `side` | string | `BUY` or `SELL` |
| `outcome` | string | Outcome name (e.g. `Yes` / `No`) |
| `price` | string | Fill price per share (string-encoded decimal) |
| `quantity` | string | Number of shares filled (string-encoded decimal) |
| `notional` | string | Total cash value of the fill (`price × quantity`), string-encoded decimal |
| `fee` | string | Taker fee charged on the fill, string-encoded decimal (USD) — the per-category fee `C × feeRate × p × (1 − p)` (example: 10-share BUY at $0.65 on a 4% politics market → `10 × 0.04 × 0.65 × 0.35 = $0.09`). `0.00` for fee-free fills (maker fills, geopolitics markets, emergency exits). The stored `fee` is always the authoritative debited amount. See [Trading Fees](/trading/fees). |
| `status` | string | Simulator order status. For trade history this is always `FILLED` (the endpoint returns filled orders only). See the note below on Polymarket parity. |
| `price_source` | string \| null | How the fill price was sourced (e.g. `clob_midpoint_live`). The exact label set is non-stable — substring-match rather than hard-coding members. See [Slippage Protection](/trading/slippage-protection) for the documented labels and how to monitor fill quality. `null` on legacy rows. |
| `price_age_ms` | int \| null | Age of the price data at execution time, in milliseconds; `null` when not recorded |
| `filled_at` | string \| null | ISO-8601 UTC fill timestamp |
| `created_at` | string \| null | ISO-8601 UTC order-creation timestamp |

  **Polymarket parity:** PolySimulator fills paper orders instantly, so the
  `status` here is the simulator's order status (`FILLED`), not Polymarket's
  on-chain trade lifecycle (`MATCHED → MINED → CONFIRMED`, with `RETRYING` /
  `FAILED`). There is no settlement-finality state to track for simulated
  trades. If you are porting a Polymarket SDK, treat `FILLED` as equivalent to
  a terminal `CONFIRMED` trade.

With `envelope=true` the rows are wrapped Polymarket-style, with a real cursor:

```json
{
  "data": [ /* … same row objects … */ ],
  "next_cursor": "50"
}
```

`next_cursor` is the next `offset` value (as a string) when a full page was
returned; it is `""` when the page was short (no more rows).

---

## Errors

All errors return `{"error": "<CODE>", "message": ""}`.

| Status | `error` code | When |
|--------|--------------|------|
| 401 | `MISSING_API_KEY` | No API key header supplied |
| 401 | `INVALID_KEY` | API key is unknown, deactivated, or expired |
| 404 | `WALLET_NOT_FOUND` | Integer `wallet_id` does not exist or is not owned by the caller |
| 422 | `VALIDATION_FAILED` | `wallet_id` is neither an integer, `all`, nor `api` |

---

## Next Steps

- [Equity Curve](/account/equity-curve) — Visualize portfolio value over time
- [Portfolio](/account/portfolio) — Aggregate portfolio snapshot
