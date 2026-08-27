# String Numerics

Source: /concepts/string-numerics

# String Numerics

All price, quantity, balance, and monetary values in the API are returned as **strings**, not floats or integers.

```json
{
  "price": "0.65",
  "quantity": "10",
  "balance": "993.50",
  "notional": "6.50"
}
```

---

## Why Strings?

IEEE 754 floating-point arithmetic causes precision errors that are unacceptable in financial applications:

```python
# Float math is broken for money
>>> 0.1 + 0.2
0.30000000000000004

>>> 1000.00 - 6.50
993.4999999999999   # Wrong!

# String → Decimal is safe
>>> from decimal import Decimal
>>> Decimal("1000.00") - Decimal("6.50")
Decimal('993.50')   # Correct!
```

The backend uses Python's `Decimal` type internally. By returning strings, we ensure **zero precision loss** from server to client.

---

## How to Handle

```python Python
from decimal import Decimal

# Parse API response
price = Decimal(order["price"])     # Decimal("0.65")
qty = Decimal(order["quantity"])    # Decimal("10")
notional = price * qty              # Decimal("6.50") — exact

# Send in requests
payload = {
    "quantity": str(qty),           # "10"
    "price": str(price),            # "0.65"
}
```

```javascript JavaScript
// Use string arithmetic libraries like bignumber.js or decimal.js
import Decimal from "decimal.js";

const price = new Decimal(order.price);     // "0.65"
const qty = new Decimal(order.quantity);     // "10"
const notional = price.times(qty);           // "6.50"

// Send in requests
const payload = {
  quantity: qty.toString(),
  price: price.toString(),
};
```

```go Go
import "github.com/shopspring/decimal"

price, _ := decimal.NewFromString(order.Price)
qty, _ := decimal.NewFromString(order.Quantity)
notional := price.Mul(qty)  // "6.50"
```

---

## Fields That Are Strings

Every numeric field in the API uses string encoding:

| Category | Fields |
|----------|--------|
| **Prices** | `buy`, `sell`, `best_bid`, `best_ask`, `mid_price`, `spread`, `last_trade`, `GET /v1/price` → `price` |
| **Order** | `quantity`, `notional`, `fee`, `limit_price`, `fill_price` |
| **Account** | `balance`, `starting_balance`, `pnl`, `pnl_percent`, `unrealized_pnl` |
| **Position** | `avg_entry_price`, `current_price`, `market_value`, `unrealized_pnl` |
| **Order Book** | `size` (bid/ask levels) |
| **Candles** | `open`, `high`, `low`, `close` |
| **Volume** | `volume` |

  **Request bodies also expect strings** for `quantity` and `price` fields.
  Sending a raw float (e.g., `10.5` instead of `"10.5"`) may work but is
  not recommended — the API will coerce it, but you lose client-side precision.

### Known exception: `GET /markets/updown` `live_price.buy` / `live_price.sell`

The legacy `/markets/updown` endpoint emits `live_price.buy` and
`live_price.sell` as **floats**, not strings — every other API surface
uses strings. Parse defensively:

```python
buy = float(market["live_price"]["buy"])   # float here, str everywhere else
```

This is a known drift slated for unification; treat the float values
as informational quotes (UI display, signal detection), and convert
to `Decimal(str(buy))` before any order math:

```python
from decimal import Decimal
buy_dec = Decimal(str(market["live_price"]["buy"]))
```

The `/v1/markets`, `/v1/markets/{id}`, `/v1/prices/batch`, and
`WS /v1/ws/prices` paths all emit string prices today and aren't
affected.

---

## Polymarket-parity exceptions

A small number of endpoints emit numeric (JSON `number`) values instead
of strings, because the corresponding Polymarket endpoint has always
done so and bot SDKs ported from Polymarket type-narrow on
`typeof === "number"`. The list is intentionally short:

| Endpoint | Field | Type | Reason |
|---|---|---|---|
| `GET /v1/tick-size/{token_id}` | `minimum_tick_size` | `number` (double) | Live Polymarket emits `{"minimum_tick_size": 0.01}` as a JSON **number**, and `py-clob-client` compares it against numeric tick constants. Note: the `tick_size` field *inside* `GET /v1/book` is a **string** — only the standalone `/tick-size` endpoint emits a number. |
| `GET /v1/markets/updown` | `live_price.buy/sell` | `number` | PM-parity for crypto-spike feeds. |

For all OTHER price-bearing endpoints (`GET /v1/price`, `/v1/midpoint`,
`/v1/spread`, `/v1/last-trade-price`, `/v1/book`, `/v1/account/positions`,
etc.) the string convention continues to hold.

  **`GET /v1/price` returns a string.** Live Polymarket emits
  `GET /price` as `{"price": "0.28"}` — a JSON **string**, matching the
  wire most trading bots actually parse, and PolySimulator does the same.
  Wrap the value with `Decimal(resp["price"])` or `float(resp["price"])`
  to be robust. The `quote_at` (ISO string) and `age_ms` (int) freshness
  fields accompany the price.

---

## Next Steps

- [CLOB Compatibility](/concepts/clob-compatibility) — Polymarket API migration path
- [Placing Orders](/trading/placing-orders) — Use string numerics in practice
