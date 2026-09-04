# Error Handling

Source: /bots/error-handling

# Error Handling

Every API response uses standard HTTP status codes with structured JSON error bodies.

---

## Error Response Format

By default, the `/v1/*` surface returns PolySimulator's stable two-field
envelope: `error` holds the machine-readable code and `message` holds the
human-readable description.

```json
{
  "error": "INSUFFICIENT_BALANCE",
  "message": "Account balance $12.50 insufficient for order cost $25.00"
}
```

| Field | Type | Description |
|-------|------|-------------|
| `error` | string | Stable machine-readable short code, also carried in `X-Polysim-Code` (for example `INSUFFICIENT_BALANCE` or `RATE_LIMIT_EXCEEDED`). |
| `message` | string | Human-readable description; do not branch on this prose. |

For stable error handling, branch on `error` or the identical
`X-Polysim-Code` header, never on `message`. The `X-Request-Id` header
echoes the request id for log correlation.

This is a documented PolySimulator extension to Polymarket's single-field
`{"error": ""}` shape. The runtime envelope did not change when
this documentation was corrected on 2026-08-29.

**`402 UPGRADE_REQUIRED` is the one enriched response**: its body also
adds `feature_key` and `upgrade_url` so SDKs can render an upsell flow
without an extra round-trip.

```json
// 402 UPGRADE_REQUIRED — feature_key + upgrade_url at the root
{
  "error": "UPGRADE_REQUIRED",
  "feature_key": "wallets.sandbox_baseline",
  "upgrade_url": "/pricing"
}
```

For the runtime allowlist gate (`ACCESS_RESTRICTED` — an already-issued
key whose account isn't on the API v1 allowlist, or is flagged / under
review), the body uses the same two-field envelope:

```json
// 403 ACCESS_RESTRICTED (not on the allowlist / flagged account)
// + headers: X-Polysim-Code: ACCESS_RESTRICTED
{"error": "ACCESS_RESTRICTED", "message": "API access restricted. Contact support to request access."}
```

The residual **key-issuance** codes use the same shape. Default open
beta self-serves keys: free callers get a read-only key, paying Pro /
Pro+ callers get a trade-capable key. `403 TIER_REQUIRES_UPGRADE` is
the expected refusal when a free caller requests `trade`.
`CLOSED_BETA` / `API_PRO_COMING_SOON` remain valid residual codes for
an emergency close. The `feature_key` / `upgrade_url` hints are body
fields **only on 402** responses, not these 403s. See
[Open Beta Errors](#open-beta-errors) below.

  **Verbose body opt-in.** Send `X-Polysim-Verbose: true` on any
  request to add diagnostic fields:
  ```json
  {"error": "INVALID_KEY", "message": "Invalid API key",
   "details": null, "request_id": "a1b2c3d4-..."}
  ```

---

## HTTP Status Codes

| Code | Meaning | Retry? | Action |
|------|---------|--------|--------|
| `200` | Success | — | Process response |
| `400` | Bad request | No | Fix request payload |
| `401` | Invalid or missing API key | No | Check `X-API-Key` header |
| `403` | Insufficient permissions | No | Check key scopes |
| `404` | Resource not found | No | Verify market ID or order ID |
| `409` | Conflict (duplicate idempotency key) | No | Use original response |
| `422` | Validation error | No | Fix input fields |
| `429` | Rate limited | **Yes** | Wait for `Retry-After` header |
| `500` | Internal server error | **Yes** | Retry with backoff |
| `502` | Upstream error | **Yes** | Retry with backoff |
| `503` | Service unavailable | **Yes** | Retry with backoff |

---

## Common Error Codes

### Trading Errors

This table is the **canonical trading error-code reference** — other
trading pages link here rather than restating the codes.

| Error Code | HTTP | Description |
|-----------|------|-------------|
| `INSUFFICIENT_BALANCE` | 400 | Not enough funds for order |
| `INSUFFICIENT_POSITION` | 400 | Sell rejected — your position size for the chosen `token_id` is smaller than the requested `quantity`. PolySimulator does not support naked shorts: every SELL must be backed by an existing position. See **No naked shorts** below for the two-sided-quote workaround. |
| `INSUFFICIENT_LIQUIDITY` | 400 | The order-book walk cannot source the requested quantity across resting liquidity. |
| `NO_EXECUTABLE_DEPTH` | 400 | Refused when order-book depth enforcement is active and the fill would rely on non-depth fallback tiers. Trades only execute against live Polymarket book depth. |
| `STALE_QUOTE` | 400 | Fill rejected when the resolved price quote age exceeds the configured maximum age threshold (`TRADE_QUOTE_MAX_AGE_MS` when enforcement is enabled). Retry with a fresh quote. |
| `INVALID_QUANTITY` | 400 | Non-positive `quantity` rejected by the execution engine. (A schema-level out-of-range value is caught earlier by Pydantic as `422 VALIDATION_FAILED` — see HTTP Status Codes above.) |
| `PRICE_REQUIRED` | 400 | Market order submitted without the required `price` (worst-price limit). |
| `FOK_ORDER_NOT_FILLED_ERROR` | 400 | The order could not fill entirely at or beyond your worst-price limit (BUY: best price above your cap; SELL: best price below your floor). This is the worst-price rejection for market / FOK orders — there is **no** `409 LIMIT_PRICE_NOT_MET` code. |
| `INVALID_ORDER_PAYLOAD` | 400 | Body shape invalid (PM-shape `/v1/order`): bad maker/taker amounts, missing `tokenId`, or unsupported `side` |
| `INVALID_ORDER_MIN_TICK_SIZE` | 400 | Limit price doesn't conform to the market's tick size (0.1 / 0.01 / 0.001 / 0.0001). Round to a multiple of `GET /v1/tick-size/{token_id}`. Mirrors Polymarket's behaviour exactly. |
| `UNSUPPORTED_ORDER_TYPE` | 400 | `order_type=IOC` on `POST /v1/clob/order` is rejected (use `GTC` to rest or `FOK` for immediate-or-fail). Code is in the `X-Polysim-Code` header. |
| `MARKET_NOT_FOUND` | 404 | Unknown `market_id` (or unknown `tokenId` on PM-shape `/v1/order`). Verify with `GET /v1/markets-by-token/{token_id}`. |
| `MARKET_CLOSED` | 400 | Market is resolved or inactive |
| `ORDER_NOT_FOUND` | 404 | Unknown `order_id` |
| `ORDER_NOT_CANCELLABLE` | 400 | The order is already `FILLED`, `CANCELLED`, or `EXPIRED` and can't be cancelled. |
| `DUPLICATE_CLIENT_ORDER_ID` | 409 | A new order reused a `client_order_id` already bound to a different order. |
| `IDEMPOTENCY_KEY_REUSE` | 409 | The same `Idempotency-Key` was replayed with a **different** request body. (An identical replay instead returns the original order — see Idempotency below.) |
| `IDEMPOTENCY_CONFLICT_PENDING` | 409 | The same `Idempotency-Key` is still being processed; carries `Retry-After: 1`. |
| `EXECUTION_ERROR` | 500 | Server-side error during fill — report with `request_id` |

  There is no `409 LIMIT_PRICE_NOT_MET`, `409 IDEMPOTENCY_CONFLICT`,
  `CANNOT_CANCEL`, or `HTTP_409` trading code — those names appeared in
  earlier drafts but are not emitted by the engine. Use the codes above.

### Deadline, busy, and concurrency errors

Order writes are serialized per **user**, not per API key. A second write
waits briefly for the first; if the lock is still held, it returns
`409 TRADE_IN_PROGRESS` with `Retry-After: 1`. Separate keys for the same
user do not bypass this lock. Wait, then retry the same logical request
with the same `client_order_id`.

| Error Code | HTTP | Persisted? | Client action |
|-----------|------|------------|---------------|
| `TRADE_IN_PROGRESS` | 409 | No new write started | Wait for `Retry-After`, then retry the same request. |
| `SERVER_BUSY` | 503 | Persistence check confirmed no row | Safe to retry. For order placement, reuse the same `client_order_id` so idempotency deduplicates a late result. |
| `DEADLINE_OVERSHOT_BUT_PERSISTED` | 503 | **Yes** | **Do not re-place.** Read `X-Polysim-Order-Id` and body `source`. For `source: "pending"`, poll `GET /v1/orders/{order_id}`. For `source: "filled"`, fetch `GET /v1/orders` and match your `client_order_id`; direct numeric ids can collide across the pending and filled tables. |
| `PERSISTENCE_UNKNOWN` | 503 | Unknown | **Do not auto-retry.** Fetch recent `GET /v1/orders` results and scan for the same `client_order_id` before deciding whether to re-place. This response intentionally has no `Retry-After`. |

```python
import time

code = response.headers.get("X-Polysim-Code")
if code in {"TRADE_IN_PROGRESS", "SERVER_BUSY"}:
    time.sleep(int(response.headers.get("Retry-After", "1")))
    # Repeat the original request with the SAME client_order_id.
elif code == "DEADLINE_OVERSHOT_BUT_PERSISTED":
    order_id = response.headers["X-Polysim-Order-Id"]
    source = response.json().get("source")
    # Do not POST again. Poll by id only for a pending row; otherwise scan
    # GET /v1/orders and match the original client_order_id.
elif code == "PERSISTENCE_UNKNOWN":
    # Do not POST again until GET /v1/orders confirms no matching order.
    pass
```

### Order status values

`OrderResponse.status` (and the `status` filter on `GET /v1/orders`) use
the PolySimulator-native enum — note the **double-L** `CANCELLED`:

| Status | Meaning |
|--------|---------|
| `PENDING` | Limit order resting on the book, not yet filled. |
| `FILLED` | Order matched and executed. |
| `CANCELLED` | Order cancelled (by you, a FOK/FAK non-fill, or the auto-cancel safeguard). Native spelling is **double-L** `CANCELLED`. |
| `EXPIRED` | Order expired (e.g. market resolved while resting). |
| `REJECTED` | Per-entry batch failure (see [Batch Orders](/trading/batch-orders)). |
| `ERROR` | Per-entry batch internal error (batch only). |

SDKs ported from Polymarket read the PM-shape `ORDER_STATUS_*` enum from
`GET /v1/data/orders` instead — where the cancelled member is the
**single-L** `ORDER_STATUS_CANCELED` (PM's exact spelling). The native
`GET /v1/orders` path uses double-L `CANCELLED`; the PM-shape
`GET /v1/data/orders` path uses single-L `ORDER_STATUS_CANCELED`. See
[CLOB Compatibility](/concepts/clob-compatibility).

#### No naked shorts — `INSUFFICIENT_POSITION` explained

PolySimulator (and Polymarket itself) requires every SELL order to be backed by an
existing position in that exact `token_id`. There is no margin, no borrow, no
synthetic short. If you try to sell shares you don't hold, the order is rejected
with `400 INSUFFICIENT_POSITION` (header `X-Polysim-Code: INSUFFICIENT_POSITION`).

For binary markets (every `/markets/{id}` with two outcomes), the standard
market-maker idiom is a **two-sided buy** rather than a buy + a short:

```python
# Wrong — naked short on the "Down" side:
sell_down = post("/v1/order", json={"token_id": down_tok, "side": "SELL", "quantity": 100, "price": 0.51})
# → 400 INSUFFICIENT_POSITION (you don't hold any Down shares)

# Right — buy both sides instead. Up + Down ≈ 1.00, so total notional is similar
# to a two-sided quote, and you keep the spread on whichever side fills first:
buy_up   = post("/v1/order", json={"token_id": up_tok,   "side": "BUY", "quantity": 100, "price": 0.49})
buy_down = post("/v1/order", json={"token_id": down_tok, "side": "BUY", "quantity": 100, "price": 0.49})
```

After a buy fills you accumulate position; subsequent SELLs against that position
are accepted up to the held quantity. Use `GET /v1/account/positions` to check
your inventory per `token_id` before submitting a SELL.

### Authentication Errors

| Error Code | HTTP | Description |
|-----------|------|-------------|
| `MISSING_AUTH` | 401 | No `Authorization: Bearer …` or `X-API-Key` on a route that accepts either |
| `MISSING_API_KEY` | 401 | `X-API-Key` (or legacy `POLY_API_KEY`) missing on a key-only route |
| `INVALID_KEY` | 401 | Key doesn't exist or is revoked |
| `INVALID_TOKEN` | 401 | Bearer JWT is malformed, missing `sub`, or fails signature verification |
| `KEY_EXPIRED` | 401 | API key has expired |
| `KEY_DEACTIVATED` | 401 | Key was administratively disabled |
| `TOKEN_EXPIRED` | 401 | Bearer JWT past its `exp` claim |
| `INSUFFICIENT_PERMISSION` | 403 | Key missing the required permission (e.g. `trade` for order endpoints) |
| `ACCESS_RESTRICTED` | 403 | An already-issued key (or Bearer JWT) is not on the API v1 access list (or the account is flagged / under review). Returned on authenticated `/v1/*` requests. Body is PM-shape; check the `X-Polysim-Code` header. |
| `CLOSED_BETA` | 403 | Residual emergency issuance close on `POST /v1/keys` / `POST /v1/keys/bootstrap`. **Not** the default open-beta outcome. Machine code in `X-Polysim-Code`. |
| `API_PRO_COMING_SOON` | 403 | Residual cohort-rollout code. Default open beta does not return this for paying Pro / Pro+. |
| `TIER_REQUIRES_UPGRADE` | 403 | Free caller requested `trade`. Free keys are read-only. |
| `UPGRADE_REQUIRED` | 402 | Pro-tier cap reached (sandbox count, etc.). **Only** 402 carries `feature_key` + `upgrade_url` at the root of the response body. |

### Rate Limit Errors

The backend emits **two distinct 429 codes** (in the `X-Polysim-Code`
header). Both are retryable and both carry `Retry-After` — branch on
either, or simply treat any 429 as a back-off signal:

| Error Code | HTTP | Description |
|-----------|------|-------------|
| `RATE_LIMIT_EXCEEDED` | 429 | The per-tier in-process concurrency cap (all `/v1/*` paths) or the IP / per-key request-rate limiter. Retry after `Retry-After`. |
| `RATE_LIMITED` | 429 | The cross-worker **trade-concurrency** limiter on the three trade-write paths (`POST /v1/orders`, `/v1/orders/batch`, `/v1/clob/order`) or the per-account order-rate limiter on session-JWT trade endpoints (`POST /trade`, `POST /limit-order`). Body also carries retry metadata. Retry after `Retry-After`. |

  A bot that branches **only** on `RATE_LIMIT_EXCEEDED` will miss the
  `RATE_LIMITED` 429s from the trade-write paths (and vice-versa). The
  robust pattern is to back off on `resp.status_code == 429` regardless
  of which code is in the header.

The per-tier limits (authoritative source: `GET /v1/keys/tiers`):

| Tier | Req/sec | Req/min | WS conns | Max batch |
|------|:-------:|:-------:|:--------:|:---------:|
| Free | 2 | 120 | 1 | 1 |
| Pro | 10 | 600 | 3 | 5 |
| Pro+ | 30 | 1,800 | 10 | 10 |
| Enterprise | 100 | 6,000 | 50 | 25 |

Legacy cohort keys (if any remain) run at the enterprise tier until
their `beta_until` cutoff, then auto-downgrade to free + read-only. If
a static value here ever disagrees with `GET /v1/keys/tiers`, the
endpoint wins.

### Open Beta Errors

Default open beta is self-serve. Key **issuance** is not waitlist-gated:

| Error Code | HTTP | Description |
|-----------|------|-------------|
| `TIER_REQUIRES_UPGRADE` | 403 | `POST /v1/keys` / `POST /v1/keys/bootstrap` refused a free caller requesting `trade`. **Not retryable** — omit `trade` or upgrade at https://polysimulator.com/pricing. |
| `CLOSED_BETA` | 403 | Residual emergency close only. Machine code in `X-Polysim-Code`. **Not retryable**. |
| `API_PRO_COMING_SOON` | 403 | Residual cohort-rollout variant. Default open beta does not return this for paying Pro / Pro+. |
| `ACCESS_RESTRICTED` | 403 | Residual runtime gate: an already-issued key (or Bearer JWT) whose account is flagged / under review. Check `X-Polysim-Code`. **Not retryable**. |
| `COHORT_FULL` | 409 | Residual cohort-issuance capacity code. Detail: `current_active`, `cap`, `requested`. |

Legacy beta-issued keys carry an `X-API-Beta-Cutoff` response header on every request after the cutoff date — SDKs can pivot to read-only mode without an extra round-trip.

```python
# Handle key-issuance refusals. Branch on `X-Polysim-Code`;
# the body's `error` field holds the human message.
resp = requests.post(f"{BASE_URL}/v1/keys", headers={"X-API-Key": KEY}, json={"name": "bot"})
if resp.status_code == 403:
    code = resp.headers.get("X-Polysim-Code")
    if code == "TIER_REQUIRES_UPGRADE":
        print("Free keys are read-only — upgrade at https://polysimulator.com/pricing")
    elif code == "CLOSED_BETA":
        print("Issuance temporarily closed — retry later or contact support")
    elif code == "API_PRO_COMING_SOON":
        print("Residual rollout code — check /account/billing")
```

---

## Retry Strategy

```python Python — Exponential Backoff
import time
import requests

def api_call_with_retry(method, url, max_retries=3, **kwargs):
    """Make API call with exponential backoff on retryable errors."""
    for attempt in range(max_retries + 1):
        try:
            resp = requests.request(method, url, **kwargs)

            if resp.status_code == 429:
                # Rate limited — use server-provided wait time
                wait = int(resp.headers.get("Retry-After", 2 ** attempt))
                print(f"Rate limited, waiting {wait}s...")
                time.sleep(wait)
                continue

            if resp.status_code >= 500:
                # Server error — retry with backoff
                wait = 2 ** attempt
                print(f"Server error {resp.status_code}, retry in {wait}s...")
                time.sleep(wait)
                continue

            # Success or client error (no retry)
            resp.raise_for_status()
            return resp.json()

        except requests.exceptions.ConnectionError:
            wait = 2 ** attempt
            print(f"Connection error, retry in {wait}s...")
            time.sleep(wait)

    raise Exception(f"Max retries ({max_retries}) exceeded for {url}")
```

```javascript JavaScript — Exponential Backoff
async function apiCallWithRetry(method, url, options = {}, maxRetries = 3) {
  for (let attempt = 0; attempt <= maxRetries; attempt++) {
    try {
      const resp = await fetch(url, { method, ...options });

      if (resp.status === 429) {
        const wait = parseInt(resp.headers.get("Retry-After") || 2 ** attempt);
        console.log(`Rate limited, waiting ${wait}s...`);
        await new Promise(r => setTimeout(r, wait * 1000));
        continue;
      }

      if (resp.status >= 500) {
        const wait = 2 ** attempt;
        console.log(`Server error ${resp.status}, retry in ${wait}s...`);
        await new Promise(r => setTimeout(r, wait * 1000));
        continue;
      }

      if (!resp.ok) throw new Error(`HTTP ${resp.status}: ${await resp.text()}`);
      return await resp.json();

    } catch (err) {
      if (err.message.includes("fetch failed") && attempt < maxRetries) {
        const wait = 2 ** attempt;
        console.log(`Connection error, retry in ${wait}s...`);
        await new Promise(r => setTimeout(r, wait * 1000));
        continue;
      }
      throw err;
    }
  }
  throw new Error(`Max retries (${maxRetries}) exceeded for ${url}`);
}
```

---

## WebSocket Error Handling

WebSocket connections use custom close codes:

| Close Code | Meaning | Action |
|-----------|---------|--------|
| `1000` | Normal close | Reconnect if desired |
| `1001` | Server going away | Reconnect after 1s |
| `4001` | Authentication failed | Get new token, reconnect |
| `4002` | Subscription limit exceeded | Reduce subscriptions |

```python
import asyncio
import aiohttp

async def resilient_ws(url, token, market_ids):
    """WebSocket connection with automatic reconnection."""
    backoff = 1
    while True:
        try:
            async with aiohttp.ClientSession() as session:
                async with session.ws_connect(f"{url}?token={token}") as ws:
                    backoff = 1  # Reset on successful connect

                    await ws.send_json({
                        "action": "subscribe",
                        "markets": market_ids,
                    })

                    async for msg in ws:
                        if msg.type == aiohttp.WSMsgType.TEXT:
                            handle_message(msg.data)
                        elif msg.type == aiohttp.WSMsgType.CLOSED:
                            break
                        elif msg.type == aiohttp.WSMsgType.ERROR:
                            break

        except Exception as e:
            print(f"WS error: {e}")

        wait = min(backoff, 30)
        print(f"Reconnecting in {wait}s...")
        await asyncio.sleep(wait)
        backoff *= 2
```

---

## Best Practices

  
    Never assume a 2xx response. Parse the status code and handle each category appropriately.
  
  
    On 429 responses, the `Retry-After` header tells you exactly how long to wait. Don't guess.
  
  
    Client errors (400-422) indicate a problem with your request. Fix the payload instead of retrying.
  
  
    Always log the full error response body for debugging — the `details` field often contains actionable info.
