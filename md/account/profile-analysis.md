# Profile Analysis

Source: /account/profile-analysis

## Overview

The Profile Analysis endpoint aggregates user profile + portfolio data into a single payload for bots, dashboards, and LLM workflows.

This page documents the **exact calculation logic** used by the API so users can verify every metric.

This endpoint is designed for MCP (Model Context Protocol) tools and AI agents.
Values are rounded for display in the API response, but formulas below describe the source calculations.

## Endpoint

  Full profile analysis with all metrics

### Query Parameters

  Number of recent trades to include (1-100)

  Days of equity history to analyze (1-365)

  Wallet scope for the whole analysis — positions, trading stats, risk
  metrics, snapshots, recent trades, cash basis and PnL baseline alike. An
  integer scopes to a single wallet you own (404 `WALLET_NOT_FOUND`
  otherwise); `api` scopes to your API wallet (including legacy rows recorded
  before per-wallet attribution); `all` restores the cross-wallet blended
  view. Keywords are case-insensitive; any other value returns 422
  `VALIDATION_FAILED`. **Omitted = `api`.**

  **Default changed on 2026-06-10.** Before 2026-06-10 this endpoint always
  blended API-wallet cash with **all** wallets' position values, so `api_pnl`
  could report large gains on a flat API wallet. The analysis is now scoped
  to the **API wallet** by default and `api_pnl` reconciles with
  [Balance](/account/balance). Pass `wallet_id=all` if you depended on the
  old blended view.

### Authentication

API key only — `X-API-Key: ` (or the PM-compat `POLY_API_KEY` /
`Authorization: Bearer ps_live_...` aliases). A Supabase Bearer JWT is **not**
accepted here.

```bash
curl -H "X-API-Key: $POLYSIM_API_KEY" \
  https://api.polysimulator.com/v1/account/profile-analysis
```

### Errors

All errors return `{"error": "<CODE>", "message": ""}`.

| Status | `error` code | When |
|--------|--------------|------|
| 401 | `MISSING_API_KEY` | No API key header supplied |
| 401 | `INVALID_KEY` | API key is unknown, deactivated, or expired |
| 401 | `USER_NOT_FOUND` | Authenticated user no longer exists in the database — re-authenticate |
| 404 | `ACCOUNT_NOT_FOUND` | Authenticated user has no account record |
| 404 | `WALLET_NOT_FOUND` | Integer `wallet_id` does not exist or is not owned by the caller |
| 422 | `VALIDATION_FAILED` | `wallet_id` is neither an integer, `all`, nor `api` |
| 500 | `ANALYSIS_ERROR` | Unexpected server error building the analysis |

## Response

The top level also carries `analysis_generated_at` (ISO-8601 UTC string),
`analysis_version` (string, currently `"1.0.0"`), and a `natural_language_summary`
string, alongside these sections:

### `profile` — User Metadata

| Field | Type | Description |
|-------|------|-------------|
| `user_id` | integer | Internal user ID |
| `username` | string \| null | Public username |
| `display_name` | string \| null | Display name |
| `bio` | string \| null | User bio |
| `avatar_url` | string \| null | Avatar image URL |
| `auth_provider` | string \| null | Auth provider used to sign up (e.g. `email`, `google`) |
| `account_created_at` | string \| null | ISO-8601 UTC registration timestamp |
| `account_age_days` | integer \| null | Days since registration |
| `api_key_tier` | string | API key tier ceiling: `free`, `pro`, or `enterprise` (a Pro+ plan maps to the `enterprise` key tier). Defaults to `free`. |
| `profile_visibility` | string | Profile visibility (e.g. `public`, `private`). Defaults to `public`. |

### `balance` — Balance Summary

| Field | Type | Description |
|-------|------|-------------|
| `currency` | string | Always `USD` |
| `ui_balance` | string | UI (MAIN) trading balance — always the MAIN wallet, regardless of `wallet_id` |
| `api_balance` | string | Cash of the **scoped wallet** (the API wallet by default; the field name is historical) |
| `starting_ui_balance` | string | UI baseline = seed + top-ups + grants |
| `starting_api_balance` | string | PnL baseline of the **scoped wallet** — its actual `starting_balance` (tier-aware: Pro $10,000 / Pro+ $25,000) |
| `ui_pnl` | string | UI profit/loss (`ui_balance − starting_ui_balance`) |
| `api_pnl` | string | Scoped-wallet profit/loss (`total_portfolio_value − starting_api_balance`) |
| `ui_pnl_percentage` | string | UI P&L as percentage |
| `api_pnl_percentage` | string | Scoped-wallet P&L as percentage |
| `total_portfolio_value` | string | Scoped-wallet cash + scoped-wallet positions |
| `unrealized_pnl` | string | Open-position mark-to-market P&L (scoped) |
| `realized_pnl` | string | Realized P&L from closed positions (scoped) |

### `trading_stats` — Trading Statistics

| Field | Type | Description |
|-------|------|-------------|
| `total_trades` | integer | Total filled trades |
| `win_rate` | string | Win rate percentage |
| `profit_factor` | string | Gross profit / gross loss |
| `total_volume` | string | Total traded volume |
| `avg_trade_size` | string | Average trade notional |
| `best_trade_pnl` | string | Best single trade P&L |
| `worst_trade_pnl` | string | Worst single trade P&L |
| `unique_markets_traded` | integer | Distinct markets |
| `active_trading_days` | integer | Days with at least one trade |

### `risk_metrics` — Risk Analysis

| Field | Type | Description |
|-------|------|-------------|
| `portfolio_diversity_score` | string | 1 - HHI (0=concentrated, 1=diverse) |
| `largest_position_weight` | string | Largest position as % of portfolio |
| `top_3_concentration` | string | Top 3 positions as % of portfolio |
| `max_drawdown_7d` | string | Max running-peak drawdown over 7 days |
| `max_drawdown_30d` | string | Max running-peak drawdown over 30 days |
| `cash_percentage` | string | Cash as % of total |

## How Metrics Are Calculated

### Balance and PnL

- **Total portfolio value** = `api_balance + total_position_value` (both
  scoped to the selected wallet — the API wallet by default)
- **UI PnL** = `ui_balance - starting_ui_balance` (UI baseline = seed + topups + grants)
- **API PnL** = `total_portfolio_value - starting_api_balance`
- **UI PnL %** = `ui_pnl / starting_ui_balance * 100`
- **API PnL %** = `api_pnl / starting_api_balance * 100`

`starting_api_balance` is the scoped wallet's **actual** `starting_balance`
(tier-aware: Pro $10,000 / Pro+ $25,000 for the API wallet; a specific
wallet's own baseline when `wallet_id=`), so `api_pnl` here agrees with
`GET /v1/account/balance` and `GET /v1/account/portfolio` for the same
wallet. Before 2026-06-10 this endpoint used a fixed $10,000 baseline
regardless of tier — that caveat no longer applies.

### Open Position Valuation

For each open position:

- **Cost basis** = `avg_entry_price * quantity`
- **Market value** = `current_price * quantity` (if live price exists)
- If no live price is available, cost basis is used as fallback valuation.
- **Unrealized PnL (position)** = `market_value - cost_basis`
- **Unrealized PnL (account)** = `sum(open_position_market_values) - sum(open_position_cost_basis)`

### Realized PnL (Closed Positions)

Closed positions in storage have `quantity = 0`, so realized PnL is reconstructed from filled orders.

For each `(market_id, outcome)` closed position:

- `buy_notional = sum(BUY order notionals)`
- `sell_notional = sum(SELL order notionals)`
- `buy_qty = sum(BUY order quantities)`
- `sell_qty = sum(SELL order quantities)`
- `remaining_qty = buy_qty - sell_qty`
- `settlement_value = exit_price * remaining_qty`
- **position_realized_pnl** = `sell_notional + settlement_value - buy_notional`

Then:

- **realized_pnl** = `sum(position_realized_pnl over closed positions)`

This handles partial closes and settlement-style closes consistently.

Positions with an **unknown outcome** — a `NULL` exit price while shares
remain unsold (`remaining_qty > 0`) — are excluded from `realized_pnl`
entirely: the unsold remainder has no settlement price, and defaulting it to
zero would mis-score a held-to-settlement position as a total loss. Positions
fully flattened by BUY/SELL orders keep their order-derived PnL even without
an exit price (the settlement leg is zero).

### Win Rate and Win/Loss Counts

`win_rate`, `total_wins`, and `total_losses` come from the canonical
closed-position win-rate used across all account surfaces
(`/v1/account/portfolio`, this endpoint, and the profile analytics panel), so
the same account shows the same Win Rate everywhere. Classification is
**price-based** over closed positions with a *known* outcome:

- Win: `exit_price > avg_entry_price`
- Loss: `exit_price < avg_entry_price`
- Break-even: `exit_price == avg_entry_price` (in the denominator, but neither win nor loss)
- Unknown outcome (`NULL` exit price or `NULL` entry price): excluded from
  both the counts and the denominator — never silently counted as a loss

`win_rate` uses the count of classifiable closed positions as denominator:

- **win_rate** = `wins / classifiable_closed_positions * 100`
- `win_rate` is `null` when no closed position has a classifiable outcome
  (unknown is not the same as 0%)

The trade-PnL magnitude stats below (`best_trade_pnl`, `avg_win_pnl`,
`profit_factor`, …) remain **realized-PnL-based** — they describe the size of
realized wins and losses, not their count.

### Profit Factor and Trade PnL Stats

- **Gross profit** = `sum(all positive position_realized_pnl)`
- **Gross loss** = `sum(abs(all negative position_realized_pnl))`
- **profit_factor** = `gross_profit / gross_loss` (when gross_loss > 0)
- **best_trade_pnl** = max positive closed-position PnL
- **worst_trade_pnl** = min negative closed-position PnL
- **avg_win_pnl** = average of positive closed-position PnLs
- **avg_loss_pnl** = average of negative closed-position PnLs

### Drawdown (7d and 30d)

Drawdown is computed from `portfolio_snapshots.total_value` using a **running peak**:

1. Walk snapshots in chronological order
2. Track highest value seen so far (`peak`)
3. Compute drawdown each point: `(value - peak) / peak`
4. Maximum drawdown = most negative drawdown in the period

This avoids overstating drawdown by comparing early values to a later peak.

### Portfolio Diversity and Concentration

For each open position:

- **Effective value** = `market_value` (live price × quantity) when a live price exists,
  or `cost_basis` (entry price × quantity) as fallback when no live price is available.
- Weight per position: `w_i = effective_value / total_invested`
- **HHI** = `sum(w_i^2)`
- **portfolio_diversity_score** = `1 - HHI`
- **largest_position_weight** = largest `w_i`
- **top_3_concentration** = sum of top 3 `w_i`

Weights always sum to approximately 100% across all open positions.

### Equity Returns

Using snapshots in the requested `equity_days` window:

- `return_7d`: first-to-last return among snapshots in last 7 days
- `return_30d`: first-to-last return among snapshots in last 30 days
- `return_all_time`: first-to-last return across the full `equity_days` window

Each is:

- **period_return** = `(end_value - start_value) / start_value * 100`

If there are fewer than two snapshots in a period, return is `null`.

## Data Scope and Semantics

- **Wallet scope**: every section — `open_positions`, `trading_stats`,
  `risk_metrics`, `equity_timeline`, `category_exposure`, `recent_trades`
  and the `balance` cash/baseline — covers the wallet selected by
  `wallet_id` (the **API wallet** when omitted; positions/orders recorded
  before per-wallet attribution count toward the API wallet). `wallet_id=all`
  spans every wallet you own. Only `ui_balance` / `ui_pnl` are always
  MAIN-wallet metrics regardless of scope.
- `open_positions` contains only `status=OPEN` with `quantity > 0`
- Trading stats and realized metrics use only filled orders
- Category exposure is based on open-position values
- Cash/invested percentages are relative to current total portfolio value

### `equity_timeline` — Performance Timeline

| Field | Type | Description |
|-------|------|-------------|
| `return_7d` | string \| null | 7-day return |
| `return_30d` | string \| null | 30-day return |
| `return_all_time` | string \| null | All-time return |
| `peak_value` / `peak_date` | string \| null | Portfolio high watermark |
| `trough_value` / `trough_date` | string \| null | Portfolio low point |
| `current_value` | string \| null | Latest snapshot's total value |
| `snapshots_available` | integer | Number of snapshots in the window (default `0`) |
| `latest_snapshot_at` | string \| null | ISO-8601 UTC time of the most recent snapshot |

### `category_exposure` — Category Breakdown

Array of objects showing exposure per market category:

```json
[
  { "category": "crypto", "position_count": 3, "total_value": "500.00", "weight_percentage": "45.5%" },
  { "category": "sports", "position_count": 2, "total_value": "300.00", "weight_percentage": "27.3%" }
]
```

### `recent_trades` — Recent Filled Orders

Array of the most recent filled orders (count controlled by the
`recent_trades` query param, default 20). Each entry:

| Field | Type | Description |
|-------|------|-------------|
| `order_id` | integer | Order ID |
| `market_id` | string | Market `condition_id` |
| `market_question` | string \| null | Market question text |
| `side` | string | `BUY` or `SELL` |
| `outcome` | string | Outcome name |
| `price` | string | Fill price per share |
| `quantity` | string | Shares filled |
| `notional` | string | Total cash value of the fill |
| `filled_at` | string \| null | ISO-8601 UTC fill timestamp |

See [Trade History](/account/trade-history) for the full per-trade semantics.

### `natural_language_summary`

A pre-computed human-readable summary of the profile:

> "trader123 is a PolySimulator paper trader who joined 45 days ago. Current API balance: $9,750.00 (started at $10,000.00, P&L: -$250.00, -2.50%). Has made 127 trades across 34 markets over 22 active trading days. Win rate: 62.5%. Currently holding 5 open positions worth ~$1,100.00. Top categories: crypto (45.5%), sports (27.3%), politics (18.2%)."

## MCP Server

A standalone MCP server for profile analysis exists and will be published alongside the official SDK (coming soon). Request early access via support if you already have a paid API key.

### Available MCP Tools

| Tool | Description |
|------|-------------|
| `get_profile_analysis` | Full analysis (primary tool) |
| `get_balance` | Quick balance check |
| `get_positions` | Position list with live prices |
| `get_trade_history` | Trade history with pagination |
| `get_equity_curve` | Equity curve snapshots |
