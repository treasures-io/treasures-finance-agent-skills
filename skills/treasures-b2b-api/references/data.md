# Data — discovery, portfolio, trades

Load when reading `/stocks/*`, `/portfolio`, or `/trades`. All four are unauthenticated public reads (no `ownership_proof`).

## `GET /stocks/tickers`

Catalog: every tradable ticker with chain availability, per-protocol token addresses, and share multipliers. Cached 5 min server-side; cheap to poll. Use it to populate pickers, validate symbols, decide `chain`/`protocol` options, and recover token addresses for `approve()`.

```jsonc
{
  "tickers": [{
    "ticker": "AAPL",
    "name": "Apple Inc.",
    "available_chains": ["sol", "eth", "robinhood"],
    "ondo":    { "sol_address": "...", "eth_address": "0x...", "share_multiplier": "0.4818", "token_ticker": "AAPLon" },
    "xstocks": { "sol_address": null, "eth_address": null, "share_multiplier": null, "token_ticker": null },
    "robinhood": { "address": "0x...", "share_multiplier": "1", "token_ticker": "AAPL" }
  }]
}
```

A listing block with all its fields `null` means that protocol/venue doesn't list this ticker. `available_chains` is the union across all three (`sol` · `eth` · `robinhood`). **The `robinhood` block has a different shape** — Robinhood Chain (4663) is single-cell, so one `address` (decimals stay internal) instead of the `sol_address`/`eth_address` pair. `token_ticker` is the symbol that appears on `/portfolio` + `/trades` rows — Ondo `<TICKER>on`, xStocks `<TICKER>x`, Robinhood the **bare** `<TICKER>` — you never derive it yourself.

## `GET /stocks/prices?tickers=AAPL,TSLA,MSFT`

Live price snapshot for a targeted set. Comma-separated, up to **50 per call**. Server enforces `/^[A-Z0-9.-]{1,10}$/i` per item, uppercased server-side.

```jsonc
{
  "prices": [{
    "ticker": "NFLX",
    "tradfi": {   // null when the tradfi feed is unavailable
      "current_price_usd": "89.33", "change_24h_pct": "-0.35694",
      "market_cap_usd": "376150764000", "pe_ttm": "28.21", "as_of": 1779220801
    },
    "onchain": {   // per-protocol/venue; any side null when no listing or no price-feed data
      "ondo":      { "share_price_usd": "88.8680", "premium_vs_tradfi_pct": "-0.517", "volume_24h_usd": "4732002" },
      "xstocks":   { "share_price_usd": "88.3500", "premium_vs_tradfi_pct": "-1.097", "volume_24h_usd": "2649" },
      "robinhood": { "share_price_usd": "88.5000", "premium_vs_tradfi_pct": "-0.930", "volume_24h_usd": "1200" }
    }
  }]
}
```

`onchain.ondo`, `onchain.xstocks`, and `onchain.robinhood` are independent — pick any or all. `premium_vs_tradfi_pct` (negative = on-chain cheaper) is computed against `tradfi.current_price_usd`; when `tradfi` is `null`, each venue's premium is `null` too. Use for quote-time comparison, P&L marks, "current price" UX.

## `GET /portfolio?sol_wallet=...&eth_wallet=...&source=all|internal`

Live reconciled USDC + tokenized-stock holdings for a wallet pair. Cached 30s per pair. Pass **either or both** wallets (single-wallet valid; neither → `400 invalid_request`). `source` (default `all`) is a **column toggle, not a row filter** — positions are always the on-chain holdings; `internal` additionally populates the three cost-basis columns below.

```jsonc
{
  "positions": [{
    "ticker": "AAPL", "token_ticker": "AAPLon", "chain": "sol", "protocol": "ondo", "token_address": "...",
    "tokens": "0.4988",
    "shares": "0.4988",                    // null when catalog row is missing share_multiplier
    "usd_per_token": "234.78",             // null when the price feed has no spot for this mint
    "usd_per_share": "234.78",             // null when usd_per_token null OR share_multiplier missing
    "usd_value": "117.10",                 // null when usd_per_token null
    "shares_internal_only": "0.4988",      // null in source=all; internal-lot shares (Treasures-bought, still held)
    "avg_entry_price_per_share": "210.00", // null in source=all; weighted-avg cost basis per share
    "unrealized_pnl": "12.35"              // null in source=all or when usd_per_token null
  }],
  "usdc": { "sol": "53.21", "eth": "0.00" },
  "usdg": { "robinhood": "100.00" },
  "as_of": 1730000050, "is_cached": true
}
```

`shares`, `usd_per_token`, `usd_per_share`, `usd_value` are each independently nullable — a position with a price-feed outage still surfaces with `tokens` populated so you can hold the row and re-render USD next poll. **Default any null to "unknown", never "0".** Tokens acquired outside Treasures reconcile in on the next read and emit a synthetic `external` row in `/trades`. **Exception — Robinhood Chain has no external-row reconciler:** you must **submit** each Robinhood-chain trade via `/trade/submit` for it to appear in `/trades` at all — it enters as `broadcast` and reaches `completed` via the same status poll / backfill as `eth` (see [`trading.md`](trading.md#robinhood)). An **unsubmitted** Robinhood trade never appears in `/trades` and carries no cost basis, though its balance still shows under `/portfolio` (live read).

The three `source=internal` columns are derived from **completed internal trades only** (see [Internal-only P&L](#internal-only-pl) below). Caveat: because off-platform transfers are ignored by the basis, `shares_internal_only` can exceed the on-chain balance after you move tokens out — treat it as "shares bought via Treasures and not yet sold via Treasures", not a custody figure.

**Robinhood-Chain rows.** A wallet holding Robinhood Stock Tokens on chain 4663 surfaces extra positions with `chain: "robinhood"`, `protocol: "robinhood"`, and a **bare** `token_ticker` (no `on`/`x` suffix). They read **live** from the caller's own 4663 EOA (keyed off `eth_wallet`) on **every** call — the 30s cache and the `is_cached` flag cover only the sol/eth custodial cells, so a Robinhood row is never stale (and costs a 4663 RPC read each request). A 4663 RPC blip omits the Robinhood rows rather than failing the response. The same live read also fills **`usdg.robinhood`** — the caller's USDG (the 4663 base currency, the Robinhood-side analog of `usdc.sol`/`usdc.eth`): a wallet holding only USDG (no Stock Tokens yet) still shows its cash there. `"0"` when no `eth_wallet`, the venue is dormant, or the 4663 read fails. Two caveats specific to the position cells:

- **Cost basis is on-chain-observed.** `tokens`/`shares` are exact (live `balanceOf`), and the `source=internal` P&L columns (`avg_entry_price_per_share`, `unrealized_pnl`, and `realized_pnl` on `/trades`) are realized from the **mined fill** — the same fidelity as `eth`, not a quote-time estimate.
- **Discoverable via `/stocks/tickers`.** The venue's `robinhood` listing block (`address`, `share_multiplier`, bare `token_ticker`) and a `robinhood` entry in `available_chains` are surfaced by [`/stocks/tickers`](#get-stockstickers); `/stocks/prices` carries an `onchain.robinhood` price cell — so Robinhood stocks are discovered and valued the same way as sol/eth. (Getting USDG onto 4663 is still your own; see [`trading.md`](trading.md#robinhood).) Note: listing here does **not** imply tradeable — only the liquid marquee names fill (others → `no_routes`).

## `GET /trades?sol_wallet=...&eth_wallet=...&limit=50&offset=0&source=all|internal`

History for the wallet pair: Treasures-executed (`source: "internal"`) + reconciler-detected drift (`source: "external"` — transfers in/out, dividend rebases on xStocks-eth). Pass **either or both** wallets (same rule as `/portfolio`). `limit` 1–200 (default 50), `offset` 0–5000 (default 0). The `source` **query param** (default `all`) is a column toggle — it does **not** filter which rows return (don't confuse it with the per-row `source` field); `internal` populates the three P&L columns below.

```jsonc
{
  "trades": [{
    "trade_id": "trd_...",
    "source": "internal",
    "side": "buy" | "sell" | null,       // ALWAYS null for source=external (reconciler never classifies drift); buy/sell only on internal
    "ticker": "AAPL", "token_ticker": "AAPLon", "chain": "sol", "protocol": "ondo",
    "tx_hash": "...", "order_hash": null, // tx_hash/order_hash split — see trading.md; external rows: both null
    "tokens": "0.4988",                  // signed for source=external (negative = out)
    "shares": "0.4988",                  // same sign as tokens on source=external
    "usdc_amount": "100",                // null for source=external
    "usd_per_share": "200.48",           // null in source=all and on non-completed rows; settled execution price (usdc_amount / shares)
    "avg_entry_price_per_share": "180.00", // null in source=all; running internal cost basis per share at this row
    "realized_pnl": "10.24",             // null in source=all, on buys, and on external rows — see P&L below
    "submitted_at": 1730000000,
    "status": "completed" | "broadcast" | "failed" | "external"
  }],
  "next_offset": 50                      // number when more pages exist; null on the last page
}
```

> `/trades` uses its own status enum (`completed | broadcast | failed | external`) — it surfaces in-flight trades as `broadcast` directly rather than collapsing to `pending` like `/quote/{id}/status` does. Loop with `while (next_offset !== null)`.

Robinhood-Chain trades appear here too — `chain: "robinhood"`, `protocol: "robinhood"`, bare `token_ticker`, `tx_hash` = the settlement hash (in-flight rows carry `order_hash` instead) — but only once you submit them (`/trade/submit`); there is no reconciler to backfill an unsubmitted Robinhood trade (see the `/portfolio` note above).

## Internal-only P&L

Passing `source=internal` to `/portfolio` or `/trades` computes cost basis from a replay of **completed internal trades only** — external (reconciler drift) and in-flight/failed rows are excluded. `realized_pnl` matches **[Hyperliquid's `closedPnl`](https://hyperliquid.gitbook.io/hyperliquid-docs/trading/entry-price-and-pnl)**:

- **Entry price** is the weighted average of your internal buys; a buy re-weights it, a sell leaves it unchanged.
- **Buys** carry `realized_pnl: null` (P&L is realized only when you close).
- **Sells** realize `(execution_price − avg_entry) × shares_closed`, **gross of fees** (fees are not subtracted, matching Hyperliquid's field).
- Selling more than you bought via Treasures (possible after depositing tokens from elsewhere) realizes only the covered portion; the excess realizes nothing.

To total realized P&L over a period, sum `realized_pnl` across the sell rows in the window (it's per-trade, not cumulative).
