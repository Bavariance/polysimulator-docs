# Batch Orders

Source: /trading/batch-orders

# Batch Orders

```
POST /v1/orders/batch
```

Place multiple orders in a single request, up to your tier's max batch size
(see the [table below](#batch-size-limits)). Each order is processed
independently — partial failures are reported per-order.

---

## Batch Size Limits

| Tier | Max Batch Size |
|------|:--------------:|
| `free` | 1 |
| `pro` | 5 |
| `pro_plus` | 10 |
| `enterprise` | 25 |

Authoritative on the wire via `GET /v1/keys/tiers` — `max_batch_size` field.
See [Rate Limits](/concepts/rate-limits) for the full per-tier matrix.

---

## Request

```bash
curl -X POST https://api.polysimulator.com/v1/orders/batch \
  -H "X-API-Key: $API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "orders": [
      {
        "market_id": "0xabc...",
        "side": "BUY",
        "outcome": "Yes",
        "quantity": "5",
        "order_type": "market",
        "price": "0.99"
      },
      {
        "market_id": "0xdef...",
        "side": "BUY",
        "outcome": "No",
        "quantity": "3",
        "order_type": "market",
        "price": "0.99"
      },
      {
        "market_id": "0xghi...",
        "side": "BUY",
        "outcome": "Yes",
        "quantity": "10",
        "order_type": "limit",
        "price": "0.45",
        "time_in_force": "GTC"
      }
    ]
  }'
```

Each order in the array follows the same schema as `POST /v1/orders`.

  **Market orders inside a batch still require `price`** — it's the
  worst-price slippage cap (Polymarket-faithful). Without it the entry
  is rejected with `Market orders require a 'price' field`. For BUY,
  set the maximum you'll pay; for SELL, the minimum you'll accept.
  See [Placing Orders](/trading/placing-orders) for the full
  slippage-cap rationale.

---

## Response

```json
{
  "results": [
    {
      "order_id": 42,
      "status": "FILLED",
      "price": "0.65",
      "quantity": "5",
      "notional": "3.25"
    },
    {
      "order_id": 43,
      "status": "FILLED",
      "price": "0.40",
      "quantity": "3",
      "notional": "1.20"
    },
    {
      "order_id": 0,
      "status": "REJECTED",
      "error": "INSUFFICIENT_BALANCE",
      "error_message": "Insufficient balance for BUY order"
    }
  ],
  "succeeded": 2,
  "failed": 1
}
```

  **Partial failures don't roll back successful orders.** Each order is
  independent — some may succeed while others fail. Always check the
  `succeeded` and `failed` counts.

### Error Response Format

Every entry in `results` is a full `OrderResponse`. A failed entry has
`status="REJECTED"` (a per-entry validation/business rejection) or
`status="ERROR"` (an internal error processing that entry), with two clean
machine-readable fields:

- **`error`** — the stable machine code (e.g. `INSUFFICIENT_BALANCE`,
  `MARKET_CLOSED`, `INVALID_QUANTITY`), matching the single-order error
  envelope byte-for-byte. See the [canonical trading error-code
  table](/bots/error-handling#common-error-codes).
- **`error_message`** — the human-readable prose for that failure.

Each failed entry in `results` has this shape:

```json
{
  "order_id": 0,
  "status": "REJECTED",
  "order_type": "market",
  "side": "BUY",
  "outcome": "Yes",
  "price": "0",
  "quantity": "1",
  "notional": "0",
  "fee": "0",
  "client_order_id": "bot-123",
  "error": "INSUFFICIENT_BALANCE",
  "error_message": "Insufficient balance for BUY order"
}
```

Branch on the `error` code; show `error_message` to humans. (Earlier drafts
showed `error` as a stringified Python-repr of the detail dict — that
behaviour was removed; `error` is now a clean code and `error_message`
carries the prose.)

  A batch request itself returns **HTTP 200** even when individual orders
  fail — iterate `results` to check per-order `status`. Request-level
  failures still use HTTP status codes: auth, rate limit, malformed JSON,
  **and an oversize batch** (see below).

#### Oversize batch → request-level 400

If `orders` exceeds your tier's max batch size, the **whole request** is
rejected with `400 BATCH_LIMIT_EXCEEDED` — no `results` array, no per-entry
processing. Check `GET /v1/keys/tiers` (`max_batch_size`) and split large
batches client-side.

```json
// 400 BATCH_LIMIT_EXCEEDED
{"error": "BATCH_LIMIT_EXCEEDED", "message": "Batch size 12 exceeds tier limit (10)"}
```

Note the key difference between the two failure shapes: a **request-level**
rejection (like this oversize-batch 400) uses the standard `{error, message}`
envelope, while a **per-entry** rejection inside a 200 `results` array uses
`{error, error_message}` (shown above). Branch on `error` in both cases.

---

## Use Cases

  
    Buy underweight positions and sell overweight positions in a single request.
  

  
    Place a ladder of limit orders at multiple price levels simultaneously.
  

  
    Buy small positions across multiple undervalued markets at once.
  

  
    1 batch request counts as 1 API call vs N individual calls.
  

---

## Next Steps

- [Slippage Protection](/trading/slippage-protection) — Control fill quality
- [Placing Orders](/trading/placing-orders) — Single order reference
