# Quick Start

Source: /quickstart

# Quick Start

  
    1. Sign up at [polysimulator.com/signin](https://polysimulator.com/signin).
    2. Open the developer console at [polysimulator.com/console](https://polysimulator.com/console).
    3. Click **Create a free API key** (on a paid plan the button names your
       plan) and copy the `ps_live_…` value shown once.

    
      **Open beta.** Anyone can mint a key from the dashboard or
      `POST /v1/keys/bootstrap`. **A free key issued on or after
      2026-09-05 can trade**: it is minted with `["read", "trade"]` and
      places orders against a **\$100 API wallet** that cannot be reset
      or topped up, capped at **100 order placements per UTC day** per
      account. **A free key issued before 2026-09-05 is read-only**
      (`["read"]`) — `trade` is stripped on every request even if the
      key was stored with it; create a new key to get a trade-capable
      one. **Paid Pro / Pro+** keys trade against a \$10,000 / \$25,000
      API wallet with no daily cap. See [API Keys](/concepts/api-keys)
      and [Authentication](/authentication#open-beta).
    

    
      The full key is shown **only once**. Save it to your password
      manager or a secret store immediately — only the SHA-256 hash is
      retained server-side, so we can't show it again later.
    

    
      The dashboard handles the one-time bootstrap with your signed-in
      Supabase session — you never see or paste a JWT. From here on
      every API call uses `X-API-Key: ps_live_...`.
    

    
      If you can't open a browser (CI runner, containerised dev env)
      and you have a Supabase access token in hand, the
      `POST /v1/keys/bootstrap` endpoint creates your first key directly:

      ```bash
      # SUPABASE_JWT comes from a programmatic Supabase sign-in.
      # Most users skip this step entirely — the /api-keys dashboard
      # is the recommended path.
      curl -X POST https://api.polysimulator.com/v1/keys/bootstrap \
        -H "Authorization: Bearer $SUPABASE_JWT" \
        -H "Content-Type: application/json" \
        -d '{"name": "my-first-bot"}'
      ```

      **Response (201 Created):**

      ```json
      {
        "id": 1,
        "raw_key": "ps_live_kJ9mNx2pQrStUvWxYz01Ab3CdEfGhI4j...",
        "key_prefix": "ps_live_kJ9mNx2p",
        "name": "my-first-bot",
        "rate_limit_tier": "free",
        "permissions": ["read", "trade"],
        "created_at": "2026-09-12T12:00:00Z"
      }
      ```

      A free-tier key issued on or after **2026-09-05** carries
      `["read", "trade"]` and trades against the \$100 free API wallet;
      one issued before 2026-09-05 is read-only (`["read"]`). A
      paid tier lifts the \$100 budget and the daily order cap. See
      [API Keys](/concepts/api-keys#create-a-key).

      | Status | Meaning |
      |--------|--------|
      | `201` | Key created — save `raw_key` |
      | `400` | You already have key(s) — use `POST /v1/keys` with `X-API-Key` instead |
      | `401` | Invalid or expired Supabase JWT |
      | `403` | `TIER_REQUIRES_UPGRADE` if a free caller asked for a paid `tier` it is not subscribed to, for `admin`, or for `trade` while holding an expired paid grant; or a residual issuance/runtime gate — branch on `X-Polysim-Code`. See [Authentication](/authentication#open-beta). |
      | `429` | Bootstrap rate limit hit — wait and retry |

      `Authorization: Bearer` is accepted on the dashboard surface
      (`POST /v1/keys/bootstrap`, `GET/POST/DELETE /v1/keys`,
      `/v1/keys/tiers`, `/v1/keys/ws-token`, `GET /v1/me`,
      `/v1/account/me/entitlements`, `/v1/me/wallets/*`).
      **All trading, market-data, websocket, and account-trading
      reads (`/v1/account/{balance,positions,portfolio,history,equity}`)
      require `X-API-Key`** — Bearer is rejected on the trade surface.
      See the [Authentication](/authentication) page for the full
      scope table.
    
  

  
    ```bash
    export POLYSIM_API_KEY="ps_live_abc123..."
    export POLYSIM_BASE_URL="https://api.polysimulator.com"
    ```
  

  
    ```bash
    curl -H "X-API-Key: $POLYSIM_API_KEY" \
         $POLYSIM_BASE_URL/v1/health
    ```

    Expected response:
    ```json
    {"status": "ok", "timestamp": "2026-03-02T12:00:00Z", "version": "1.0.0"}
    ```
  

  
    ```bash
    curl -H "X-API-Key: $POLYSIM_API_KEY" \
         "$POLYSIM_BASE_URL/v1/markets?limit=5"
    ```

    This returns actively traded markets with live prices from Polymarket.
  

  
    The fastest path is the **Python SDK** — it picks a live market and places
    a trade in a few lines. This snippet is **complete and runnable as-is**
    (it resolves a real market for you — no IDs to fill in):

    ```python Python SDK
    # pip install polysimulator
    from polysim_sdk import PolySimClient

    with PolySimClient() as client:                 # reads POLYSIM_API_KEY from env
        # active_only=True skips closed markets: an order on one is cancelled, not filled
        market = client.list_markets(limit=20, active_only=True)[0]
        fill = client.place_order(
            market_id=market["condition_id"],
            side="BUY",
            outcome="Yes",
            quantity=10,
            order_type="market",
            price="0.99",            # worst-price cap; "0.99" on a YES = accept any fill
        )
        print(f"filled {fill['status']} @ {fill.get('price')} — order {fill['order_id']}")
    ```

    
      
    

    
      **`outcome` takes the human-readable label** (`"Yes"`, `"No"`, or custom labels like `"Trump"`), not Polymarket's 77-digit token ID. To map a token ID to its outcome label, use `GET /v1/markets-by-token/{token_id}`.

      **Market orders require `price` as a worst-price limit** — Polymarket-faithful
      slippage protection. A BUY won't fill above it; a SELL won't fill below it.
      `"0.99"` on a YES means "accept any fill" (great for your first trade); for
      tighter control use the current best ask × 1.05.
    

    Prefer raw HTTP? Resolve a live `market_id` from the markets endpoint first,
    then POST — these are runnable too:

    
      ```bash cURL
      # 1. grab a TRADEABLE market_id (jq)
      #    Without active_only=true, /v1/markets also returns closed markets,
      #    and an order against one is cancelled rather than filled. The jq
      #    select is a second guard.
      MARKET_ID=$(curl -s -H "X-API-Key: $POLYSIM_API_KEY" \
        "$POLYSIM_BASE_URL/v1/markets?limit=20&active_only=true" \
        | jq -r '[.[] | select(.closed == false)][0].condition_id')

      # 2. place the trade
      curl -X POST $POLYSIM_BASE_URL/v1/orders \
        -H "X-API-Key: $POLYSIM_API_KEY" -H "Content-Type: application/json" \
        -d "{\"market_id\":\"$MARKET_ID\",\"side\":\"BUY\",\"outcome\":\"Yes\",\"quantity\":\"10\",\"order_type\":\"market\",\"price\":\"0.99\"}"
      ```

      ```python Python (requests)
      import requests, os

      base, key = os.environ["POLYSIM_BASE_URL"], os.environ["POLYSIM_API_KEY"]
      h = {"X-API-Key": key, "Content-Type": "application/json"}

      # resolve a live market, then trade
      # active_only=true skips closed markets (an order on one is cancelled);
      # the client-side check below is a second guard.
      markets = requests.get(f"{base}/v1/markets?limit=20&active_only=true", headers=h).json()
      market_id = next(m for m in markets if not m["closed"])["condition_id"]
      resp = requests.post(f"{base}/v1/orders", headers=h, json={
          "market_id": market_id, "side": "BUY", "outcome": "Yes",
          "quantity": "10", "order_type": "market",
          "price": "0.99",  # worst-price limit (slippage cap)
      })
      print(resp.json())
      ```

      ```javascript JavaScript
      const base = process.env.POLYSIM_BASE_URL, key = process.env.POLYSIM_API_KEY;
      const h = { "X-API-Key": key, "Content-Type": "application/json" };

      // active_only=true skips closed markets (an order on one is cancelled);
      // the client-side check below is a second guard.
      const markets = await (await fetch(`${base}/v1/markets?limit=20&active_only=true`, { headers: h })).json();
      const market = markets.find((m) => !m.closed);
      const resp = await fetch(`${base}/v1/orders`, {
        method: "POST", headers: h,
        body: JSON.stringify({
          market_id: market.condition_id, side: "BUY", outcome: "Yes",
          quantity: "10", order_type: "market",
          price: "0.99",  // worst-price limit (slippage cap)
        }),
      });
      console.log(await resp.json());
      ```
    

    Response:
    ```json
    {
      "order_id": 42,
      "status": "FILLED",
      "order_type": "market",
      "side": "BUY",
      "outcome": "Yes",
      "price": "0.65",
      "quantity": "10",
      "notional": "6.50",
      "fee": "0.09",
      "slippage_bps": 15,
      "account_balance": "9993.41"
    }
    ```

    
      `account_balance` is your **API wallet** balance after the fill,
      not the dashboard MAIN wallet. API wallets start at \$100 (Free,
      keys issued on or after 2026-09-05 — non-renewable), \$10,000 (Pro)
      or \$25,000 (Pro+); a free key issued before 2026-09-05 is read-only
      and cannot place this order. Here a \$6.50 fill plus the \$0.09 taker
      fee (PM-V2 per-category schedule — see [Trading Fees](/trading/fees))
      against the \$10,000 Pro wallet leaves \$9,993.41.
    
  

  
    ```bash
    curl -H "X-API-Key: $POLYSIM_API_KEY" \
         $POLYSIM_BASE_URL/v1/account/portfolio
    ```
  

  **All numeric values are strings** (`"10"`, not `10`). This prevents
  floating-point precision loss — critical for financial applications.
  See [String Numerics](/concepts/string-numerics) for details.

---

## Run your first backtest

Backtesting is the reason most people come to the API, and it was reachable from
here only as a link. This is the whole path, in four calls.

Needs your plan's `analytics.backtesting` entitlement. Every call takes
`X-API-Key`.

### 1. Check what you have left

```bash
curl -H "X-API-Key: $POLYSIM_API_KEY" \
  $POLYSIM_BASE_URL/v1/simulation/quota
```

Backtests are metered in **market-hours** — window length x number of markets. A
10-market, 3-day run costs `10 x 72 = 720` market-hours, not "one backtest".
Check here before a large run rather than discovering the cap mid-request.

### 2. Check the market has the data

```bash
curl -H "X-API-Key: $POLYSIM_API_KEY" \
  "$POLYSIM_BASE_URL/v1/simulation/coverage?condition_id=$MARKET_ID&start=2026-09-07T12:00:00Z&end=2026-09-07T18:00:00Z"
```

**Do this first.** A backtest is refused outright when book coverage for the
window is below the minimum (80% by default), and the refusal names the
percentage. Checking coverage turns a confusing rejection into a window you can
adjust. `confidence` and `missing_periods` in the response tell you which hours
are thin.

### 3. Submit the run

Save the request body first — the next step reuses it:

```bash
cat > backtest.json <<JSON
{
    "condition_ids": ["$MARKET_ID"],
    "start": "2026-09-07T12:00:00Z",
    "end":   "2026-09-07T18:00:00Z",
    "initial_bankroll": "1000",
    "strategy": {
      "spec_version": "1",
      "side": "BUY",
      "outcome": "Yes",
      "size": "10",
      "fill_model": "depth_walk",
      "entry": [{"field": "price", "op": "lte", "value": "0.40"}],
      "exit":  [{"field": "price", "op": "gte", "value": "0.60"}]
    }
}
JSON

JOB_ID=$(curl -s -X POST $POLYSIM_BASE_URL/v1/simulation/backtests \
  -H "X-API-Key: $POLYSIM_API_KEY" -H "Content-Type: application/json" \
  -d @backtest.json | python3 -c 'import sys,json;print(json.load(sys.stdin)["job_id"])')
echo "$JOB_ID"
```

**Submit once.** Each successful POST charges the window's market-hours and one
job against your monthly count. Re-running this block starts a *second* backtest;
it does not re-read the first.

Worth knowing before you size a run: **both endpoints count**. The 12:00→18:00
window above is **7 market-hours, not 6** — 12:00, 13:00 … 18:00 is seven hour
marks. Multiply by the number of markets.

Buy 10 shares of Yes whenever it trades at or below 40c, sell at or above 60c.
`depth_walk` walks the real order book rather than assuming you fill at the top —
see [Fill models](/simulation) for what each one assumes.

You get back a `job_id` and `status: "pending"`. The run is **charged at
submission**; a run that fails is refunded.

### 4. Poll for the result

```bash
curl -H "X-API-Key: $POLYSIM_API_KEY" \
  $POLYSIM_BASE_URL/v1/simulation/backtests/$JOB_ID
```

When `status` is `completed`, the summary carries `total_pnl`,
`total_return_pct`, and — read these — `tick_mode`, `ticks_evaluated`,
`tick_degraded` and `tick_notice`. **`tick_degraded: true` means the run
evaluated at a coarser resolution than requested**, and `tick_notice` says why.

They do **not**, on their own, tell you the data was complete. Those two describe
the evaluation GRID; coverage describes the DATA. A window can clear the 80%
coverage gate with an hour missing, and if that hour held the only entry your
rules would have matched, the run completes with zero trades, `tick_degraded:
false`, and nothing obviously wrong. **Zero trades is a result to investigate,
not a finding** — check `missing_periods` from step 2 before concluding a
strategy does not trade.

Then:

```bash
curl -H "X-API-Key: $POLYSIM_API_KEY" \
  $POLYSIM_BASE_URL/v1/simulation/backtests/$JOB_ID/trades
curl -H "X-API-Key: $POLYSIM_API_KEY" \
  $POLYSIM_BASE_URL/v1/simulation/backtests/$JOB_ID/equity
```

`settlement_scored: false` in the summary means the window ended before the
market resolved: any position still open was marked to its last observed price
rather than to a payout. That is the normal case for a sub-interval backtest and
not an error.

## What's Next?

- [Authentication deep dive](/authentication) — Key management, security, permissions
- [Rate Limits](/concepts/rate-limits) — Understand your tier's request budget
- [Build a trading bot](/bots/example-trading-bot) — Complete Python example
- [Simulation API](/simulation) — Historical fill / coverage. **Live and key-gated**: 401 without a key, not 404. Also needs your plan's `analytics.backtesting` entitlement.
