# Wallets

Source: /account/wallets

# Wallets

PolySimulator separates trading balance into distinct **wallets** so that bot
trading, manual UI trading, and paid sandbox experiments don't interfere with
each other. The wallet that fills your order depends on **how the order was
authenticated**.

## Wallets at a glance

| Wallet | Starting balance | Reset / top-up | Used by |
|---|---|---|---|
| **API** | Plan-dependent baseline (Free: **$100**, Pro: $10,000, Pro+: $25,000) | Resets to your tier baseline; cooldown is `API_RESET_COOLDOWN_DAYS` (currently 0). **Not available on Free** — returns `402 UPGRADE_REQUIRED` | Any request authenticated with `X-API-Key` |
| **MAIN** | $1,000 | Free resets capped per 30 days (Free: 1, Pro: 4, Pro+: unlimited). Paid `topup_main_reset` SKU bypasses the cooldown. | UI trading on `polysimulator.com` |
| **SANDBOX** | Plan-dependent baseline (Free: $0, Pro: $10,000, Pro+: $25,000) | Paid top-ups (`topup_500`, `topup_2000`, `topup_10000`) credit on top of the baseline | Pro-tier UI users running paid experiments |
| **COMPETITION** | Varies per event | Event-scoped, frozen during competition | Event entrants only |

  As an API user, you only need to think about the **API wallet**. The UI MAIN
  and SANDBOX wallets are for human-trader use on the website.

## The API wallet

When you create your first key with `POST /v1/keys/bootstrap`, your account is
seeded with your tier's API wallet baseline ($10,000 on Pro, $25,000 on Pro+;
Free-tier API keys are read-only and have no API wallet seeded). Every order
you place via `X-API-Key` debits this wallet — `GET /v1/account/balance`
always reports the API wallet balance for API-authenticated requests.

```bash
curl -H "X-API-Key: $POLYSIM_API_KEY" \
  https://api.polysimulator.com/v1/account/balance
# Pro tier → {"balance": "9745.20", "starting_balance": "10000.00", ...}
# Pro+ tier → {"balance": "24745.20", "starting_balance": "25000.00", ...}
```

### Resetting the API wallet

You can reset the API wallet to your tier's starting balance at any time —
this also closes any open API-sourced positions. Pro keys reset to $10,000;
Pro+ keys reset to $25,000.

```bash
curl -X POST -H "X-API-Key: $POLYSIM_API_KEY" \
  https://api.polysimulator.com/v1/account/reset-api-balance
# Pro → {"message": "API balance reset to $10,000.00",
#        "new_api_balance": "10000.00",
#        "positions_closed": 3,
#        "cooldown_days": 0}
# Pro+ → {"message": "API balance reset to $25,000.00",
#         "new_api_balance": "25000.00",
#         "positions_closed": 0,
#         "cooldown_days": 0}
```

The reset response carries four fields:

| Field | Type | Description |
|-------|------|-------------|
| `message` | string | Human-readable confirmation |
| `new_api_balance` | string | The post-reset balance (your tier baseline) |
| `positions_closed` | integer | Number of open API positions force-closed by the reset |
| `cooldown_days` | integer | Days until the next reset is allowed (`0` during the beta period) |

  **On a paid tier, API wallet resets are currently uncapped** — the cooldown
  is `API_RESET_COOLDOWN_DAYS`, presently 0, and a daily cooldown may be
  enabled later. Use them freely to test strategies from a clean baseline.

  **Not available on Free.** Reset returns `402 UPGRADE_REQUIRED`: the $100
  free budget is deliberately non-renewable, so spending it is the upgrade
  prompt rather than a reset away.

## Scoping account reads to a wallet

The account read endpoints accept a `wallet_id` query parameter with three
forms (case-insensitive keywords):

| Value | Meaning |
|---|---|
| *(omitted)* | **Your API wallet** — the default on every account read |
| `api` | Your API wallet, explicitly (includes legacy rows recorded before per-wallet attribution) |
| `all` | Every wallet you own — UI MAIN/SANDBOX/COMPETITION included |
| `` | One specific wallet you own (ids from `GET /v1/me/wallets`); 404 `WALLET_NOT_FOUND` otherwise. MAIN/API wallet ids fold in legacy rows recorded before per-wallet attribution |

Supported on [Positions](/account/positions),
[Trade History](/account/trade-history),
[Profile Analysis](/account/profile-analysis),
[Portfolio](/account/portfolio) and [Equity Curve](/account/equity-curve).
[Balance](/account/balance) is always API-wallet scoped. Any other
`wallet_id` value returns 422 `VALIDATION_FAILED`.

```bash
# Default — API wallet only
curl -H "X-API-Key: $POLYSIM_API_KEY" \
  https://api.polysimulator.com/v1/account/positions

# Everything you own, including UI wallets
curl -H "X-API-Key: $POLYSIM_API_KEY" \
  "https://api.polysimulator.com/v1/account/positions?wallet_id=all"
```

  **Migration note (2026-06-10):** `GET /v1/account/positions`,
  `GET /v1/account/history` and `GET /v1/account/profile-analysis`
  previously defaulted to **all wallets** when `wallet_id` was omitted.
  Since 2026-06-10 they default to the **API wallet**, consistent with
  Balance/Portfolio/Equity. Pass `wallet_id=all` to keep the old behaviour.

## Why the wallets are separate

The split exists so that:

- **Bots can crash without nuking your UI portfolio.** A runaway loop that
  blows the API wallet doesn't touch your $1,000 MAIN balance or any SANDBOX
  experiment.
- **The leaderboard stays clean.** Only MAIN-wallet trades count toward
  leaderboard ranking — bot performance is tracked separately on the API beta
  dashboard.
- **Paid top-ups don't mix with sacred state.** Top-ups always credit
  SANDBOX wallets; the MAIN $1,000 is preserved as the canonical
  "fresh-account" baseline for any user.

## Top-ups (paid)

Top-ups are paid Stripe purchases that credit a **SANDBOX wallet** for users
who want extra paper-trading capital on the UI side. They do **not** affect
the API wallet.

| SKU | Price | Credits |
|---|---|---|
| `topup_500` | $2.99 | $500 to SANDBOX |
| `topup_2000` | $4.99 | $2,000 to SANDBOX |
| `topup_10000` | $9.99 | $10,000 to SANDBOX |
| `topup_main_reset` | $2.99 | Resets MAIN to its seed baseline ($1,000); credits no new funds (bypasses the reset cooldown) |

If you only use the API, you can ignore this entire section.

## See also

- [Balance](/account/balance) — `GET /v1/account/balance` reference
- [Portfolio](/account/portfolio) — combined balance + positions snapshot
- [API keys](/concepts/api-keys) — bootstrap and key management
