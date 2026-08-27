# Datalake Schema

Source: /market-data/datalake-schema

# Datalake Schema

PolySimulator captures tick-level prediction market data and underlying oracle pricing into high-fidelity Apache Parquet datasets stored in Cloudflare R2 (`polysim-datalake`).

The archive format (version `v3`) is **byte-compatible with the standard 16-column orderbook format**, ensuring existing PyArrow, DuckDB, and Polars pipelines run unmodified across historical datasets.

---

## 1. Orderbook Dataset — `v3/orderbook/polymarket/`

Stores raw order book snapshots and tick events partitioned hourly: `polymarket_orderbook_.parquet`.

### Column Specification

| # | Column | Arrow Type | Nullable | Event Scope | Description |
| -: | :--- | :--- | :--- | :--- | :--- |
| 0 | `timestamp_received` | `timestamp[ms, tz=UTC]` | No | All | UTC timestamp when the ingestion relay received the WebSocket frame. |
| 1 | `timestamp` | `timestamp[ms, tz=UTC]` | No | All | Exchange timestamp from the message payload. |
| 2 | `market` | `fixed_size_binary[66]` | No | All | ASCII bytes of the `0x...` condition ID (66 characters / 66 bytes). |
| 3 | `event_type` | `string` | No | All | `book` \| `price_change` \| `last_trade_price` \| `tick_size_change` |
| 4 | `asset_id` | `string` | No | All | CLOB token ID in decimal string representation. |
| 5 | `bids` | `string` | Yes | `book` | JSON-encoded array of `[["price", "size"], ...]` pairs. |
| 6 | `asks` | `string` | Yes | `book` | JSON-encoded array of `[["price", "size"], ...]` pairs. |
| 7 | `price` | `decimal128(9,4)` | Yes | `price_change`, `last_trade_price` | Price of the trade or price delta. |
| 8 | `size` | `decimal128(18,6)` | Yes | `price_change`, `last_trade_price` | Size in shares (up to 6 decimal places). |
| 9 | `side` | `string` | Yes | `price_change`, `last_trade_price` | `BUY` or `SELL`. |
| 10 | `best_bid` | `decimal128(9,4)` | Yes | `price_change` | Best bid price at the time of the event. |
| 11 | `best_ask` | `decimal128(9,4)` | Yes | `price_change` | Best ask price at the time of the event. |
| 12 | `fee_rate_bps` | `uint16` | Yes | `last_trade_price` | Taker fee rate applied to the trade in basis points. |
| 13 | `transaction_hash` | `string` | Yes | `last_trade_price` | On-chain Polygon transaction hash (if settled on-chain). |
| 14 | `old_tick_size` | `decimal128(9,4)` | Yes | `tick_size_change` | Previous minimum tick size. |
| 15 | `new_tick_size` | `decimal128(9,4)` | Yes | `tick_size_change` | Updated minimum tick size. |

### Technical Details
* **Compression:** Zstandard (ZSTD) compression across all columns.
* **Row Groups:** 1,048,576 rows per row group.
* **Sorting:** Rows sorted by `market` ascending for fast condition ID predicate pushdown.

---

## 2. Underlying Oracle Prices — `v3/underlying/polymarket/`

Captures reference asset spot prices (Binance spot, Chainlink oracles, and equity indices) used for settling binary Up/Down and scalar markets.

### Column Specification

| # | Column | Arrow Type | Nullable | Description |
| -: | :--- | :--- | :--- | :--- |
| 0 | `timestamp_received` | `timestamp[ms, tz=UTC]` | No | UTC timestamp when the ingestion worker received the price tick. |
| 1 | `timestamp` | `timestamp[ms, tz=UTC]` | No | Oracle's native timestamp. |
| 2 | `topic` | `string` | No | Price source category: `crypto_prices` \| `crypto_prices_chainlink` \| `equity_prices` |
| 3 | `symbol` | `string` | No | Normalized symbol (e.g. `btcusdt` for Binance, `btc/usd` for Chainlink, `SPY` for equities). |
| 4 | `full_accuracy_value` | `string` | Yes | Verbatim raw integer/string from the oracle (preserves precision for settlement). |
| 5 | `value` | `float64` | Yes | Standard floating-point price representation. |

---

## Reader Recipes

The archive is **private**. These recipes do not publish credentials or
raw objects. You must supply a local download/mount **or** an endpoint
configuration you already hold. Never commit `BACKTEST_RO_*`,
`INGEST_RW_*`, or R2 access keys.

  PolySimulator does **not** ship a public download URL, a sample
  parquet, or a credential. If you do not already have an authorized
  local copy or an endpoint, these recipes will not run — that is
  intentional.

Configure one of:

| Variable | Meaning |
| --- | --- |
| `POLYSIM_DATALAKE_ROOT` | Local directory you downloaded or mounted (contains `v3/orderbook/polymarket/`). |
| `POLYSIM_DATALAKE_URI` | Optional object-store URI (`s3://…` / `https://…`) **you** configure. The docs never publish a host or key. |
| Standard S3-compatible auth env vars | Only if you already hold a read-only token. Leave unset for a local mount. Do not paste values into docs or git. |

```bash
# Local mount you created (recommended)
export POLYSIM_DATALAKE_ROOT="$HOME/data/polysim-datalake"

# Or an endpoint you already have — never a committed secret
# export POLYSIM_DATALAKE_URI="s3://your-bucket/v3/orderbook/polymarket/"
# Point your S3-compatible client at an endpoint you already hold.
```

### Reading with PyArrow

```python
import os
from pathlib import Path

import pyarrow.dataset as ds

root = os.environ.get("POLYSIM_DATALAKE_ROOT")
uri = os.environ.get("POLYSIM_DATALAKE_URI")
if not root and not uri:
    raise SystemExit(
        "Set POLYSIM_DATALAKE_ROOT to a local mount you downloaded, "
        "or POLYSIM_DATALAKE_URI to an endpoint you already hold."
    )

source = uri or str(Path(root) / "v3" / "orderbook" / "polymarket")
dataset = ds.dataset(source, format="parquet")

trades_filter = (
    (ds.field("event_type") == "last_trade_price")
    & (ds.field("asset_id") == os.environ.get("POLYSIM_TOKEN_ID", ""))
)
table = dataset.to_table(filter=trades_filter) if os.environ.get("POLYSIM_TOKEN_ID") else dataset.head(10)
print(f"Loaded {table.num_rows} rows from {source}")
```

### Reading with DuckDB

```python
import os
from pathlib import Path

import duckdb

root = os.environ.get("POLYSIM_DATALAKE_ROOT")
uri = os.environ.get("POLYSIM_DATALAKE_URI")
if not root and not uri:
    raise SystemExit(
        "Set POLYSIM_DATALAKE_ROOT to a local mount you downloaded, "
        "or POLYSIM_DATALAKE_URI to an endpoint you already hold."
    )

glob = uri or str(Path(root) / "v3" / "orderbook" / "polymarket" / "*.parquet")
con = duckdb.connect()
con.execute(
    """
    SELECT
        date_trunc('hour', timestamp) AS hour,
        asset_id,
        sum(price * size) AS notional_usd,
        count(*) AS trade_count
    FROM read_parquet(?)
    WHERE event_type = 'last_trade_price'
    GROUP BY 1, 2
    ORDER BY 1 DESC
    LIMIT 20
    """,
    [glob],
)
print(con.fetchall())
```
