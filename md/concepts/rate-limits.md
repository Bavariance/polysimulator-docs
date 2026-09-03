# Rate Limits

Source: /concepts/rate-limits

# Rate Limits

Rate limits are enforced **per API key** using Redis **fixed-window** counters with both
per-second (RPS) and per-minute (RPM) buckets.

  **Fixed window, not sliding — and what that means for you.** Each bucket is keyed
  to a wall-clock second (and a wall-clock minute) and resets at the boundary. A
  client that happens to align with the boundary can therefore send up to **twice**
  its per-second allowance in a very short interval: at a 2 RPS tier, two requests
  at `t = 0.999s` and two at `t = 1.001s` are four requests in two milliseconds and
  all four are admitted. Sustained throughput is still capped by the per-minute
  bucket. Do not build a strategy on that burst — it is an artefact of the window
  shape, it is not part of the contract, and it may be removed without a breaking
  change. `X-RateLimit-Reset` tells you when the current minute bucket resets.

  An environment may instead run a **weighted sliding approximation** over the
  same buckets, which is strictly *stricter*: the boundary burst above is
  rejected, `Retry-After` is solved from how much of the previous bucket is
  still counted rather than being a flat 1 s, and `X-RateLimit-Reset` names the
  instant your next request would actually be served rather than the wall-clock
  boundary. A client that honours `Retry-After` and `X-RateLimit-Reset` is
  correct under both; a client that assumes the boundary burst is not.

  **If Redis is unavailable, the limiter fails open.** Counters live in Redis; when
  Redis errors, the API admits the request rather than rejecting it, because an
  unavailable cache should not take the whole API down. Enforcement is therefore
  best-effort during a Redis incident — you may briefly see no limits at all, or a
  reduced fallback budget where that has been enabled. Your client should still
  honour 429 and `Retry-After` at all times; do not treat a period without limits as
  a licence to raise your send rate.

---

## Tiers

| Tier | Requests/sec | Requests/min | Max WS Connections | Max Batch Size |
|------|:-----------:|:------------:|:------------------:|:--------------:|
| `free` | 2 | 120 | 1 | 1 |
| `pro` | 10 | 600 | 3 | 5 |
| `pro_plus` | 30 | 1,800 | 10 | 10 |
| `enterprise` | 100 | 6,000 | 50 | 25 |

  The `free` tier allows short bursts (up to 2 requests inside one wall-clock
  second) and is capped at 120 requests inside one wall-clock minute, so the
  per-minute bucket is the one you'll hit first under sustained load. Both
  buckets are fixed windows keyed to the clock rather than rolling ones: the
  minute bucket resets at the top of each minute, and `x-ratelimit-reset` tells
  you when. Use `POST /v1/prices/batch` and the WebSocket feeds (which don't
  count against the REST limit) to stay well inside it.

  The authoritative source for tier limits is `GET /v1/keys/tiers`. If a doc
  page ever disagrees with that endpoint, the endpoint wins.

---

## Rate Limit Response

When you exceed your limit, the API returns **HTTP 429** with the
Polymarket-shape single-field `error` envelope and a stable
`X-Polysim-Code` response header:

```http
HTTP/1.1 429 Too Many Requests
Retry-After: 1
X-Polysim-Code: RATE_LIMIT_EXCEEDED
X-Request-Id: a1b2c3d4-...
Content-Type: application/json
```

```json
{"error": "Rate limit exceeded. Retry after 1s."}
```

Branch on `X-Polysim-Code === "RATE_LIMIT_EXCEEDED"` rather than the
body prose, and read the `Retry-After` header for the exact wait time
in seconds.

---

## Rate Limit Headers (on authenticated responses)

Every authenticated response carries `x-ratelimit-*` headers so bots
can **pre-throttle** instead of waiting for an actual 429 (unauthenticated
public routes and legacy keyed public 429s return `Retry-After`, `X-Polysim-Code`,
and `X-Request-Id`; full quota metadata headers are returned on authenticated
endpoints and flagged token resolver routes):
| Header | Type | Description |
|--------|------|-------------|
| `x-ratelimit-tier` | string | Tier of the key making the call — `free` / `pro` / `pro_plus` / `enterprise` |
| `x-ratelimit-limit` | int | Per-minute cap for the tier |
| `x-ratelimit-limit-per-second` | int | Per-second cap for the tier |
| `x-ratelimit-remaining` | int | Requests remaining in the current fixed one-minute window |
| `x-ratelimit-remaining-per-second` | int | Requests remaining in the current fixed one-second window |
| `x-ratelimit-reset` | int | Unix epoch seconds when the per-minute window rolls over |

  Every header above is **also** emitted under the PolySim-namespaced
  `x-polysim-ratelimit-*` prefix with identical values (e.g.
  `x-polysim-ratelimit-remaining`). Read whichever your SDK or proxy
  keys off — the unprefixed `x-ratelimit-*` form is canonical.

```python
import time
import requests

# Pre-throttle off the response headers — don't wait for 429.
resp = requests.get(url, headers={"X-API-Key": KEY})
remaining = int(resp.headers.get("x-ratelimit-remaining", "1"))
remaining_sec = int(resp.headers.get("x-ratelimit-remaining-per-second", "1"))
reset_at = int(resp.headers.get("x-ratelimit-reset", "0"))

# If the per-second bucket is almost drained, sleep a beat
if remaining_sec <= 1:
    time.sleep(1.0)
# If the current minute bucket is almost drained, sleep until reset
elif remaining <= 5:
    sleep_for = max(0.0, reset_at - time.time())
    time.sleep(min(sleep_for, 60.0))
```

---

## Handling Rate Limits

When the limiter does fire, exponential backoff keyed off `Retry-After`:

```python
import time
import requests

def api_request(url, headers, json_data=None, max_retries=3):
    for attempt in range(max_retries):
        resp = requests.post(url, headers=headers, json=json_data)

        # 429 → respect Retry-After (always populated in seconds)
        if resp.status_code == 429:
            retry_after = int(resp.headers.get("Retry-After", 1))
            time.sleep(retry_after)
            continue

        if resp.status_code >= 500:
            time.sleep(2 ** attempt)  # Exponential backoff
            continue

        return resp

    raise Exception(f"Failed after {max_retries} retries")
```

---

## Safe Order Polling Contract

A common pattern among trading bots is checking whether a resting limit order has been filled or canceled. **Do not poll `GET /v1/orders/{id}` or `GET /v1/orders` in an unthrottled loop.** Rapid polling exhausts per-second (RPS) and per-minute (RPM) buckets, generating `429 RATE_LIMIT_EXCEEDED` responses.

### Authoritative REST Polling Rules

REST endpoints are the authoritative source for order state, fills, and cancellations. When implementing order tracking:

1. **Minimum 1–2 Second Interval:** Never poll an individual resting order more frequently than once every 1–2 seconds.
2. **Add Jitter:** Apply randomized delays (e.g. `1.0s + random(0.1s, 0.5s)`) to prevent synchronized polling bursts across multiple orders.
3. **Honor `Retry-After`:** If a `429` occurs, parse the `Retry-After` header and back off for the specified duration before retrying.
4. **Batch Fetching:** Use `GET /v1/data/orders` or `GET /v1/orders?status=PENDING` to inspect all open orders in a single request rather than executing individual `GET /v1/orders/{id}` calls per order.

### WebSocket Fill Streams (Opportunistic & Fill-Only)

For lower fill latency, clients may listen to WebSocket execution channels alongside authoritative REST reconciliation:
* **PolySim Native:** `WS /v1/ws/executions?token=` receives push notifications when limit orders fill in-process.
* **Polymarket Parity:** `WS /v1/ws/user` (with `auth.apiKey`) streams live `trade` frames with `status: "MATCHED"`.

  **WebSocket Limitations (Fill-Only & Process-Local):**
  * **No Cancellation Events:** Execution WebSockets emit order fill events only; they do **not** emit order cancellation events. Cancellations must be monitored or confirmed via REST endpoints.
  * **Process-Local Scope:** In multi-process and daemon deployments, matching loops running in background daemon processes do not cross process boundaries to API-worker WebSocket registries. Bots **must retain periodic REST reconciliation** as the authoritative source of truth.

---

## Best Practices

  
    `POST /v1/orders/batch` and `POST /v1/prices/batch` combine multiple
    operations into one request — and a batch call counts as **one** tick
    against your RPS/RPM. It's bounded by your tier's **Max Batch Size**
    (see the Tiers table above): `free=1` means no batching benefit on
    free, so this pays off most on `pro` (5) / `pro_plus` (10) /
    `enterprise` (25).
  

  
    Subscribe to `WS /v1/ws/prices` instead of polling `GET /v1/markets`.
    WebSocket connections don't count against your REST rate limit.
  

  
    Market metadata (slug, question, outcomes) changes infrequently.
    Cache it locally and only refresh periodically.
  

  
    On order placement (`POST /v1/orders`, `POST /v1/order`,
    `POST /v1/clob/order`), send an `Idempotency-Key` header — a
    Stripe-style alias for the body's `client_order_id` — so a retried
    request can't double-fill. Reusing a key with a **different** payload
    returns `409 IDEMPOTENCY_KEY_REUSE`; reuse it only for the exact same
    order you're retrying.
  

---

## Next Steps

- [String Numerics](/concepts/string-numerics) — Why all numbers are strings
- [Batch Orders](/trading/batch-orders) — Reduce request count with batching
