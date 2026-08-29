# Data — discovery, portfolio, trades

Load when reading `/stocks/*`, `/portfolio`, or `/trades`. All four are unauthenticated public reads (no `ownership_proof`).

## `GET /stocks/tickers`

Catalog: every tradable ticker with chain availability, per-protocol token addresses, and share multipliers. Cached 5 min server-side; cheap to poll. Use it to populate pickers, validate symbols, decide `chain`/`protocol` options, and recover token addresses for `approve()`.

```jsonc
{
  "tickers": [{
    "ticker": "AAPL",
    "name": "Apple Inc.",
    "available_chains": ["sol", "eth", "robinhood", "base"],
    "ondo":    { "sol_address": "...", "eth_address": "0x...", "share_multiplier": "0.4818", "token_ticker": "AAPLon" },
    "xstocks": { "sol_address": null, "eth_address": null, "share_multiplier": null, "token_ticker": null },
    "robinhood": { "address": "0x...", "share_multiplier": "1", "token_ticker": "AAPL" },
    "coinbase":  { "address": "0x...", "share_multiplier": "1", "token_ticker": "AAPL" }
  }]
}
```

A listing block with all its fields `null` means that protocol/venue doesn't list this ticker. `available_chains` is the union across all four (`sol` · `eth` · `robinhood` · `base`). **The `robinhood` and `coinbase` blocks have a different shape** — Robinhood Chain (4663) and Base (8453) are single-cell venues, so each carries one `address` (decimals stay internal) instead of the `sol_address`/`eth_address` pair. `token_ticker` is the symbol that appears on `/portfolio` + `/trades` rows — Ondo `<TICKER>on`, xStocks `<TICKER>x`, Robinhood and Coinbase the **bare** `<TICKER>` — you never derive it yourself.

**This endpoint is the authority on where a ticker trades — especially for `base`.** The Coinbase B20 contracts were deployed unminted, and Treasures hides a cell until real supply exists: an unminted ticker shows `coinbase.address: null`, omits `base` from `available_chains`, and returns `422 no_routes` if you quote it anyway. The listed set therefore **grows over time** as Coinbase mints. Re-read this endpoint (5-min cache) instead of hardcoding a venue list.

<a id="tradability-warnings"></a>
### Tradability warnings

A background probe quotes a ladder of buy sizes against every listed venue — and watches whether real orders settle — then publishes what it found. Its verdicts ride `/stocks/tickers` (and the `/stocks` list, same fields) at **two grains at once** — per venue cell inside the listing blocks, and per ticker at the top level of the item:

| Field | Values | Meaning |
| --- | --- | --- |
| `min_trade_size_usd` | integer USD | The smallest size measured to actually fill. Advisory: a smaller order is still accepted and still sent to the venue. |
| `tradability` | `"thin"` \| `"untradable"` | `thin` = fills, but not at every size. `untradable` = a full measurement window passed with no fill at any size. **Neither is a block** — both are still quotable and submittable. |
| `warn_reason` | `"settlement_failure"` \| `"no_price_feed"` \| `"no_fill_window"` \| `"low_volume"` | Why the warning was raised. Per-reason action table: [`trading.md`](trading.md#tradability). |
| `thin_since` | ISO-8601 string | When the warning was first raised. Cell grain only, and present exactly when that cell's `tradability` is. |

Which keys appear where:

| Grain | Where | Keys |
| --- | --- | --- |
| cell | `ondo`, `xstocks` blocks — each spans **two** cells (sol + eth) | chain-suffixed: `sol_min_trade_size_usd`, `sol_tradability`, `sol_warn_reason`, `sol_thin_since`, and the four `eth_*` twins |
| cell | `robinhood`, `coinbase` blocks — one cell each | bare: `min_trade_size_usd`, `tradability`, `warn_reason`, `thin_since` |
| ticker | top level of the item, beside `ticker`/`name` | `min_trade_size_usd`, `tradability`, `warn_reason` — **no `thin_since`** (one stamp cannot date several cells' warnings) |

- **The ticker-level roll-up is deliberately asymmetric.** `min_trade_size_usd` is the **cheapest floor across cells** (a buy routes to whichever cell can serve it), and the word follows that floor — `thin` beside a floor, `untradable` without one. `tradability` is emitted only when **every measured cell is warned**, never when just one is: a ticker with one unwarned cell is healthy somewhere, and labelling the whole ticker would hide a live listing. `warn_reason` at this grain is the **strongest** reason across the warned cells, not the cheapest cell's. So read the per-cell keys, not the ticker keys, to decide which venue to trade.
- **Absent ≠ null, everywhere these fields appear.** They are omitted rather than sent as `null`. An absent `min_trade_size_usd` means no minimum is known (fall back to $3), never that there is none — and an absent `tradability` means unmeasured or unwarned, never "healthy". `warn_reason` can be absent beside a *present* `tradability` too: that warning predates the reason field, so read it as "reason not recorded", never "no reason".
- **`available_chains` is untouched by any of this.** It answers "where does this token exist", so a warned venue still appears there — and a holder's chain never vanishes from a sell flow.
- **Not stable over a cell's lifetime.** A `low_volume` warning flips to `settlement_failure` once a real order dies on that cell. Re-read (5-min cache) rather than caching the value yourself.
- The same three warning fields ride **every quote leg** ([`trading.md`](trading.md#tradability)) and **portfolio positions** (below).

## `GET /stocks/prices?tickers=AAPL,TSLA,MSFT`

Live price snapshot for a targeted set. Comma-separated, up to **50 per call**. Server enforces `/^[A-Z0-9.-]{1,10}$/i` per item, uppercased server-side.

```jsonc
{
  "prices": [{
    "ticker": "NFLX",
    "tradfi": {   // null when the tradfi feed is unavailable
      "current_price_usd": "89.33", "change_24h_pct": "-0.35694",
      "market_cap_usd": "376150764000", "pe_ttm": "28.21",
      "volume_shares": "3412088",   // current-session shares traded; null when the feed omits it
      "volume_1d_usd": "304801821", // = volume_shares × current_price_usd; null exactly when volume_shares is null
      "as_of": 1779220801
    },
    "onchain": {   // per-protocol/venue; any side null when no listing or no price-feed data
      "ondo":      { "share_price_usd": "88.8680", "premium_vs_tradfi_pct": "-0.517", "volume_24h_usd": "4732002" },
      "xstocks":   { "share_price_usd": "88.3500", "premium_vs_tradfi_pct": "-1.097", "volume_24h_usd": "2649" },
      "robinhood": { "share_price_usd": "88.5000", "premium_vs_tradfi_pct": "-0.930", "volume_24h_usd": "1200" },
      "coinbase":  { "share_price_usd": "88.6100", "premium_vs_tradfi_pct": "-0.807", "volume_24h_usd": "9400" }
    }
  }]
}
```

`onchain.ondo`, `onchain.xstocks`, `onchain.robinhood` and `onchain.coinbase` are independent — pick any or all. `onchain.coinbase` carries the same supply gate as the listing: an unminted B20 cell prices `null`. `premium_vs_tradfi_pct` (negative = on-chain cheaper) is computed against `tradfi.current_price_usd`; when `tradfi` is `null`, each venue's premium is `null` too. Use for quote-time comparison, P&L marks, "current price" UX.

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
  "usdc": { "sol": "53.21", "eth": "0.00", "base": "12.50" },  // `base` is Base-native USDC on 8453 — NOT mainnet USDC
  "usdg": { "robinhood": "100.00" },
  "partial": false,                        // true → a balance read failed; `positions` is short (see below)
  "as_of": 1730000050, "is_cached": true
}
```

**`partial` — the row list itself may be short.** The nullable columns below cover a cell we *read* but couldn't *price*. `partial` covers the other case: a cell we couldn't read at all (an RPC blip on the sol/eth side, or on either single-cell venue — Robinhood Chain 4663 or Base 8453). Such a cell is **dropped** from `positions` rather than reported as a zero — an unreliable balance must never look like a real one — so on `partial: true` the list omits holdings the wallet may have and **any total you sum from `usd_value` undercounts**. It is otherwise a normal `200`: retry rather than treat it as authoritative, and don't overwrite a good cached view with a partial one. `partial` can be `true` with `is_cached: true` (the 30s snapshot captured the failure); a retry inside that window returns the same partial answer, so back off past it. Cash is **not** covered — `usdc`/`usdg` independently fall back to `"0"` on a failed read, which is why you should never treat their `"0"` as proof of an empty balance either.

`shares`, `usd_per_token`, `usd_per_share`, `usd_value` are each independently nullable — a position with a price-feed outage still surfaces with `tokens` populated so you can hold the row and re-render USD next poll. **Default any null to "unknown", never "0".** Tokens acquired outside Treasures reconcile in on the next read and emit a synthetic `external` row in `/trades`. **Exception — neither single-cell venue has an external-row reconciler:** you must **submit** each Robinhood-chain and Base trade via `/trade/submit` for it to appear in `/trades` at all — it enters as `broadcast` and reaches `completed` via the same status poll / backfill as `eth` (see [`trading.md`](trading.md#robinhood) and [`trading.md`](trading.md#base)). An **unsubmitted** trade on either venue never appears in `/trades` and carries no cost basis; whether its balance shows on `/portfolio` is governed by the per-venue position gate below.

**Positions can carry a tradability warning.** A position may include `tradability`, `warn_reason` and `thin_since` for its own `(ticker, protocol, chain)` cell — but only when the reason is **side-neutral** (`low_volume` or `no_price_feed`), since those two bear on an exit as much as on an entry. A cell warned for a buy-sided reason (`no_fill_window`, `settlement_failure`), or one whose reason was never recorded, emits **nothing** here. `min_trade_size_usd` is never published on a position: it is a USD **buy** floor and this is an exit surface. Advisory as everywhere else — it never marks a holding unsellable. Field meanings: [Tradability warnings](#tradability-warnings).

The three `source=internal` columns are derived from **completed internal trades only** (see [Internal-only P&L](#internal-only-pl) below). Caveat: because off-platform transfers are ignored by the basis, `shares_internal_only` can exceed the on-chain balance after you move tokens out — treat it as "shares bought via Treasures and not yet sold via Treasures", not a custody figure.

**Single-cell venue rows (Robinhood Chain 4663 · Base 8453).** Both venues report on `/portfolio`, and they behave identically — a wallet holding Robinhood Stock Tokens or Coinbase B20 tokens surfaces extra positions with a **bare** `token_ticker` (no `on`/`x` suffix): `chain: "robinhood"`, `protocol: "robinhood"` and `chain: "base"`, `protocol: "coinbase"` respectively. Both read **live** from the caller's own EOA on that chain (keyed off `eth_wallet`) — the 30s reconciler cache and the `is_cached` flag cover only the sol/eth cells, so these rows are never stale (and cost an RPC read per request). An RPC blip on either chain omits that venue's rows rather than failing the response — flagged `partial: true` (above), so an omitted row stays distinguishable from a wallet that holds none.

Each venue's **idle cash** is read from the same EOA and reported alongside:

| Venue | Cash field | Asset |
|---|---|---|
| Robinhood Chain (4663) | `usdg.robinhood` | USDG — the 4663 base currency |
| Base (8453) | `usdc.base` | **Base-native USDC** (`0x8335…2913`) |

> ⚠️ **`usdc.base` is a different contract from mainnet USDC.** `usdc` carries `sol`, `eth` **and `base`**; treat the three as separate balances on separate chains, never as one pooled figure.

A wallet holding only cash (no stock tokens yet) still shows it. Cash serves from a 30s snapshot that a settled trade invalidates, so a post-trade read reflects the new balance rather than waiting out the TTL; a failed read is never cached. `"0"` when there is no `eth_wallet`, the venue lists no cells, or the chain read fails — a failed cash read is **not** flagged `partial`, because `"0"` is indistinguishable from a real zero.

> ⚠️ **Positions on these two venues are gated on your trade history — cash is not.** We read a venue's positions only for a wallet we already hold a ledger row for **on that venue** (a trade made through this API and reported via `/trade/submit`). Neither venue writes to the reconciler ledger, so the read costs a live round-trip every request and is skipped for wallets that have never touched the venue. The gate is per **(wallet, venue)**: trading on 4663 does not open the Base read, and vice versa. Consequence — tokens you moved in by **direct transfer**, or a trade you **never submitted**, are absent from `positions`, and because nothing was read the response is **not** flagged `partial`. Submit your trades on both venues to keep positions complete, or value those holdings from your own RPC.

Two caveats specific to the position cells:

- **Cost basis is on-chain-observed.** `tokens`/`shares` are exact (live `balanceOf`), and the `source=internal` P&L columns (`avg_entry_price_per_share`, `unrealized_pnl`, and `realized_pnl` on `/trades`) are realized from the **mined fill** — the same fidelity as `eth`, not a quote-time estimate. Both venues' trades are recorded, so their basis replays like the sol/eth cells.
- **Discoverable via `/stocks/tickers`.** Each venue's listing block (`address`, `share_multiplier`, bare `token_ticker`) and its entry in `available_chains` are surfaced by [`/stocks/tickers`](#get-stockstickers); `/stocks/prices` carries `onchain.robinhood` and `onchain.coinbase` price cells — so these stocks are discovered and valued the same way as sol/eth. (Funding the venue is still your own: USDG onto 4663, USDC onto 8453 — see [`trading.md`](trading.md#robinhood) and [`trading.md`](trading.md#base).) Note: listing does **not** imply tradeable — on Robinhood only the liquid marquee names fill, and on Base only minted-and-priceable tickers do (others → `no_routes`).

> **A held position is never hidden by the Base supply gate.** The Base supply gate can refuse to *open* a position, never to show or close one — so a B20 holding stays visible on `/portfolio` and sellable even if its listing stops quoting.

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

Base trades behave identically — `chain: "base"`, `protocol: "coinbase"`, bare `token_ticker`, amounts in Base-native USDC — and carry the same submit-or-it-never-exists rule, for the same reason (no external-row reconciler on a single-cell venue).

## `GET /settlements?limit=50&cursor=...` — your own settled trades, filterable

**Requires an integrator API key** (`X-API-Key`), and returns *only* trades submitted with that key
presented. This is the one endpoint a read-only `trk_` reporting key may call; a `tik_` general key
works too and returns the same rows. Unlike `/trades` (which is wallet-scoped and public), this is
**organisation-scoped** — it answers "what did my integration do", not "what happened to this wallet".

Three things that will otherwise surprise you:

1. **Send `X-API-Key` on `/trade/submit`.** A submit without it is attributed to nobody and will
   *never* appear here — there is no way to recover it afterwards.
2. **Settled fills only.** A trade shows up once it reaches `completed`. On every EVM venue
   (`eth`, `robinhood`, `base`) that is when the order fills, which can be minutes after you submitted. Until then
   the recorded amounts are quote-time estimates, so they're withheld rather than reported as fact —
   poll `/quote/{quote_id}/status` for in-flight legs.
3. **`has_more` is authoritative, not `data.length`.** A short page does *not* mean the end.
   Every amount, price and fee is **truncated, never rounded up**, so a figure here is always at or
   below the true value — safe to reconcile or re-spend against without a buffer.
4. **Re-scan a trailing ~2h window if your totals must be exact.** A few Solana trades are settled
   by a recovery job that records the on-chain block time, typically minutes earlier than when the
   row appeared — so they can land *behind* a cursor you already passed. `trade_id` is stable, so
   re-scanned rows deduplicate cleanly. ~2h covers normal operation but is not a guaranteed bound:
   a delayed recovery job backdates by as long as it was delayed, so back exact balances with a
   periodic wider re-scan.

```jsonc
{
  "data": [{
    "trade_id": "trd_01J8...",
    "status": "DONE",
    "side": "buy",
    "ticker": "NVDA",
    "protocol": "ondo",
    "chain": "sol",
    "from_address": "7xKX...",           // EVM addresses come back EIP-55 checksummed
    "to_address": "7xKX...",             // a swap returns output to the same wallet
    "recipient": "7xKX...",              // alias of to_address
    "timestamp": 1785340800,             // settlement time, unix seconds
    "tx_hash": "5Uf...",
    "token_out_address": "So1...",       // mirrors receiving.token.address
    "amount": "1500000000",              // raw base units, mirrors receiving.amount
    "amount_usd": "250.420000",          // string, like every money field; USDG on "robinhood", Base-native USDC on "base"
    "shares": "1.500000000",
    "sending":   { "tx_hash": "5Uf...", "tx_link": "https://solscan.io/tx/5Uf...",
                   "chain": "sol", "timestamp": 1785340800,
                   "amount": "250420000",
                   "transfer_amount": "250420000",  // raw movement on this leg; see note below
                   "tokens": "250.420000",
                   "token": { "address": "EPjF...", "symbol": "USDC", "name": "USDC",
                              "price_usd": "1", "price_usd_per_share": null } },
    "receiving": { "tx_hash": "5Uf...", "tx_link": "https://solscan.io/tx/5Uf...",
                   "chain": "sol", "timestamp": 1785340800,
                   "amount": "1500000000",
                   "transfer_amount": "1500000000", // differs from amount only on xStocks/Ethereum
                   "tokens": "1.500000000",
                   "token": { "address": "So1...", "symbol": "NVDAon",
                              "name": "NVIDIA Corporation", "price_usd": "166.946666666666",
                              "price_usd_per_share": "166.946666666666" } },
    // dex_fee is the venue's take, net of ours. Absent (not 0) on trades settled before it was
    // recorded; a recorded 0 is emitted and means the venue charged no fee.
    "fee_costs": [{ "name": "treasures_fee", "percentage": "0.0025",
                    "amount_usd": "0.626050", "included": true },
                  { "name": "dex_fee", "percentage": "0.0010",
                    "amount_usd": "0.250420", "included": true }]
  }],
  "next_cursor": "MjAyNi0wNy0zMFQx...",  // pass back as ?cursor= ; null on the last page
  "has_more": true
}
```

Treasures trades are single-chain swaps, so `sending` and `receiving` share one `tx_hash`, chain and
timestamp — only the token and amount differ. Use `tokens` rather than assuming a decimal scale;
token precision is not part of the contract. Loop with `while (has_more)`, passing `next_cursor`.

`price_usd` is per *token*, matching `amount`/`tokens` on the same leg; `price_usd_per_share` is the
same fill priced per equity share. Prefer the per-share figure when comparing against a market or
tradfi reference — on the xStocks/Ethereum venue `price_usd` is a price per conserved internal unit
and matches no market quote. It is `null` on the stablecoin leg.

Each leg also carries `transfer_amount`: the raw figure the on-chain transfer moved, i.e. what a
block explorer renders. It equals `amount` everywhere except xStocks/Ethereum, whose token rebases
and emits *two* events — a standard `Transfer` (this field) and a shares-transfer event carrying
`amount`, the conserved internal unit. Both are on-chain; only the first is what an explorer shows,
so diffing `amount` against Etherscan reports a phantom mismatch of exactly the share multiplier.
Reconcile explorer movement on `transfer_amount`, equity on `shares`, internal accounting on
`amount`. A `null` means the trade settled before the field was recorded — unknown, not zero. Note
on the stablecoin leg, whether `fee_costs` is already netted out is venue-dependent — on a buy it is
the full spend; on an eth/Robinhood sell the venue skims from that leg so it already equals the wallet
credit; on a Solana sell the referral fee is a separate transfer so it sits above the credit. Treat it
as the raw movement on the leg and do not blanket-subtract `fee_costs` from it.

`token.address` is `null` in the rare case the catalog entry is unavailable; the trade is still
returned, because omitting a settled fill would understate your volume.

One user-facing sell can fan out across several venues, producing several entries with distinct
`trade_id`s and hashes — sum them rather than expecting one row per instruction. On a `400` with
`cursor: malformed`, restart from the first page instead of retrying the cursor.

### Filtering

Every filter is optional and they combine with AND. `chain`, `protocol` and `ticker` take a
comma-separated list, which is an OR *within* that one parameter:

```
GET /settlements?chain=sol,eth&ticker=AAPL,MSFT&side=buy&settled_from=1785412800&settled_to=1785499200
```

reads as "AAPL or MSFT, bought on Solana or Ethereum, settled in that 24-hour window".

`chain` accepts `sol` · `eth` · `robinhood` · `base`; `protocol` accepts `ondo` · `xstocks` ·
`robinhood` · `coinbase`. The two axes are ANDed, so an impossible pair (`?chain=base&protocol=ondo`)
is a valid request that simply matches nothing — it is not a `400`.

- **Repeating a parameter is not additive.** `?chain=sol&chain=eth` is read as `sol` alone — always
  use the comma form. Repeats within one list are fine (`?chain=sol,sol` is just `sol`).
- **Casing follows the response.** `ticker` is reported uppercase and is the one filter accepted in
  any case; it matches the response's `ticker` field, not the venue-suffixed `token.symbol`.
  `chain`, `protocol` and `side` are reported lowercase and matched exactly, so `?side=BUY` is a
  `400` — don't uppercase your whole query builder. EVM `token_out_address` values are
  case-insensitive; Solana addresses are base58 and case-sensitive.
- **`settled_from` / `settled_to` are Unix seconds and both inclusive.** They compare against the
  same `timestamp` each entry reports, so an entry whose `timestamp` equals `settled_to` is always
  included. An inverted range is a `400`.
- **`token_out_address` is side-aware**, because the field it filters is. Pass a stock token's
  address to get *buys* of that token; pass a base currency's address to get *sells* on that chain.
  **A base currency's address selects one chain, not "all sells"** — each chain has its own USDC
  contract (Solana `EPjF…Dt1v`, Ethereum `0xA0b8…eB48`, Base `0x8335…2913`) and Robinhood Chain uses
  USDG, so filtering on mainnet USDC returns Ethereum sells only. An address we don't carry returns
  an empty page, not an error.
- **A cursor belongs to the filters that produced it.** Keep the filter parameters identical for
  every page of a loop. Changing any of them returns `400` with `cursor: filters_changed` — start
  that new query from the first page. Changing `limit` mid-loop is fine.

```jsonc
// paging a filtered query
let cursor = null;
do {
  const qs = new URLSearchParams({ chain: "sol", side: "buy", limit: "100" });
  if (cursor) qs.set("cursor", cursor);          // filters stay identical every page
  const page = await get(`/settlements?${qs}`);
  cursor = page.next_cursor;
} while (cursor);                                 // or drive it off has_more
```

## Internal-only P&L

Passing `source=internal` to `/portfolio` or `/trades` computes cost basis from a replay of **completed internal trades only** — external (reconciler drift) and in-flight/failed rows are excluded. `realized_pnl` matches **[Hyperliquid's `closedPnl`](https://hyperliquid.gitbook.io/hyperliquid-docs/trading/entry-price-and-pnl)**:

- **Entry price** is the weighted average of your internal buys; a buy re-weights it, a sell leaves it unchanged.
- **Buys** carry `realized_pnl: null` (P&L is realized only when you close).
- **Sells** realize `(execution_price − avg_entry) × shares_closed`, **gross of fees** (fees are not subtracted, matching Hyperliquid's field).
- Selling more than you bought via Treasures (possible after depositing tokens from elsewhere) realizes only the covered portion; the excess realizes nothing.

To total realized P&L over a period, sum `realized_pnl` across the sell rows in the window (it's per-trade, not cumulative).
