# Quick Start

Source: /quickstart

# Quick Start

  
    1. Sign up at [polysimulator.com/signin](https://polysimulator.com/signin).
    2. Open [polysimulator.com/api-keys](https://polysimulator.com/api-keys).
    3. Click **Create your first API key**, give it a name, and copy
       the `ps_live_…` value shown once.

    
      **Open beta.** Anyone can mint a key from the dashboard or
      `POST /v1/keys/bootstrap`. A **free** key is **read-only**
      (`permissions: ["read"]`) — use it to explore markets, books, and
      account reads. **Paid Pro / Pro+** keys are trade-capable
      (`["read", "trade"]`) against an isolated API wallet. See
      [API Keys](/concepts/api-keys) and
      [Authentication](/authentication#open-beta).
    

    
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
        "permissions": ["read"],
        "created_at": "2026-03-04T12:00:00Z"
      }
      ```

      A free-tier key is **read-only** (`["read"]`); trading needs a
      paid tier. See [API Keys](/concepts/api-keys#create-a-key).

      | Status | Meaning |
      |--------|--------|
      | `201` | Key created — save `raw_key` |
      | `400` | You already have key(s) — use `POST /v1/keys` with `X-API-Key` instead |
      | `401` | Invalid or expired Supabase JWT |
      | `403` | `TIER_REQUIRES_UPGRADE` if a free caller requested `trade`, or a residual issuance/runtime gate — branch on `X-Polysim-Code`. See [Authentication](/authentication#open-beta). |
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
        market = client.list_markets(limit=1)[0]   # an actively-traded market
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
      # 1. grab a live market_id (jq)
      MARKET_ID=$(curl -s -H "X-API-Key: $POLYSIM_API_KEY" \
        "$POLYSIM_BASE_URL/v1/markets?limit=1" | jq -r '.[0].condition_id')

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
      market_id = requests.get(f"{base}/v1/markets?limit=1", headers=h).json()[0]["condition_id"]
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

      const markets = await (await fetch(`${base}/v1/markets?limit=1`, { headers: h })).json();
      const resp = await fetch(`${base}/v1/orders`, {
        method: "POST", headers: h,
        body: JSON.stringify({
          market_id: markets[0].condition_id, side: "BUY", outcome: "Yes",
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
      not the dashboard MAIN wallet. API keys start at $10,000 (Pro) or
      $25,000 (Pro+); Free-tier keys are read-only with no API wallet.
      Here a $6.50 fill plus the $0.09 taker fee (PM-V2 per-category
      schedule — see [Trading Fees](/trading/fees)) against the $10,000
      Pro wallet leaves $9,993.41.
    
  

  
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
