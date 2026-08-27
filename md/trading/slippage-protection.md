# Slippage Protection

Source: /trading/slippage-protection

# Slippage Protection

PolySimulator uses the same slippage protection model as Polymarket:
the `price` field on every market order acts as a **worst-price limit**.

There is no separate slippage parameter — the price _is_ your slippage protection.

---

## How It Works

On Polymarket, all orders are limit orders. A "market order" is simply a
limit order at a marketable price with FOK (Fill-or-Kill) time-in-force.
PolySimulator mirrors this exactly.

The `price` field is **required** on market orders and sets the worst price
you'll accept:

- **BUY**: fills at the best ask, but **never above** your `price`
- **SELL**: fills at the best bid, but **never below** your `price`

```json
{
  "market_id": "0xabc...",
  "side": "BUY",
  "outcome": "Yes",
  "quantity": "10",
  "order_type": "market",
  "price": "0.68"
}
```

### BUY Example

```
Best ask:    $0.65
Your limit:  $0.68
→ FILLED at $0.65 (price improvement — you pay less than your limit)

Best ask:    $0.72
Your limit:  $0.68
→ REJECTED — fill price $0.72 exceeds your worst-price limit $0.68
```

### SELL Example

```
Best bid:    $0.60
Your limit:  $0.55
→ FILLED at $0.60 (price improvement — you receive more than your limit)

Best bid:    $0.50
Your limit:  $0.55
→ REJECTED — fill price $0.50 is below your worst-price limit $0.55
```

  **Polymarket migration**: This is identical to how Polymarket's `price`
  field works. Your existing slippage logic transfers directly to
  PolySimulator — and vice versa when you migrate to live trading.

---

## Price Is Required

Market orders **must** include a `price` field. Submitting a market order
without `price` returns a `400 PRICE_REQUIRED` error:

```json
{
  "error": "PRICE_REQUIRED",
  "message": "Market orders require a 'price' field as a worst-price limit..."
}
```

This matches Polymarket's design: there are no "blind" market orders.
You always control the worst price you'll accept.

---

## Time in Force

Market orders default to **FOK** (Fill-or-Kill), matching Polymarket.
You can also use **FAK** (Fill-and-Kill), Polymarket's term for IOC.

| Value | Behavior | Polymarket Equivalent |
|-------|----------|----------------------|
| `FOK` | All-or-nothing — fill entirely or cancel (default for market orders) | FOK |
| `FAK` | Fill available quantity, cancel remainder | FAK |
| `IOC` | Same as FAK (PolySimulator alias) | FAK |
| `GTC` | Overridden to FOK for market orders | — |

```json
{
  "market_id": "0xabc...",
  "side": "BUY",
  "outcome": "Yes",
  "quantity": "10",
  "order_type": "market",
  "price": "0.68",
  "time_in_force": "FAK"
}
```

  If you omit `time_in_force` on a market order, it defaults to FOK.
  If you explicitly set `GTC`, it is overridden to FOK — market orders
  never persist in the book.

---

## Execution Model

Market orders fill at the **best available price** — BUY at the best ask,
SELL at the best bid — matching Polymarket's execution model.

The fill-price resolution cascade (each tier falls through to the next on a
miss or a failed sanity guard):

1. **Order-book walk (VWAP)** across live CLOB levels — most realistic
2. **Best bid/ask** from the CLOB order book (cross the spread)
3. **CLOB midpoint** — sub-ms cache fast-path, then the cached read-through
4. **Live CLOB midpoint** — synchronous fetch (cold-market guard)
5. **Cached display price** — last-resort fallback

See [Trade Execution Internals](/trading/execution-internals) for the full
cascade and the exact `price_source` labels each tier emits.

---

## Fill Diagnostics

Every market order response includes transparency metadata:

```json
{
  "order_id": 42,
  "status": "FILLED",
  "price": "0.65",
  "price_source": "book_walk",
  "slippage_bps": 461
}
```

The `slippage_bps` field is **informational only** — it shows how far the
fill price deviated from the cached mid-price, in basis points. It does not
affect order execution. Your `price` (worst-price limit) is the only
protection mechanism.

### Price Sources

`price_source` is an **opaque diagnostic label** — log it for post-trade
analysis, but treat it as a non-stable enum (the label set evolves with
engine changes). Some labels carry a transport prefix (e.g. `ws:`) when the
underlying snapshot arrived over a streaming source rather than a poll —
strip the prefix or substring-match. The labels the engine actually emits
today, from highest to lowest confidence:

| Label (prefix-stripped) | Path | Quality |
|-------------------------|------|---------|
| `book_walk` | VWAP across live order-book levels | Gold standard — all telemetry fields populated |
| `best_ask` (BUY) / `best_bid` (SELL) | Single top-of-book quote | Good — `spread_bps` populated, walk fields null |
| `clob_midpoint_cached_hft` | Sub-ms midpoint cache | Good — `quote_age_ms = 0` |
| `clob_midpoint_cached` | Cached CLOB-midpoint read-through | Acceptable |
| `clob_midpoint_live` | Synchronous CLOB midpoint fetch (cold-market guard) | Acceptable |
| `outcome_yes` / `outcome_no` / `outcome_mid` / `midpoint` / `last_trade_only` / `last_trade_spread_fallback` | Cached display-price fallback | Lower confidence |
| `emergency_exit_entry_price` / `emergency_exit_entry_price_postexpiry` | Position-close at entry price (resolved / post-expiry SELL) | Informational — not a market-priced fill |

  **Monitor `price_source`** in your bot logs. Frequent fills on the
  `clob_midpoint_*` or cached-display-price labels (rather than `book_walk` /
  `best_ask` / `best_bid`) mean the live book was unavailable — consider
  pausing during degraded price quality. (Labels like `clob_book` or
  `redis_cache` are **not** emitted — don't grep for them.)

---

## Recommendations by Strategy

| Strategy | Approach |
|----------|----------|
| Scalping / HFT | `price` = best ask + $0.01 (tight limit, minimal overpay) |
| Swing trading | `price` = best ask + $0.03 (moderate buffer) |
| Bulk accumulation | `price` = best ask + $0.05 (wider buffer for fill certainty) |
| Illiquid markets | Use **limit orders** (GTC) for price certainty |

  For illiquid markets with wide spreads, consider **limit orders** instead
  of market orders. Limits give you price certainty at the cost of fill
  uncertainty.

---

## Comparison: PolySimulator vs Polymarket

| Feature | Polymarket | PolySimulator |
|---------|-----------|---------------|
| Slippage protection | `price` field (worst-price limit) | `price` field (identical) |
| Price required on market orders | Yes | Yes |
| Default time-in-force (market) | FOK | FOK |
| Execution model | BUY at best ask, SELL at best bid | BUY at best ask, SELL at best bid |
| Price improvement | Yes — fill at best available | Yes — fill at best available |
| Order types (`/v1/orders`) | FOK, FAK, GTC, GTD | GTC (default for limit), FOK (default for market), FAK, IOC; GTD rolling out |
| Fees | Per-category taker fees | Per-category taker fees (PM-V2 schedule, mirrored) |

---

## Fees

PolySimulator charges the **same per-category taker fee schedule as
Polymarket V2** — fees are **not** zero, takers pay and makers pay zero, and
the charged amount is returned in `OrderResponse.fee`. A bot computing PnL
on the assumption of zero fees will be wrong on most markets.

See [Trading Fees](/trading/fees) for the canonical formula, the full
per-category rate table, the maker/taker (marketability) classification, and
the documented divergences from Polymarket.

  Always read `OrderResponse.fee` when computing realized PnL — don't infer
  it from the rate alone, since the charged amount is price-dependent. To
  discover a market's rate up front, call `GET /v1/fee-rate?token_id=…` and
  read the `fee_rate_bps` field (the effective category rate in bps). See
  [Trading Fees](/trading/fees).

---

## Next Steps

- [Placing Orders](/trading/placing-orders) — Full order API reference
- [Order Book](/market-data/order-book) — Inspect liquidity before trading
