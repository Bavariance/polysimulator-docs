# PolySimulator API

Source: /introduction

# PolySimulator HFT API v1

Build, test, and deploy prediction market trading bots in a risk-free environment
powered by **real-time Polymarket data**. When you're ready, switch to live trading
by changing a single environment variable.

  

  
    Get your API key and place your first trade in under 2 minutes.
  
  
    `pip install polysimulator` — the official `PolySimClient`. Porting a `py-clob-client` or py-sdk bot? Drop-ins included.
  
  
    Interactive playground — test every endpoint with your API key.
  
  
    Copy-paste a working Python trading bot and start experimenting.
  

---

## Why PolySimulator?

| Feature | Description |
|---------|-------------|
| **Real Data** | Live mid-prices, bids, asks from Polymarket's CLOB order book |
| **Book Walking** | Market orders fill against real order book depth — no infinite liquidity |
| **Slippage Protection** | `price` field acts as worst-price limit — identical to Polymarket |
| **Limit Orders** | GTC / FOK / GTD time-in-force with sub-second matching engine |
| **Batch Orders** | Tier-dependent batch size (free 1, pro 5, pro+ 10, enterprise 25; see `GET /v1/keys/tiers`) |
| **WebSocket Feeds** | Real-time price ticks and execution notifications |
| **OHLCV Candles** | Historical candlestick data for backtesting |
| **CLOB-Compatible** | `/v1/clob/order` mirrors Polymarket's schema — URL-swap migration |
| **String Numerics** | All price/size/balance values are strings — no float precision loss |

---

## Architecture Overview

Your bot talks to a single endpoint. The API streams **live order books, prices,
and markets from Polymarket's CLOB and Gamma feeds** and serves them from an
**in-memory cache (Redis)** — so your strategy reads current market data with
low latency, without a database round-trip on the hot path. Orders execute
against your **simulated paper account**, and the wire contract is identical to
live Polymarket, so going live is a config change — not a rewrite.

```mermaid
flowchart TD
    PM["Polymarketlive CLOB + Gamma"]
    Cache["In-memory price cacheRedis, low-latency reads"]
    API["PolySimulator API v1"]
    Bot["Your Trading BotREST + WebSocket"]
    Acct["Your paper accountbalance, positions, orders"]
    PM -->|realtime feed| Cache
    Cache -->|low-latency reads| API
    Bot -->|orders and queries| API
    API -->|live prices and order book| Bot
    API -->|simulated fills| Acct
```

---

## Base URL

Production: **`https://api.polysimulator.com`**

Endpoints are mounted under `/v1` — append the version + path to the
base, e.g. `GET https://api.polysimulator.com/v1/health`. (The one
documented exception is `/api/beta/cohort-status`, a public read-only
endpoint used by the pricing page for residual cohort inventory.)
Keep the base URL **without** `/v1` so each path can prepend its own
`/v1` — that's the convention the [Quick Start](/quickstart) and
[API Reference](/api-reference) use.

---

## One Config Change to Go Live

  
    Develop and test your bot against the virtual API. Free keys are
    read-only (no API wallet). Paid Pro / Pro+ keys trade a simulated
    API wallet ($10,000 on Pro, $25,000 on Pro+).
  
  
    Review your portfolio, trade history, and equity curve to confirm your edge.
  
  
    Set `TRADING_MODE=live` and add your Polymarket credentials. Same API, real money.
  

  The `/v1/clob/order` endpoint uses the same schema as Polymarket's real CLOB API.
  When migrating to live, you can even point your bot directly at `clob.polymarket.com`
  with zero code changes.

---

## AI Agent Integration (MCP)

PolySimulator docs are indexed for the most common AI-coding-tool
integrations — a hosted **MCP server**, `llms.txt`, the machine-readable
OpenAPI spec, and **Context7** — so Cursor, Windsurf, Claude Desktop,
Continue, and similar assistants get live access to the current API
schema. See [Use with AI assistants](/ai-assistants) for setup configs
and example prompts.

---

## What's Next?

  
    Learn how API key auth works and create your first key.
  
  
    Execute market and limit orders with string-precision numerics.
  
  
    Subscribe to real-time price updates and execution notifications.
  
  
    Handle rate limits, errors, and build retry strategies for your bot.
