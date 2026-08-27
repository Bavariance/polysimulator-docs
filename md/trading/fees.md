# Trading Fees

Source: /trading/fees

# Trading Fees

PolySimulator mirrors **Polymarket's V2 taker fee schedule exactly**. Fees
are simulated money (this is paper trading), but the *economics* are real:
every taker fill is debited the same fee a live Polymarket fill would pay,
so a strategy that is fee-unprofitable here is fee-unprofitable there.

  Fees are **not zero**. A bot that models zero trading costs will compute
  wrong PnL on every non-geopolitics market. Always read the `fee` field
  returned on fills (`OrderResponse.fee`, `/v1/account/history`).

---

## Fee Formula

```
fee = C × feeRate × p × (1 − p)
```

| Symbol | Meaning |
|--------|---------|
| `C` | Number of shares filled |
| `feeRate` | The market category's taker rate (table below) |
| `p` | Fill price (0–1) |

The `p × (1 − p)` factor makes the fee symmetric around $0.50 — it peaks
there and shrinks toward the price extremes, so a fill at $0.99 pays a tiny
fraction of what the same share count pays at $0.50. Fees are rounded to 5
decimal places; the smallest charged fee is 0.00001 USD (amounts below that
round to zero). The amount actually debited from your balance is settled at
cent precision.

**Worked example** — 10-share BUY at $0.65 on a crypto market:

```
fee = 10 × 0.07 × 0.65 × 0.35 = 0.15925 USD
```

---

## Category Rates

The schedule mirrors [Polymarket's published fee
table](https://docs.polymarket.com/trading/fees):

| Category | Taker fee rate | `fee_rate_bps` |
|----------|:--------------:|:--------------:|
| Crypto | 7% | 700 |
| Economics / Culture / Weather / Other | 5% | 500 |
| Finance / Politics / Mentions / Tech | 4% | 400 |
| Sports | 3% | 300 |
| Geopolitics | 0% | 0 |
| Unknown / missing category | 5% (conservative fallback) | 500 |

Discover a market's rate programmatically:

```bash
curl "https://api.polysimulator.com/v1/fee-rate?token_id=7132104567925221..."
# crypto market → {"base_fee": 1000, "fee_rate_bps": 700}
# geopolitics market → {"base_fee": 0, "fee_rate_bps": 0}
```

The response is a **two-field contract**:

- `base_fee` mirrors **Polymarket's legacy base-fee parameter** (observed
  live: `1000` on fee-charging markets — sports, politics and crypto alike —
  and `0` on fee-free markets). It tells you *whether* fees are enabled, not
  the rate. Ported bots that read the PM field see byte-compatible PM
  behavior.
- `fee_rate_bps` is the PolySimulator extra field carrying the **effective
  per-category taker rate actually charged**, in basis points (the table
  above). Use this one for fee math.

`token_id` is required, and errors mirror Polymarket's live responses
verbatim: a missing or malformed `token_id` returns
`400 {"error": "Invalid token id"}`; a well-formed token that resolves to
no synced market returns `404 {"error": "fee rate not found for market"}`.

---

## Makers Pay Zero — Classified by Marketability

Matching Polymarket V2, **takers pay the fee and makers pay nothing**:

- **Market orders, FOK, FAK/IOC** — always taker.
- **GTC limit orders that are marketable at placement** (a BUY at/above
  the live ask, or a SELL at/below the live bid) — **taker**. The order
  crosses the book the moment you place it; the ~1-second matching-loop
  latency doesn't make it a maker. Swapping FOK market orders for
  marketable GTC limits does *not* dodge the fee.
- **GTC limit orders that genuinely rest** (placed away from the touch and
  filled later when the market moves into them) — **maker, zero fee**.

In addition to the marketability check at placement, any GTC fill that
happens **within ~3 seconds of the order's creation** is classified as a
taker fill (the window is operator-tunable). An order that fills that fast
was effectively marketable when you placed it — the matching loop simply
took a cycle or two to get to it — so the few-second latency cannot be used
to dodge the fee. Orders that rest longer than the window before filling
keep maker (zero-fee) treatment.

Partial fills follow Polymarket's split treatment: the slice that crosses
the book when your marketable order arrives pays the taker fee, while later
fills of the resting remainder are maker (zero-fee).

`fee_rate_bps` on `/v1/data/trades` rows reports the rate actually applied
to each fill: the category rate in bps for taker fills, `0` for fee-free
fills (maker fills, geopolitics markets, emergency exits).

---

## Fee-Free Fills

| Case | Why |
|------|-----|
| Maker fills (genuinely resting GTC) | Matches Polymarket V2 — makers never pay |
| Geopolitics markets | 0% category rate on Polymarket's schedule |
| Emergency-exit fills | Break-even safety-valve exits on stuck/expired markets — charging would turn the recovery into a loss |

---

## Documented Divergences from Polymarket

| Polymarket | PolySimulator | Why |
|------------|---------------|-----|
| Maker **rebate** (25% of taker fees; 20% on crypto) redistributed to makers | **No rebate** — makers simply pay zero | The rebate is PM's redistribution of real collected fees; the simulator has no fee pool to fund it from |
| Per-market `feesEnabled` flag — fees only charged on fee-enabled markets | Category schedule applied to **all** markets | The simulator doesn't sync PM's per-market fee flags (yet); the per-category schedule is applied uniformly |

---

## Next Steps

- [Placing Orders](/trading/placing-orders) — fee field on fill responses
- [Slippage Protection](/trading/slippage-protection#fees) — fees in the execution model
- [Trade History](/account/trade-history) — auditing debited fees per fill
