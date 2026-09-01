# Endpoint reference

All paths relative to `host` (`https://api.treasures.io`).
**Two base paths:** `API = {host}/api/v1`, `READS = {host}/public/v1`.
Auth column: `key:quote`/`key:trade` = `X-API-Key` with that scope; `owner` = Privy access token
(`Authorization: Bearer`) + identity token (`Privy-Id-Token`); `device` = Bearer `device_secret`;
`none` = public, IP-rate-limited.

## Agent plane — `API/wallets/...`

### `GET API/wallets/:id` — wallet metadata · auth `none`
→ `{ wallet_id, status, addresses:{evm,solana}, owner_quorum_id, signers:[{role,app_enabled}],
policies:[{chain,scope,owner_quorum_id}], created_at }`. Use once to resolve + cache addresses. 404
`wallet_not_found`.

### `GET API/wallets/:id/balances` — auth `none`
→ `{ native:{sol,eth,robinhood,base}, stablecoins:[{chain,asset,amount}],
positions:[{issuer,asset,chain,token_address,raw_token,shares,usd_per_token,usd_per_share,notional_usd}],
needs_funding, as_of }`. `chain` ∈ `sol|eth|robinhood|base`, `issuer` ∈ `ondo|xstocks|robinhood|coinbase`.
`native.robinhood` (4663 Orbit L2) and `native.base` (8453 OP-Stack L2) are native ETH held by the
same EOA as `native.eth` — raw amounts only, no USD (for gas valued in USD see the B2C portfolio
reads, D-NativeInPortfolio).
`needs_funding` is true only when **all four** are zero, so a 4663-only funded wallet is not
"needs funding" — the flag is coarse by design; per-chain readiness is derivable from `native`.

**Freshness:** positions/stablecoins and the `native` AMOUNTS are served from the reconciler's result
cache (30s fresh, up to 300s stale-while-revalidate), so this is on-chain truth as of `as_of`, not as of
the request. `needs_funding` is the exception and is never stale in the direction that matters: a
would-be `true` is re-confirmed with a live on-chain read before being returned, so a wallet you just
funded flips to `false` immediately rather than after the cache window. (`false` is served from cache —
staleness cannot un-gas a wallet.)

### `GET API/wallets/:id/delegation` — auth `none`
→ `{ app_enabled, signers:[{role,app_enabled}] }`. Is delegated trading enabled.

### `GET API/wallets/:id/quotes` — auth `key:quote` (or `owner`)
Query (`.strict`): `chain?` (`solana|ethereum`), `protocol?` (`ondo|xstocks`), `side` (`buy|sell`),
`asset`, **exactly one of** `notional_usdc` (buy) / `shares` (sell), `slippage_bps` (int, ≤ 500).
Omit `chain`/`protocol` → auto-route (single best cell). **Buy** →
`{ chain, protocol, side, asset, max_amount_in, min_amount_out, route_type:"dex_aggregator" }` —
atomic strings; `chain`/`protocol` are the **resolved** cell. **Sell** →
`{ side, asset, route_type:"dex_aggregator", legs:[{ chain, protocol, shares_consumed, max_amount_in,
min_amount_out }] }` — one entry per cell the sell draws from. Errors: 400, 403 cap, 422
`quote_unavailable`/`asset_not_whitelisted`, 503 `routing_unavailable`.

A 422 `quote_unavailable` may carry an optional `reason` naming the finer cause. `price_implausible`
means the quote sits so far from the market price that we do not trust our own arithmetic and will not
execute on it — no size and no `slippage_bps` clears it, though a re-quote may. `price_off_reference`
is a venue-side price refusal that is likewise not caller-tunable. `reference_unavailable` is a
**retained wire word with no current producer**: an asset with no market price to check against is now
priced and disclosed rather than refused (see below). Anything else — including a `quote_unavailable`
with no `reason` — is a liquidity condition worth retrying later.

<a id="warnings"></a>
#### `warnings[]` on the preview, and the advisory acknowledgement

Every preview carries a top-level `warnings[]` — **always present**, `[]` when there is nothing to
disclose — plus `warnings_ack_token` (32 hex chars) beside a non-empty set. Each entry has a `code`:
`no_reference` (no market price exists to check this venue's price against — permanent for assets our
market-data provider does not cover, transient during a feed outage), `off_reference` (a reference
exists and the price sits beyond its band in the direction that costs you), or the priced cell's own
`warn_reason` from the table below. Treat an unrecognised code as "warned, reason unknown".

A warned preview is still tradeable: this plane discloses and does not gate. `POST /trades` accepts an
**optional** `acknowledged_warnings` — echo the token to have your acknowledgement recorded beside the
trade; omit it and the trade proceeds unchanged.

**What the token is bound to.** The digest covers the **`(wallet, asset, side)`** this preview names
together with the **codes** in the `warnings[]` served (and each entry's `thin_since`) — not their
magnitudes, and not a quote id, because this plane mints none and the trade re-quotes from scratch. So
a token minted for one asset **does not verify for another**, and a buy token does not verify on a
sell, even on the same wallet. Mint one per asset and side, from the preview you actually acted on.

The binding is deliberately at code grain here, and only here: because the trade prices itself, an
`off_reference` entry's `deviation_bps` and prices are all but certain to differ from the preview's, so
a digest over those numbers would refuse every honest acknowledgement of the one warning class it
matters most for. What an echoed token establishes is therefore **which risks** you were shown for this
asset and side — not the numbers beside them, which may have moved by the time the trade prices.

The trade route re-derives the digest from its OWN quote: a token matching nothing that quote disclosed
is refused with `422 warnings_not_acknowledged` rather than recorded, which you clear by omitting the
field or by re-previewing and echoing the fresh token. What still moves the set between your preview
and your trade is a change in **which** codes apply — a price crossing into or out of the reference
band, or a tradability verdict appearing or disappearing (including the case where that verdict table
is briefly unavailable at trade time and the set comes back **without** its tradability entry). A
price that merely moved further out of band does not. If you would rather never be refused on this,
omit the field: what was disclosed is recorded against the trade either way — in full, magnitudes
included.

<a id="tradability"></a>
#### Tradability warnings on the preview

The preview may carry three optional advisory fields for the cell it priced — on a **buy** at the top
level (the resolved cell), on a **sell** inside each `legs[]` entry. They come from a background probe
that quotes a ladder of buy sizes against every listed cell and watches whether real orders settle:

| Field | Values | Meaning |
| --- | --- | --- |
| `tradability` | `"thin"` \| `"untradable"` | `thin` = the cell fills, but not at every size. `untradable` = a full measurement window passed with no fill at any size. |
| `warn_reason` | `"settlement_failure"` \| `"no_price_feed"` \| `"no_fill_window"` \| `"no_settlements"` \| `"low_volume"` | Why the warning was raised — act per the table below. |
| `thin_since` | ISO-8601 string | When the warning was first raised. Present exactly when `tradability` is. |

**Advisory, never a block.** A warned cell is still priced and a trade to it is still accepted — the
server drops no leg and refuses no preview over these, and there is no request param to exclude warned
cells. `untradable` means "nothing filled across a full measurement window", **not** "disabled". The
agent decides, by reason:

| `warn_reason` | What it means | What to do |
|---|---|---|
| `settlement_failure` | A real order on this cell recently expired without settling — quotes here can look fine and still never fill | Pin another `chain`+`protocol`; if this is the only cell, warn the user before submitting |
| `no_price_feed` | The cell holds no price for this pair at all | Do not trade this cell at any size; pick another cell or skip the asset |
| `no_fill_window` | No probe size has filled here for weeks | Treat as likely-to-fail; pin another cell |
| `no_settlements` | No value-bearing on-chain settlement observed for this cell in the 21-day census window | Treat as likely-to-fail; pin another cell |
| `low_volume` | Measured on-chain trading volume is below a floor | Expect poor pricing on entry AND exit; keep sizes small or pin another cell |

- **Absent ≠ null.** All three keys are omitted, never sent as `null`. An absent `tradability` means the
  cell is **unmeasured or unwarned** — never "healthy, value null". `warn_reason` can be absent beside a
  *present* `tradability`: that warning predates the reason field — read it as "reason not recorded",
  never "no reason".
- **`warn_reason` is an open vocabulary.** New values are added over time; treat an unlisted one as
  "warned, reason unknown" rather than an error.
- **Sell legs publish side-neutral reasons only** — `low_volume`, `no_price_feed` or `no_settlements`.
  A cell warned for `no_fill_window` or `settlement_failure` (both buy-sided), or for a reason never
  recorded, emits **nothing** on a sell leg. Nothing here ever marks a holding unsellable.
- **Not stable over a cell's lifetime.** A `low_volume` warning flips to `settlement_failure` once a real
  order dies there. Re-read it per preview; don't cache it.
- The same fields ride `READS/portfolio` positions (side-neutral only) and, cell-by-cell per ticker,
  `READS/stocks/tickers` — documented in the `treasures-b2b-api` skill.

### `POST API/wallets/:id/trades` — auth `key:trade` (or `owner`)
Headers: `Idempotency-Key` (required → 400 `missing_idempotency_key`; base64url/UUID charset `[A-Za-z0-9_-]`, ≤ 128 chars → 400 `invalid_idempotency_key`), `Content-Type: application/json`.
Body (`.strict`): `{ chain?, protocol?, side, asset, size:{notional_usdc}|{shares}, slippage_bps }`.
→ **202** job (see job shape). Route-first: no-route/cap/unwhitelisted are **sync 4xx** (no job).
Body also accepts an **optional** `acknowledged_warnings` (the preview's `warnings_ack_token`) — see
[`warnings[]` on the preview](#warnings). Errors: 400, 401, 403 `key_cap_exceeded`/`policy_denied`, 404,
422 (incl. `warnings_not_acknowledged` when a token you sent matches nothing this quote disclosed), 503.
The execute-time re-quote runs the same price-check as `GET /quotes`, so a 422 here carries the same
optional `reason`.

### `GET API/wallets/:id/trades/:job_id` — auth `key:trade` (or `owner`)
Same credential as `POST /trades` — the poller is the trade's creator. → job. State machine
`routing → simulating → signing → broadcast → {confirmed|failed|rejected}`. Errors: 401, 403
`insufficient_scope`, 404 `job_not_found` (also if the job belongs to another wallet).

**Job shape:** `{ job_id, state, chain, protocol, side, asset, route_type, reject_reason, tx_sig,
attempts, result, created_at, updated_at }`. At `confirmed`: `result = { amount_in, amount_out, tx_sig }`
(atomic). EVM: `tx_sig` at `broadcast` = the order hash; `result.tx_sig` at `confirmed` = on-chain
fill. `failed`/`rejected` ⇒ no funds moved.

## Reads plane — `READS/...` (auth `none`)

### `GET READS/trades?sol_wallet=&eth_wallet=&source=&limit=&offset=`
≥1 of `sol_wallet`/`eth_wallet` required. `source` = `all` (default) | `internal` — **toggles P&L
columns, does not filter rows**. `limit` (1–MAX, default), `offset`. 30 s cached. →
`{ trades:[{trade_id, source, side, ticker, token_ticker, chain, protocol, tx_hash, order_hash,
tokens, shares, usdc_amount, usd_per_share, avg_entry_price_per_share, realized_pnl, submitted_at,
status}], next_offset }`. `internal` = Treasures-executed; `external` = on-chain transfers (null side/usdc).

### `GET READS/portfolio?sol_wallet=&eth_wallet=&source=`
≥1 address. → positions + `usd_value`; on `source=internal` adds `shares_internal_only`,
`avg_entry_price_per_share`, `unrealized_pnl`.

## Onboarding plane — `API/onboarding-sessions/...`
See [`onboarding.md`](onboarding.md). Mint (`none`) · consent `GET /:requestId` (`none`) · complete
`POST /:requestId/complete` (`owner`) · poll `POST /poll` (`device`).

## Owner plane — `API/wallets/...` (auth `owner`; the agent's API key CANNOT call these)

| Method + path | Purpose | Returns |
|---|---|---|
| `POST API/wallets` | provision wallet (body `{}`) | 201/200 `{…walletSummary, email, needs_funding}` |
| `GET API/wallets/me` | own wallet by identity | 200 `{…, email}` / 404 |
| `POST API/wallets/:id/api-keys` | create key | 201 `{api_key, key_id, scopes, caps}` (plaintext once) |
| `GET API/wallets/:id/api-keys` | list keys | `{api_keys:[{key_id,scopes,caps,last_used_at,revoked_at}]}` |
| `DELETE API/wallets/:id/api-keys/:key_id` | revoke (idempotent) | `{revoked:true}` |
| `GET API/wallets/:id/delegation/grant-spec` | per-chain inputs for client-side Privy grant | `{specs:[{chain,address,signerId,policyIds}]}` |
| `POST API/wallets/:id/delegation/enable` | enable delegated trading (409 if not granted) | `{app_enabled,signers}` |
| `POST API/wallets/:id/delegation/disable` | disable | `{app_enabled,signers}` |
| `POST API/wallets/:id/withdrawals` | build UNSIGNED withdrawal tx (owner signs client-side); over the per-tx / rolling-24h cap → `422 withdraw_cap_exceeded` with `cap: per_tx\|daily` | unsigned tx |

Source of truth: `src/api/routes/api/v1/wallets.ts`, `src/api/routes/api/v1/onboarding-sessions.ts`,
`src/api/routes/public/v1/{trades,portfolio}.ts`; mounts in `src/api/index.ts`.
