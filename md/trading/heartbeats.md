# Heartbeats (Dead-Man's-Switch)

Source: /trading/heartbeats

# Heartbeats — Dead-Man's-Switch

If your trading bot crashes between placing orders and the next intended
cancel, those orders sit on the book exposed to adverse selection until
you notice and intervene. The **heartbeat dead-man's-switch** protects
unattended bots by auto-cancelling all of your resting orders if the
server stops receiving heartbeats for longer than a deadline you choose.

This mirrors Polymarket's `POST /heartbeats` **dead-man's-switch
behaviour** — the same path, the same `interval_ms`-style cadence, and the
same "miss a heartbeat → all your resting orders are auto-cancelled"
guarantee. A PM SDK's heartbeat loop keeps your orders protected when
pointed at PolySimulator.

  **The 200 response body differs from Polymarket** — it is **not**
  wire-identical. PolySimulator returns `{"ok": true, "expires_at_ms": }`,
  whereas Polymarket's `/heartbeats` returns a `{"status": }` body
  (and its newer SDKs thread a `heartbeat_id` through each call). If your
  bot reads PM's `status` / `heartbeat_id` field off the response, it will
  see `None` / a `KeyError` against PolySimulator — read `ok` /
  `expires_at_ms` instead. The dead-man's-switch fires on the **absence** of
  a heartbeat, so a bot that ignores the response body entirely (just keeps
  pinging on a timer) is fully protected on both platforms.

---

## Endpoints

PolySimulator exposes **two paths** that route through the same handler:

| Path | Notes |
|---|---|
| `POST /heartbeats` | Polymarket-shape root path (no `/v1/`) — for SDKs ported from PM without URL rewrite. |
| `POST /v1/heartbeats` | PolySimulator canonical `/v1/` alias — preferred for new code. |

Both accept the same body and emit the same response.

---

## Request

```json
{
  "interval_ms": 5000,
  "client_label": "alpha-bot"
}
```

| Field | Type | Required | Notes |
|---|---|---|---|
| `interval_ms` | int | Yes | Deadline between heartbeats, in milliseconds. Bounded `[1000, 60000]`. |
| `client_label` | string | No | Free-form label so a single API key can register multiple independent heartbeats from different bot processes. Defaults to `""` (single-stream). Max 64 chars. |

## Response — `200 OK`

```json
{
  "ok": true,
  "expires_at_ms": 1715518800123
}
```

| Field | Type | Notes |
|---|---|---|
| `ok` | bool | Always `true` on a successful registration / refresh. |
| `expires_at_ms` | int | Wall-clock (Unix milliseconds) at which the dead-man's-switch will fire if no further heartbeat arrives. |

The `expires_at_ms` value is `last_heartbeat_at_ms + interval_ms + grace`, where:

- `grace = max(1000ms, 0.25 × interval_ms)` — absorbs network jitter and the ~1-second evaluation cadence so bots pinging at exact intervals don't trigger spurious cancels.

---

## How to use it (the heartbeat loop)

The expected pattern is to ping at **half your `interval_ms`** so you
stay comfortably ahead of expiry even with one missed beat.

```python Python
import asyncio
import httpx

API_KEY = "ps_live_..."
INTERVAL_MS = 5000          # 5-second deadline
BEAT_EVERY = INTERVAL_MS / 2 / 1000  # ping every 2.5s

async def heartbeat_loop():
    async with httpx.AsyncClient(
        base_url="https://api.polysimulator.com",
        headers={"X-API-Key": API_KEY},
    ) as client:
        while True:
            try:
                resp = await client.post(
                    "/v1/heartbeats",
                    json={"interval_ms": INTERVAL_MS},
                )
                resp.raise_for_status()
            except Exception as e:
                # NOTE: if the heartbeat call itself fails (network blip
                # or a 5xx), DON'T treat that as "bot is dead" — the
                # server-side switch fires only on absence, not error.
                # Just log and try again on the next tick.
                print(f"heartbeat failed: {e!r}")
            await asyncio.sleep(BEAT_EVERY)

asyncio.run(heartbeat_loop())
```

```bash curl
# One-shot — refreshes the heartbeat. Call this on a 2.5-second timer
# (cron / systemd / your own scheduler) for a 5-second deadline.
curl -X POST https://api.polysimulator.com/v1/heartbeats \
  -H "X-API-Key: ${API_KEY}" \
  -H "Content-Type: application/json" \
  -d '{"interval_ms": 5000}'
```

```javascript JavaScript
const API_KEY = "ps_live_...";
const INTERVAL_MS = 5000;

setInterval(async () => {
  try {
    const r = await fetch("https://api.polysimulator.com/v1/heartbeats", {
      method: "POST",
      headers: {
        "X-API-Key": API_KEY,
        "Content-Type": "application/json",
      },
      body: JSON.stringify({ interval_ms: INTERVAL_MS }),
    });
    if (!r.ok) console.warn("heartbeat failed:", r.status);
  } catch (e) {
    console.warn("heartbeat error:", e);
  }
}, INTERVAL_MS / 2);
```

---

## What happens when a heartbeat is missed?

Once your deadline lapses without a fresh heartbeat — typically within
about a second of the `expires_at_ms` you were given — the server:

1. Retires the lapsed registration.
2. Cancels **all** pending limit orders for the API key's account (same
   logic as `POST /v1/cancel-all` — refunds BUY notional, returns SELL
   shares to position).

To resume, the bot just calls `POST /v1/heartbeats` again — a new
registration is created from scratch.

  The dead-man's-switch cancels every resting order for the API key's
  account, including orders placed by other processes sharing the
  same key. If you run multiple bot strategies on one key, use
  `client_label` to register independent heartbeats — but be aware
  that the cancel-all still cancels EVERY pending order, not just the
  labelled subset. Use distinct API keys per strategy if you need
  strategy-level isolation.

---

## Bounds and error responses

| Field | Bound | Out-of-range response |
|---|---|---|
| `interval_ms` | `[1000, 60000]` | `422 Unprocessable Entity` (Pydantic validation envelope) |
| `client_label` | `≤ 64` chars | `422 Unprocessable Entity` |

The `[1000, 60000]` bound is deliberate:

- **Below 1000ms** would burn rate-limit quota (2 RPS per registration just for heartbeats) with no safety benefit beyond what 1-second sweeps already provide.
- **Above 60000ms** defeats the point — a crashed bot would stay exposed for a full minute before its orders cancel.

---

## Durability guarantee

Heartbeat registrations are persisted, not held only in process memory:

- **Server restarts don't drop your protection.** A deploy or graceful
  reload preserves every active heartbeat. Your bot's next refresh
  after the restart simply bumps the expiry forward — no need to
  re-register from scratch.
- **Refreshes and the expiry sweep never race.** A heartbeat that
  lands at the same instant the deadline is being evaluated keeps your
  orders alive — the switch never fires spuriously from a timing race.
  It fires only on a genuine, sustained absence of heartbeats past your
  deadline.

---

## Related

- [Cancel All Orders (`POST /v1/cancel-all`)](/trading/order-management#cancel-all-orders) — Manual cancel-all that the dead-man's-switch delegates to internally.
- [Polymarket Raw HTTP](/concepts/pm-raw-http) — Other PM-shape compat endpoints.
- [Error Handling](/bots/error-handling) — Bot resilience patterns including heartbeat-failure recovery.
