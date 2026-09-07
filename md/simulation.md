# Simulation API

Source: /simulation

# Simulation API

The `/v1/simulation` surface is a **live**, key-gated historical fill and
backtest contract. Without a key it returns **401**, not 404 — the endpoints
exist and are serving. `FEATURE_SIMULATION_API_ENABLED` defaults to `true`; it
is a kill switch an operator can throw, not an activation step you must wait for.

  Live does not mean unlimited. Access is gated on your plan's
  `analytics.backtesting` entitlement and metered in simulated market-hours, so
  a call can still be refused on entitlement or quota rather than on auth. The
  volume caps below are plan-matrix numbers. This page does not claim a Telonex
  or vendor-validated SKU.

## First successful call

`simulateFill` is the hook: one condition, one timestamp, one memorable
VWAP. Do not start with a backtest.

```python
from polysim_sdk import PolySimClient

with PolySimClient() as client:
    fill = client.simulation.fill(
        condition_id="0xabc123",
        shares="10",
        at="2026-08-04T18:00:00Z",
        fill_model="depth_walk",
        outcome="Yes",
    )
    print(fill["vwap_fill_price"], fill["token_id"], fill["comp_token_id"])
```

All money and share fields are **decimal strings**. Parse with
`Decimal`, never `float`.

## Nine paths

| Method | Path | Purpose |
| --- | --- | --- |
| `POST` | `/v1/simulation/fill` | Point-in-time counterfactual fill |
| `GET` | `/v1/simulation/coverage` | Per-hour archive verdicts + settlement overlay |
| `GET` | `/v1/simulation/fill-models` | Named fill-model ladder |
| `GET` | `/v1/simulation/book` | Reconstructed book at one timestamp |
| `POST` | `/v1/simulation/backtests` | Enqueue a declarative job (`202`) |
| `GET` | `/v1/simulation/backtests` | Keyset list |
| `GET` | `/v1/simulation/backtests/{id}` | Job summary, including resolved `token_ids` |
| `GET` | `/v1/simulation/backtests/{id}/trades` | Keyset trades |
| `GET` | `/v1/simulation/backtests/{id}/equity` | Keyset equity (not a full dump) |
| `DELETE` | `/v1/simulation/backtests/{id}` | Cancel |
| `POST` | `/v1/simulation/exports` | Signed Parquet URL (`503` until R2 is configured) |

`GET /v1/simulation/book` resolves a bounded hour record from Redis or
an adjacent v4 sidecar. It never walks an unbound `hours` list.

## Token identity

Fill and book responses echo the **resolved** `token_id` and
`comp_token_id`. Backtest create accepts an optional `token_ids` map
(`condition_id → token_id`). The job stores the resolved map so clients
can verify the exact books that were walked. Exact `asset_id` matching
is preserved: `primary` / `yes` / `no` never match a real token.

## Errors

Every simulation error is a structured `/v1` envelope. Branch on
`X-Polysim-Code`, not the body text. Declared codes include:

- `INVALID_REQUEST`, `INVALID_WINDOW`, `INVALID_CURSOR`
- `UPGRADE_REQUIRED`, `INSUFFICIENT_PERMISSION`, `NOT_FOUND`
- `IDEMPOTENCY_KEY_REUSE`, `IDEMPOTENCY_CONFLICT_PENDING`
- `UNIVERSE_EXCLUDED`
- `BACKTEST_MONTHLY_CAP`, `BACKTEST_MARKET_CAP`, `BACKTEST_WINDOW_CAP`
- `SIMULATION_HOURS_EXCEEDED`
- `METERING_UNAVAILABLE`, `JOB_STORE_UNAVAILABLE` (`503`)
- `CONDITION_HOUR_NOT_COVERED`, `ARCHIVE_UNAVAILABLE`, `ARCHIVE_NOT_FOUND`

OpenAPI advertises `503` plus those codes. A missing sidecar next to a
v4 capture object is fail-closed (`quarantined`), not a fabricated
valid hour.

## Volume caps (matrix, not activation)

`FEATURE_MATRIX` gates **volume**, not the fill/coverage features:

| Key | free | pro | pro_plus |
| --- | --- | --- | --- |
| `simulation.backtests_per_month` | 10 | 100 | 9999 |
| `simulation.backtest_max_days` | 7 | 30 | 90 |
| `simulation.backtest_max_markets` | 5 | 25 | 100 |
| `simulation.market_hours_monthly` | 24 | 720 | 9999 |

`analytics.backtesting` remains `false` for free/pro. `depth_walk_depletion`
and signed exports still require `dataset.historical_full`. Do not treat
these numbers as a live commercial SKU.

## SDK

```python
client.simulation.fill(...)
client.simulation.coverage(condition_id=...)
client.simulation.book(condition_id=..., at=...)
job = client.simulation.create_backtest(...)
job = client.simulation.wait(job["job_id"])
client.simulation.list_backtests()
client.simulation.get_backtest(job_id)
client.simulation.cancel_backtest(job_id)
```

`wait()` polls `get_backtest` until `completed` / `failed` / `cancelled`.
It does not enable the flag and does not talk to Telonex.
