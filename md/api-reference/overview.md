# API Reference

Source: /api-reference/overview

# API Reference

The PolySimulator API is a **REST + WebSocket** surface. Every endpoint
below is generated from a **curated, stable OpenAPI spec**
([`/openapi.json`](https://docs.polysimulator.com/openapi.json))
and ships with an interactive playground — paste your `ps_live_…` key
into the **Authorization** widget and you can fire requests against the
live production environment from this browser tab.

  The published reference is generated from a curated, stable OpenAPI
  spec. The live backend at
  [`api.polysimulator.com/openapi.json`](https://api.polysimulator.com/openapi.json)
  may expose additional unstable / legacy routes (v0 endpoints,
  internal helpers) that aren't shown here — only the endpoints
  documented here are part of the supported public surface.

  **URL convention for endpoint pages**: each endpoint has a dedicated
  page at `/api-reference/{operationId}` (camelCase) — e.g.,
  `POST /v1/orders` lives at [/api-reference/postOrder](/api-reference/postOrder),
  `GET /v1/markets/{condition_id}` at [/api-reference/getMarket](/api-reference/getMarket).
  For convenience, natural API-path guesses (e.g. `/api-reference/v1/orders`,
  `/api-reference/v1/markets/0xabc...`) are auto-redirected to the
  canonical operationId page, so you can paste live API paths straight
  into the URL bar and land on the right reference.

  
    How `X-API-Key`, bootstrap, and the WebSocket JWT flow work.
  
  
    From signup to first trade in under two minutes.
  

---

## Endpoint groups

| Group | Coverage |
|-------|----------|
| **API Keys** | Bootstrap, create, list, revoke, tier metadata, WebSocket token mint |
| **Account** | Balance, equity curve, portfolio, positions, trade history |
| **Trading** | Place / cancel / batch / list orders, cancel-all, market cancel |
| **Market Data** | Markets list + lookup, candles, prices, order book |
| **CLOB Read (Public)** | Polymarket-shape book / midpoint / spread / price endpoints |
| **CLOB Compat** | Polymarket-shape `POST /v1/order`, `POST /v1/orders`, `GET /v1/data/orders` |
| **WebSocket** | `/v1/ws/prices` + `/v1/ws/executions` subscription endpoints |
| **Simulation** | Historical fill, coverage, book replay, and curated backtests. Feature-dark until `FEATURE_SIMULATION_API_ENABLED` is on. |
| **Health** | `/v1/health`, `/v1/health/ready`, `/v1/status`, `/v1/me` |

Use the sidebar to drill into any group. Each endpoint page shows the
request schema, response schema, error codes, and a copy-pasteable
`curl` / TypeScript / Python snippet.

---

## Base URL

| Environment | URL |
|-------------|-----|
| Production | `https://api.polysimulator.com` |

Most endpoints in this reference are mounted under `/v1` —
example: `GET https://api.polysimulator.com/v1/account/balance`.
The one documented exception is `/api/beta/cohort-status`, a public
read-only endpoint used by the pricing page to surface residual
cohort inventory.

---

## Authentication at a glance

Most endpoints require an `X-API-Key` header. Public exceptions
that work without a key: `/v1/health`, `/v1/health/ready`,
`/api/beta/cohort-status`, and the public CLOB-read endpoints under
**CLOB Read (Public)**. Note `/v1/keys/bootstrap` is **not** public —
it requires a Supabase `Authorization: Bearer ` JWT
(returns `401 MISSING_AUTH` without one), it just doesn't take an
`X-API-Key` because it mints your first key. Each endpoint's
individual page lists its own auth requirement at the top — check
there if unsure. The [Authentication guide](/authentication) covers
key minting, rotation, revocation, and the short-lived WebSocket JWT
flow.

```bash
curl https://api.polysimulator.com/v1/account/balance \
  -H "X-API-Key: ps_live_..."
```

  The interactive playground on every endpoint page lets you store your
  key once and replay requests without retyping it. Keys you paste into
  the playground are stored locally in your browser and attached to
  playground requests as the `X-API-Key` header — they're not transmitted
  anywhere until you click **Send**, at which point they're sent to
  `api.polysimulator.com` in the request header like any other API
  client call.

---

## Polymarket compatibility

PolySimulator is wire-compatible with `py-clob-client` and the Polymarket
CLOB JSON shape for the endpoints under the **CLOB Read (Public)** and
**CLOB Compat** groups. The
[CLOB compatibility guide](/concepts/clob-compatibility) lists every
identical-vs-deviation contract item — read it before you port a bot.

For the few documented deviations (the `X-Polysim-Code` machine header,
the `402 UPGRADE_REQUIRED` enrichment, paper-trading-only EIP-712 signing),
the [Polymarket Raw HTTP guide](/concepts/pm-raw-http) walks through
each one with side-by-side request/response examples.

---

## Rate limits + tiers

| Tier | Burst (req/s) | Sustained (req/min) | WebSockets | Batch size |
|------|---------------|---------------------|------------|------------|
| Free | 2 | 120 | 1 | 1 |
| Pro | 10 | 600 | 3 | 5 |
| Pro+ | 30 | 1,800 | 10 | 10 |
| Enterprise | 100 | 6,000 | 50 | 25 |

`X-RateLimit-Limit`, `X-RateLimit-Remaining`, and `X-RateLimit-Reset`
are present on authenticated responses. On legacy keyed and unauthenticated
public endpoints, `429` responses return `Retry-After`, `X-Polysim-Code`,
and `X-Request-Id` (full quota metadata headers are returned on authenticated
private endpoints and flagged token resolver routes). `X-RateLimit-Tier` is
present on authenticated responses only — unauthenticated requests
metered against IP-only buckets omit it. The authoritative source for
these numbers is `GET /v1/keys/tiers`. See
[Rate Limits](/concepts/rate-limits) for back-off strategy. Legacy
cohort keys (if any remain) run at their granted tier — admin
beta-issued keys are enterprise-tier — then auto-downgrade to free +
read-only at the key's `beta_until` cutoff.

---

## LLM-targeted reference

If you're driving this API from an AI coding assistant, point it at
[`/llms.txt`](/llms.txt) — a single ~50 KB markdown blob with every
endpoint, schema, and Polymarket parity note in a model-friendly
format. Cursor / Claude Code / Continue / Windsurf can also ingest the
docs via the hosted [MCP server or Context7](/ai-assistants).
