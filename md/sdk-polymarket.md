# Polymarket py-sdk Mirror

Source: /sdk-polymarket

# Polymarket py-sdk Mirror

Polymarket ships a newer **unified Python SDK**, published to PyPI as
[`polymarket-client`](https://pypi.org/project/polymarket-client/) and imported as
`polymarket`. It folds the CLOB, gamma (markets/events), data, RFQ, and
realtime-stream surfaces into one client and returns **typed pydantic models**
(`OrderBook`, `Market`, `LastTradePrice`, …) instead of raw dicts.

`polysim_polymarket` is our **paper-mode mirror** of that SDK — the **v2 mirror**
(the [`polysim_clob_client`](/concepts/clob-compatibility) drop-in is v1). It
mirrors the unified client family in paper mode: **sync and async**
`PublicClient` and `SecureClient`, the authenticated trading write surface, the
account/auth/order reads, the on-chain methods (as paper no-ops), the
rewards/builder/RFQ stubs, and the core realtime streams — returning typed
pydantic models so the port stays mechanical.

  This surface mirrors [`polymarket-client`](https://pypi.org/project/polymarket-client/)
  **`0.1.0b8`** in **paper mode**. Parity is re-locked against this exact pin; when
  the pin bumps, the [compat matrix](/concepts/py-sdk-compatibility#compat-matrix)
  bumps with it. It is a **paper-trading** mirror, not a drop-in for *everything*
  py-sdk does — read the
  [sim→real seam contract](/concepts/py-sdk-compatibility#the-simreal-seam-contract)
  and [compat matrix](/concepts/py-sdk-compatibility#compat-matrix) before you rely
  on a surface area.

  
    `pip install polysimulator` — one install ships `polysim_sdk`, `polysim_clob_client`, and `polysim_polymarket`.
  
  
    The full sim→real seam contract, compat matrix, method reference, fidelity gaps, and stream seams.
  

## Install

```bash
pip install polysimulator
```

One install (Python 3.10+) ships all three import surfaces — `polysim_sdk`,
`polysim_clob_client`, and `polysim_polymarket` (this mirror). For the full install
walkthrough and how the three relate, see the [Python SDK overview](/sdk).

Provide your key via the `POLYSIM_API_KEY` environment variable or pass it to the
client. There is **no on-chain anything** in paper mode — no private key, no
`chain_id`, no signing, no USDC allowance. Auth is a single `ps_live_…` key.

## The port — swap prefix, host, auth

The port is the same mechanical swap as v1 — change the import **prefix**, the
**host**, and the **auth**:

```python Real Polymarket (polymarket)
from polymarket import SecureClient

client = SecureClient(private_key="0x…")       # real wallet + EIP-712 signing

book = client.get_order_book(token_id="711811…")
# py-sdk OrderBook ordering: bids ascending (best = bids[-1]),
# asks descending (best = asks[-1]).
print(book.bids[-1].price, book.asks[-1].price, book.token_id)

signed = client.create_market_order(token_id="711811…", side="BUY", amount=10)
resp = client.post_order(signed)                # on-chain-settled
print(resp.order_id, resp.status)
```

```python Paper mode (polysim_polymarket)
from polysim_polymarket import SecureClient

client = SecureClient(host="https://api.polysimulator.com", api_key="ps_live_…")

book = client.get_order_book(token_id="711811…")
# Same OrderBook ordering: bids ascending (best = bids[-1]),
# asks descending (best = asks[-1]).
print(book.bids[-1].price, book.asks[-1].price, book.token_id)

signed = client.create_market_order(token_id="711811…", side="BUY", amount=10)
resp = client.post_order(signed)                # paper-accepted, not on-chain-settled
print(resp.order_id, resp.status)
```

Everything from there on is unchanged — same method names, same keyword-only call
shapes, same model field names and types.

The `PublicClient` swap is identical, but its reads are unauthenticated — pass
only the host, no key:

```python
# real:  PublicClient()
from polysim_polymarket import PublicClient

client = PublicClient(host="https://api.polysimulator.com")

book = client.get_order_book(token_id="711811…")
print(book.bids[-1].price, book.asks[-1].price, book.token_id)
```

## Sync + async clients

Both clients have async twins — `AsyncPublicClient` and `AsyncSecureClient` —
exposing the identical surface with every per-request method an `async def`
coroutine (the `list_*` reads stay synchronous, returning an `AsyncPaginator` you
then `await`):

```python Sync
from polysim_polymarket import SecureClient

client = SecureClient(host="https://api.polysimulator.com", api_key="ps_live_…")
book = client.get_order_book(token_id="711811…")
print(book.bids[-1].price, book.asks[-1].price)
```

```python Async
import asyncio
from polysim_polymarket import AsyncSecureClient

async def main():
    async with AsyncSecureClient(api_key="ps_live_…", host="https://api.polysimulator.com") as client:
        book = await client.get_order_book(token_id="711811…")
        print(book.bids[-1].price, book.asks[-1].price)

asyncio.run(main())
```

Construct the async client with `AsyncSecureClient(api_key=…)` or
`await AsyncSecureClient.create(api_key=…)`, drive with `await`, and use
`async with AsyncSecureClient(…) as client:` for lifecycle — exactly as py-sdk's
async client does. The async twins share the same transport-free logic as the
sync clients, so behaviour is identical by construction.

## Re-exports and the one error-base divergence

The model field names and types (`OrderBook`, `OrderBookLevel`, `LastTradePrice`,
`PriceHistoryPoint`, `Market` — with its nested `.state` `MarketState` — `Page`,
`Paginator`, the trading types `OrderSide` / `OrderType` / `SignedOrder` /
`OrderResponse` / `CancelOrdersResponse`, and the RFQ types) all track py-sdk so
the swap stays mechanical. Every public name a bot imports is re-exported off the
package root, exactly as on real Polymarket — including the named errors the
surface raises (`UserInputError`, `InsufficientLiquidityError`,
`UnexpectedResponseError`), so `from polymarket import UserInputError` survives
the prefix swap unchanged.

  Like py-sdk, `MarketState` itself is **not** promoted to the package root — it's
  reached via `market.state` or `polysim_polymarket.models.MarketState`, never
  `from polysim_polymarket import MarketState`.

  **The one deliberate divergence is the error-tree *base* name.** Those three
  named errors subclass the py-clob-client-lineage `PolyException` /
  `PolyApiException` base (shared by identity with the v1 mirror) rather than
  py-sdk's `PolymarketError`. So a bot that catches the broad base by name needs
  `except PolyException` here vs `except PolymarketError` on real Polymarket; an
  `except UserInputError` block is identical on both.

```python
from polysim_polymarket import PublicClient, OrderBook, Environment, PolyException, Paginator
```

## Next steps

  
    What paper mode does and doesn't do, per surface area — read before you rely on anything.
  
  
    The clean native client (sync + async, WS + SSE streaming, Up/Down helpers).
