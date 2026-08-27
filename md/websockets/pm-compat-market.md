# Polymarket-compat WebSocket

Source: /websockets/pm-compat-market

# Polymarket-compatible WebSocket

```http
WS /v1/ws/market   # public market channel
WS /v1/ws/user     # auth-gated user channel
```

These routes mirror Polymarket's CLOB WS contract field-for-field so an
SDK port of Polymarket (py-clob-client, etc.) can be pointed at PolySim
by changing only the host. No event-handler rewrites required.

If you're starting fresh, the polysim-native
[Price Feed](/websockets/price-feed) is more compact and slightly cheaper
to parse. The PM-compat layer exists for SDK porters — pick this route
when you have an existing `book` / `price_change` event consumer and
don't want to re-key on `condition_id`.

---

## Subscribe message

The subscribe key differs by channel, matching Polymarket:

- **`/ws/market`** keys by **CLOB token id** (`assets_ids` — long decimal
  strings, two per binary market, one per outcome). This is PM's market
  channel contract.
- **`/ws/user`** keys by **condition id** (`markets`). This is PM's user
  channel contract — *"the user channel subscribes by condition IDs
  (market identifiers), not asset IDs."* A faithful PM SDK port sends
  `markets` to its user channel.

The polysim backend accepts either field on both routes (reverse-resolving
token ↔ condition transparently), so a single client can use whichever it
already has.

```json
// /ws/market — PM keys by token id:
{
  "type": "market",
  "assets_ids": [
    "72936048731589292555781174533757608024096898681344338414570549242843090464013",
    "84876458999851945121196253324884098156932869069716020095394562124131470571066"
  ]
}
```

```json
// /ws/user — PM keys by condition id:
{
  "type": "user",
  "markets": ["0xabc123...condition_id"],
  "auth": {"apiKey": "ps_live_..."}
}
```

| Field | Type | Description |
|---|---|---|
| `type` | string | `"market"` for `/ws/market`, `"user"` for `/ws/user`. Both routes also accept `"subscribe"` for symmetry with polysim-native clients. |
| `assets_ids` | array | List of CLOB token ids. The PM-canonical field for **`/ws/market`**. Capped at 50 per connection. |
| `markets` | array | List of condition ids (resolves both yes/no tokens server-side). The PM-canonical field for **`/ws/user`** (PM's user channel keys by condition id). Also accepted on `/ws/market` as a convenience; the markets list is capped at 25 (= 50 tokens). |
| `operation` | string | `"subscribe"` or `"unsubscribe"` — PM's dynamic-subscription verb for mutating subscriptions without reconnecting (see [Dynamic subscribe/unsubscribe](#dynamic-subscribe-unsubscribe) below). |
| `auth` | object | Optional inline auth; required on `/ws/user`. Fields documented below. |

### Auth block

```json
{
  "auth": {
    "apiKey": "ps_live_..."
  }
}
```

| Field | Required | Description |
|---|---|---|
| `apiKey` | yes (on `/ws/user`); optional on `/ws/market` | Your `ps_live_...` raw API key. Same key you use for HTTP requests. |
| `secret` | no | Accepted for PM-shape compatibility — ignored by PolySim (we are single-secret). |
| `passphrase` | no | Accepted for PM-shape compatibility — ignored. |

Alternative: pass `?token=` on connect (the polysim-native auth
model). The JWT comes from `POST /v1/keys/ws-token` and expires in 60
seconds. Pick exactly one auth path — passing both is fine but the
JWT takes precedence.

---

## Events emitted

All four PM event types are emitted with PM's exact field names:

### `book` — full L2 snapshot

```json
{
  "event_type": "book",
  "asset_id": "72936048731589292555781174533757608024096898681344338414570549242843090464013",
  "market": "0xabc123...",
  "bids": [
    {"price": "0.55", "size": "100"},
    {"price": "0.54", "size": "250"}
  ],
  "asks": [
    {"price": "0.56", "size": "100"}
  ],
  "hash": "0x...",
  "tick_size": "0.01",
  "timestamp": "1715518800123"
}
```

Emitted once per token on subscribe (with the current CLOB orderbook
snapshot). Subsequent updates flow through `price_change` /
`best_bid_ask` — the `book` event is not re-emitted on every level change.

### `price_change` — orderbook delta

```json
{
  "event_type": "price_change",
  "asset_id": "...",
  "market": "0xabc123...",
  "price_changes": [
    {
      "asset_id": "...",
      "price": "0.55",
      "best_bid": "0.55",
      "best_ask": "0.56",
      "size": "0",
      "side": "",
      "hash": ""
    }
  ],
  "timestamp": "1715518800123",
  "hash": ""
}
```

  **Two PM deviations:**
  - PolySim adds a convenience **top-level `asset_id`** that PM's
    `price_change` does not send (PM puts `market` / `price_changes[]` /
    `timestamp` / `event_type` at the top level; the asset id lives only
    inside each `price_changes[]` entry).
  - The inner `size`, `side`, and `hash` fields are **stubbed**
    (`"0"` / `""` / `""`). PolySim's price cache is a top-of-book-only
    payload with no per-level deltas or rolling book hash, so these are
    emitted for schema parity but carry no data. PM populates them.

### `last_trade_price` — fill broadcast

```json
{
  "event_type": "last_trade_price",
  "asset_id": "...",
  "market": "0xabc123...",
  "price": "0.55",
  "side": "BUY",
  "size": "0",
  "fee_rate_bps": "0",
  "timestamp": "1715518800123"
}
```

### `best_bid_ask` — top-of-book change

```json
{
  "event_type": "best_bid_ask",
  "asset_id": "...",
  "market": "0xabc123...",
  "best_bid": "0.55",
  "best_ask": "0.56",
  "timestamp": "1715518800123"
}
```

  **Polymarket Compatibility & Null-for-Unknown Semantics:**
  - PolySim omits the `spread` field that PM's `best_bid_ask` carries
    (compute it yourself as `best_ask - best_bid` if you need it).
  - PM gates `best_bid_ask` (plus `new_market` / `market_resolved`)
    behind `custom_feature_enabled: true` in the subscribe message.
    PolySim ignores `custom_feature_enabled` and emits `best_bid_ask`
    **unconditionally** — you do not need to set the flag.
  - **Null-for-unknown**: For non-first outcome tokens (e.g. NO tokens), if
    outcome-specific orderbook data is unavailable, the `best_bid_ask` frame
    is **omitted** rather than fabricating prices from other outcomes or
    inverting YES. In `price_change` frames, unobserved top-of-book fields are
    emitted as empty strings `""`.

All prices are **strings** (no JSON numbers) to match PM verbatim and
avoid float-precision drift.

---

## Error frames

The PM-compat layer emits structured error frames (PM does the same):

```json
{
  "event_type": "error",
  "error": "AUTH_REQUIRED",
  "message": "/v1/ws/user requires authentication. Pass {\"auth\": {\"apiKey\": \"ps_live_...\"}} in the subscribe message, or use ?token= on connect."
}
```

| `error` code | Trigger | Connection closes? |
|---|---|---|
| `INVALID_JSON` | Subscribe frame isn't valid JSON | no — send a corrected frame |
| `INVALID_SUBSCRIBE` | Subscribe payload missing `assets_ids` or has wrong type | no |
| `EMPTY_SUBSCRIBE` | `assets_ids` parsed but was empty | no |
| `UNKNOWN_ASSET` | One or more token ids didn't resolve to a known market — others (if any) ARE subscribed | no |
| `MAX_ASSETS_EXCEEDED` | Cumulative subscriptions on this connection would exceed 50 — none of the new tokens were added | no — send `unsubscribe` first or open a new connection |
| `AUTH_REQUIRED` | `/ws/user` subscribe without valid `auth.apiKey` or `?token=` | yes (close code 1008) |
| `AUTH_INVALID` | An `auth.apiKey` was sent but didn't resolve to an active key | yes (close code 1008) |
| `MAX_WS_EXCEEDED` | Per-user WebSocket connection cap reached (mixes native + PM-compat sockets) | yes (close code 4002) |
| `UNKNOWN_FRAME` | Sent frame `type` not recognised | no |

`UNKNOWN_ASSET` and `INVALID_SUBSCRIBE` are non-fatal — the connection
stays open so you can send a corrected subscribe. The three rows marked
"yes" above are terminal: the server sends the error frame, then closes
the socket so the client knows to re-handshake (with corrected auth) or
back off.

---

## Dynamic subscribe/unsubscribe

PM SDKs mutate subscriptions **without reconnecting** by sending an
`operation` frame (the initial subscribe uses `type`; subsequent updates
use `operation`). PolySim accepts the same shape:

```json
// Add tokens (on /ws/market):
{"assets_ids": [""], "operation": "subscribe"}

// Remove tokens (on /ws/market):
{"assets_ids": [""], "operation": "unsubscribe"}

// On /ws/user, mutate by condition id instead:
{"markets": ["0xabc123...condition_id"], "operation": "subscribe"}
```

The per-connection 50-token cap is cumulative across all subscribe
operations — exceeding it returns a `MAX_ASSETS_EXCEEDED` error frame
(none of the new tokens are added). Send `unsubscribe` to free room first.

---

## Ping/pong

PolySim accepts **both** heartbeat shapes:

```json
// JSON shape:
{"type": "ping"}
```

```
// Plain-string shape (PM's exact form):
ping
```

In **both** cases the server replies with a **JSON** pong frame:

```json
{"event_type": "pong", "ts": 1715518800123}
```

  **Deviation from PM for porters:** Polymarket replies to a plain-string
  `PING` with a plain-string `PONG`. PolySim always replies with the JSON
  `{"event_type": "pong", "ts": }` frame — even when you sent a
  plain-string `ping`. A PM bot waiting for a literal `PONG` string never
  sees it; key your heartbeat handler off the JSON `pong` frame instead.

The PM-compat shape uses `event_type` (not `type`) on the pong frame
for consistency with the rest of the protocol.

---

## Complete example

```python Python (websockets)
import asyncio
import json
import websockets

API_KEY = "ps_live_..."
ASSET_ID = "72936048731589292555781174533757608024096898681344338414570549242843090464013"

async def stream():
    async with websockets.connect("wss://api.polysimulator.com/v1/ws/market") as ws:
        await ws.send(json.dumps({
            "type": "market",
            "assets_ids": [ASSET_ID],
            "auth": {"apiKey": API_KEY},  # optional on /ws/market; required on /ws/user
        }))
        async for raw in ws:
            evt = json.loads(raw)
            kind = evt.get("event_type")
            if kind == "book":
                print(f"BOOK {evt['asset_id'][:16]}... bids={len(evt['bids'])} asks={len(evt['asks'])}")
            elif kind == "price_change":
                for change in evt.get("price_changes", []):
                    print(f"PRICE {change['asset_id'][:16]}... -> {change['price']}")
            elif kind == "last_trade_price":
                print(f"TRADE {evt['asset_id'][:16]}... {evt['side']} @ {evt['price']}")
            elif kind == "best_bid_ask":
                print(f"BBO {evt['asset_id'][:16]}... bid={evt['best_bid']} ask={evt['best_ask']}")
            elif kind == "error":
                print(f"ERROR: {evt['error']} - {evt['message']}")

asyncio.run(stream())
```

For py-clob-client port-overs, the only change is the WS URL:

```python
# Before (Polymarket):
WS_URL = "wss://ws-subscriptions-clob.polymarket.com/ws/market"

# After (PolySim):
WS_URL = "wss://api.polysimulator.com/v1/ws/market"
```

Event handlers — including the `event_type` field name and PM-style
stringified prices — work without modification.

---

## What's NOT in the PM-compat layer (yet)

- **PM's `order` PLACEMENT / UPDATE / CANCELLATION events on `/ws/user`** —
  **fill (`trade`) events ARE emitted**: when one of
  your orders fills on a subscribed market, `/ws/user` pushes PM's
  user-channel `trade` frame (`event_type: "trade"`, `type: "TRADE"`,
  `status: "MATCHED"`, unix-string timestamps). Divergences: paper
  trades terminate at `MATCHED` (no `MINED`/`CONFIRMED` lifecycle —
  no chain), `maker_orders` is empty, and `owner`/`trade_owner` are
  empty strings. PM's `order` placement/cancel events are still NOT
  emitted — poll `GET /v1/data/orders` for order state, or use the
  polysim-native [Execution Feed](/websockets/execution-feed)
  (`/v1/ws/executions`).
- **L2 signature auth** — PM's full L2 contract requires
  `POLY_ADDRESS` / `POLY_SIGNATURE` / `POLY_TIMESTAMP` / `POLY_API_KEY` /
  `POLY_PASSPHRASE`. PolySim is single-secret — we accept the `apiKey`
  field and ignore the others. SDK code that calls a PM signer to
  build the L2 block doesn't need changes; the signer's output is
  simply ignored.

---

## Compatibility matrix

| Feature | Polymarket | PolySim PM-compat |
|---|:-:|:-:|
| `wss://.../ws/market` URL path | yes | yes (under `/v1/`) |
| `/ws/market` keys by `assets_ids` (token ids) | yes | yes |
| `/ws/user` keys by `markets` (condition ids) | yes | yes |
| `operation` dynamic subscribe/unsubscribe | yes | yes |
| Inline `auth.apiKey` | yes | yes |
| L2 signature (`secret`/`passphrase`) | required | ignored (single-secret) |
| `book` event shape | yes | yes |
| `price_change` event shape | yes | yes (adds a top-level `asset_id` PM omits; inner `size`/`side`/`hash` stubbed — TOB-only cache) |
| `last_trade_price` event shape | yes | yes |
| `best_bid_ask` event shape | yes | yes (no `spread` field; emitted unconditionally — PM gates it behind `custom_feature_enabled`) |
| Plain-string `PING` → `PONG` | yes | accepted, but reply is JSON `{event_type:"pong"}` (not plain `PONG`) |
| `tick_size_change` event | yes | not on this channel — read it on the SSE `/prices/stream` feed instead (see [Price Feed](/websockets/price-feed#tick-size-changes)) |
| Per-user execution events on `/ws/user` | yes | use `/v1/ws/executions` for now |
