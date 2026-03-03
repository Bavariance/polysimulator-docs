# PolySimulator API Documentation

> **This repository is auto-synced from the private [polysimulator](https://github.com/Bavariance/polysimulator) repo.**  
> Do not open issues or PRs here — the content is overwritten on every sync.

---

## What is PolySimulator?

PolySimulator is a virtual trading environment for [Polymarket](https://polymarket.com) prediction markets. It provides a complete HFT API for algorithmic trading bots:

- **Paper trading**: Trade with simulated $1,000 using real market data
- **Full CLOB API**: Drop-in compatible with Polymarket's native API
- **Instant live migration**: Change only your base URL and credentials to trade for real

**Website**: [polysimulator.com](https://polysimulator.com)  
**API Base URL**: `https://api.polysimulator.com/v1`  
**Docs**: [docs.polysimulator.com](https://docs.polysimulator.com)

---

## For AI Coding Assistants

- [`/llms.txt`](./llms.txt) — Machine-readable API reference (endpoints, auth, examples)
- [`/openapi.json`](./openapi.json) — Full OpenAPI 3.0 specification
- [`/asyncapi.json`](./asyncapi.json) — WebSocket API specification

**Context7 library ID**: `/bavariance/polysimulator-docs`  
**NPM**: _not applicable — this is an API, not a library_

---

## Documentation Structure

| Path | Contents |
|------|----------|
| `authentication.mdx` | API key setup, bootstrap flow, OAuth |
| `quickstart.mdx` | 5-minute getting started guide |
| `introduction.mdx` | Platform overview |
| `trading/` | Order types, CLOB compatibility, market orders |
| `account/` | Balance, positions, portfolio, trade history |
| `market-data/` | Market listings, prices, order book, candles |
| `websockets/` | Real-time price & execution feeds |
| `bots/` | Algorithmic trading patterns & examples |
| `concepts/` | Core concepts: prediction markets, probabilities |

---

## Sync Status

This repo is updated automatically when `docs-site/` changes are merged to `main` or `staging` in the private repo. The sync runs via a GitHub Actions workflow.

---

## License

Documentation content © 2025 PolySimulator. All rights reserved.  
Not for redistribution without permission.
