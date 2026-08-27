# Python SDK

Source: /sdk

# Python SDK

The official Python client is published on PyPI as
[`polysimulator`](https://pypi.org/project/polysimulator/) — `pip install polysimulator`.

## Start here: `PolySimClient`

**New to PolySimulator? Use the native client.** It's the cleanest path — sync **and**
async, WebSocket + SSE streaming, pagination iterators, typed exceptions — and the
[**Quickstart**](/quickstart) places your first trade in ~5 lines. That's the one path
most developers want; jump to [Install](#install) below.

```python
from polysim_sdk import PolySimClient   # pip install polysimulator

with PolySimClient() as client:                              # POLYSIM_API_KEY from env
    market = client.list_markets(limit=1)[0]  # an actively-traded market
    fill = client.place_order(market_id=market["condition_id"], side="BUY",
                              outcome="Yes", quantity=10, order_type="market", price="0.99")
    print(fill["status"])                                    # FILLED
```

  **Porting an existing bot?** The package ships two **drop-in mirrors** so you don't
  rewrite your strategy — use one **only** if you already have a bot on that library;
  otherwise stick with `PolySimClient` above.

  | You already use… | Import | How to port |
  | --- | --- | --- |
  | [`py-clob-client`](https://github.com/Polymarket/py-clob-client) **v1** | `polysim_clob_client` | Drop-in mirror — change the import path, host, and auth call. See the [CLOB migration map](/concepts/clob-compatibility). |
  | Polymarket's unified **[py-sdk](https://pypi.org/project/polymarket-client/)** (`polymarket-client`) | `polysim_polymarket` | Paper-mode mirror of `PublicClient` / `SecureClient` — swap the import prefix + host + auth. See the [py-sdk Mirror overview](/sdk-polymarket) + [compatibility](/concepts/py-sdk-compatibility). |

Because PolySimulator is paper trading, there is **no on-chain anything** — no private
key, no `chain_id`, no signing, no USDC allowances. The package depends only on
`httpx` and `websockets`, and auth is a single `ps_live_…` key sent as `X-API-Key`.

  
    `pip install polysimulator` — the published package, with the complete README and method reference.
  
  
    The full `py-clob-client` → PolySimulator migration map.
  

## Install

Targets Python 3.10+.

```bash
pip install polysimulator
```

  The install name is **`polysimulator`**; the import name stays **`polysim_sdk`**
  (the same way `pip install pillow` gives you `import PIL`). The older `polysim-sdk`
  package name is now a thin alias that installs `polysimulator` for you, so existing
  installs keep working unchanged.

Provide your key via the `POLYSIM_API_KEY` environment variable (recommended) or pass
it to the constructor. Point at staging with `POLYSIM_BASE_URL`:

```bash
export POLYSIM_API_KEY="ps_live_…"
# optional — defaults to https://api.polysimulator.com
export POLYSIM_BASE_URL="https://staging-api.polysimulator.com"
```

## Quickstart — native `polysim_sdk`

```python
from polysim_sdk import PolySimClient

with PolySimClient() as client:            # POLYSIM_API_KEY from env
    me = client.me()
    print(me["balance"])                   # your trading balance

    market = client.list_markets(limit=1)[0]
    cid = market["condition_id"]

    fill = client.place_order(
        market_id=cid,
        side="BUY",
        outcome="YES",
        quantity=10,
        order_type="market",
        price="0.99",                      # slippage cap: the worst price you'll accept.
                                           #   For a YES market BUY, 0.99 is "fill at any price"
                                           #   (a YES share can never cost more than $1).
    )
    print(f"filled {fill['status']} @ {fill.get('price')}")
```

Async is the same API with `await`:

```python
import asyncio
from polysim_sdk import AsyncPolySimClient

async def main():
    async with AsyncPolySimClient() as client:
        print((await client.me())["balance"])

asyncio.run(main())
```

The native client also ships WebSocket price/execution streams (`polysim_sdk.ws`), an
SSE underlying-spot firehose (`polysim_sdk.sse`), Up/Down helpers (`polysim_sdk.updown`),
and pagination iterators (`polysim_sdk.pagination`). The
[full README on PyPI](https://pypi.org/project/polysimulator/) documents every surface.

The native client also exposes `client.simulation` for the dark
`/v1/simulation` surface (`fill`, `coverage`, `book`,
`create_backtest` / `wait`, list/get/cancel). Methods exist so typed
clients can generate; the live router stays 404 until
`FEATURE_SIMULATION_API_ENABLED` is on. See [Simulation](/simulation).

## Migrating a `py-clob-client` bot

Change **three things** and delete the on-chain prelude — method names, argument shapes,
and return shapes all stay the same:

```python
from polysim_clob_client.client import ClobClient
from polysim_clob_client.clob_types import OrderArgs

client = ClobClient(
    host="https://api.polysimulator.com",
    key="ps_live_…",                       # your PolySimulator API key — no wallet, no chain_id
)

order = client.create_order(OrderArgs(token_id=tid, price=0.55, size=10, side="BUY"))
resp = client.post_order(order)
```

The full per-method mapping (mirror / adapt / stub-noop), the `token_id` ↔ market+outcome
seam, and the read-vs-write parity rules are documented in
[CLOB compatibility](/concepts/clob-compatibility).

## Rate limits & pacing

The client paces itself (a 50 ms floor between requests) and backs off on `429`/`5xx`
using `Retry-After`. Authoritative tier limits come live from `client.tiers()` — see
[Rate limits](/concepts/rate-limits) for the per-tier table.

## Errors

Catch `PolySimError` to handle every SDK-raised error (`PolyApiException`, the drop-in
alias, is the same class):

```python
from polysim_sdk.exceptions import PolySimError, RateLimitError, ValidationError

try:
    fill = client.place_order(...)
except RateLimitError as exc:
    print(f"backing off for {exc.retry_after}s")
except ValidationError as exc:
    print(f"bad request: {exc.code} → {exc}")
except PolySimError as exc:
    print(f"other API error ({exc.status_code}): {exc}")
```
