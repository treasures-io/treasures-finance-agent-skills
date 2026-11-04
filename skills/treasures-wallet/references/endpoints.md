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
→ `{ native:{sol,eth,robinhood,base,arbitrum,bnb}, stablecoins:[{chain,asset,amount}],
positions:[{issuer,asset,chain,token_address,raw_token,shares,usd_per_token,usd_per_share,notional_usd}],
needs_funding, as_of }`. `chain` ∈ `sol|eth|robinhood|base|arbitrum|bnb`, `issuer` ∈
`ondo|xstocks|backpack|robinhood|coinbase|reality|bstocks`. Only `sol`/`eth`/`bnb` positions with issuer
`ondo`/`xstocks`/`backpack`/`bstocks` are tradeable on this plane (`backpack` on `sol` only, `bstocks`
on `bnb` only); the rest are reported for completeness. ⚠ `bnb` carries up to three issuers (`ondo`,
`xstocks`, `bstocks`), so a 56 position's cell is `(chain, issuer)`: grouping bnb positions by `chain`
alone merges three of them. A `bnb` `stablecoins` entry is 18-decimal **Binance-Peg USDC**: still
`asset: "USDC"` on the wire, a different token from USDC on every other chain, never to be summed with
them. A `backpack` position carries the bare ticker as its token symbol on the
`READS` routes (no `on`/`x` suffix), so branch on `issuer`, never on the symbol's shape.
Both unions, and the `native` keys, are open: each new chain or issuer adds a value, so never fail a
parse on one you don't recognise.
`native.robinhood` (4663 Orbit L2), `native.base` (8453 OP-Stack L2) and `native.arbitrum` (42161
Arbitrum One) are native ETH held by the same EOA as `native.eth`; **`native.bnb` (BNB Smart Chain 56)
is BNB, not ETH**, the only chain here whose gas token is not ether. Raw amounts only, no USD (for gas
valued in USD see the B2C portfolio reads, D-NativeInPortfolio). 56 rows are reported whether or not the
bnb venue is open for new trades.
`needs_funding` is true only when **every** `native` entry is zero, so a 4663-only funded wallet is not
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
Query (`.strict`): `chain?` (`solana|ethereum`), `protocol?` (`ondo|xstocks|backpack|bstocks`; `backpack` is
Solana-only, so pair it with `solana` or omit `chain`; with `ethereum` it is `400 invalid_request`;
`bstocks` lists only on BNB Smart Chain, which has no `chain` value here, so omit `chain` with it),
`side` (`buy|sell`),
`asset`, **exactly one of** `notional_usdc` (buy) / `shares` (sell), `slippage_bps` (int, ≤ 500).
Omit `chain`/`protocol` → auto-route (single best cell). **Buy** →
`{ chain, protocol, side, asset, max_amount_in, min_amount_out, route_type:"dex_aggregator" }` —
atomic strings; `chain`/`protocol` are the **resolved** cell. **Sell** →
`{ side, asset, route_type:"dex_aggregator", legs:[{ chain, protocol, shares_consumed, max_amount_in,
min_amount_out }] }` — one entry per cell the sell draws from. Errors: 400, 403 cap, 422
`quote_unavailable`/`asset_not_whitelisted`, 503 `routing_unavailable`.

⚠ **BNB Smart Chain (56) is reachable only by auto-routing, and the echoed `chain` is lossy.** 56
contributes up to three candidate cells (`ondo`, `xstocks`, `bstocks`) while the venue is open for you,
but it has no `chain` pin here, and every EVM chain echoes `chain: "ethereum"` on this response: a
56-routed quote returns `{ chain: "ethereum", protocol: "bstocks" | "ondo" | "xstocks" }`. **Read
`protocol`** (and, for `ondo`/`xstocks`, the job view or `/balances`, which report the real chain code
`bnb`). A 56 leg settles in 18-decimal Binance-Peg USDC, so its atomic `max_amount_in` /
`min_amount_out` are 18-dp on the stablecoin side.

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
→ **202**. A **buy** returns one flat job (see job shape), `job_id` at the top level. A **sell**
returns a group — `{ order_status, legs:[ job… ] }` — because the server splits it across every venue
the wallet holds; `job_id` is on each `legs[]` entry, and you poll them all. `order_status` is
`pending` while any leg is non-terminal, else `confirmed` / `partially_filled` / `failed`; it is a
submit-time snapshot, so re-derive it from the polled legs. A **pinned** sell (`chain`+`protocol`)
returns the same group shape with one entry, and 422s `quote_unavailable` if the pinned cell holds
less than the requested size. One `Idempotency-Key` covers the whole order, legs included.
Route-first: no-route/cap/unwhitelisted are **sync 4xx** (no job).
Body also accepts an **optional** `acknowledged_warnings` (the preview's `warnings_ack_token`) — see
[`warnings[]` on the preview](#warnings). Errors: 400, 401, 403 `key_cap_exceeded`/`policy_denied`, 404,
422 (incl. `warnings_not_acknowledged` when a token you sent matches nothing this quote disclosed), 503.
The execute-time re-quote runs the same price-check as `GET /quotes`, so a 422 here carries the same
optional `reason`.

### `GET API/wallets/:id/trades` — auth `key:quote` (or `owner`)
The wallet's jobs, newest first, each in the **job shape** below. Query: `limit` (1–100, default 20),
`cursor` (echo `next_cursor` verbatim), optional `state`. → `{ data: [job…], next_cursor, has_more }`;
a page can be short while `next_cursor` is set — keep paging until it is null. Rate limit: its own
120/min per-IP bucket (separate from the by-id poll's 30/min). Errors: 400 (unknown
param / malformed cursor), 401, 403 `insufficient_scope`.

### `GET API/wallets/:id/trades/:job_id` — auth `key:quote` (or `owner`)
Same read scope as the list (a trade-only key is refused). → job. State machine
`routing → simulating → signing → broadcast → {confirmed|failed|rejected}`. Errors: 401, 403
`insufficient_scope`, 404 `job_not_found` (also if the job belongs to another wallet).

**Job shape:** `{ job_id, state, chain, protocol, side, asset, route_type, reject_reason, tx_sig,
attempts, result, created_at, updated_at }`. At `confirmed`: `result = { amount_in, amount_out, tx_sig }`
(atomic). EVM: `tx_sig` at `broadcast` = the order hash; `result.tx_sig` at `confirmed` = on-chain
fill. `failed`/`rejected` ⇒ no funds moved.

⚠ **The job's `chain` is the INTERNAL code** (`sol|eth|bnb`), not the `solana|ethereum` the request and
quote response speak, and `protocol` ∈ `ondo|xstocks|backpack|bstocks`. It is the one trade surface on
this plane that tells you a trade settled on BNB Smart Chain by its `chain` field.

**Dashboard reads (no dedicated endpoint):** `GET /:id/delegation` (signer granted: `app_enabled`,
`signers[]`; public) · `GET /:id/api-keys` (owner) · `GET /:id/policy-migration/spec` (`migrated`,
`specs`; owner).

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
`POST /:requestId/complete` (`owner`) · poll `POST /poll` (`device`). A mint body carrying a `wallet`
key → 410 `sma_onboarding_retired` (SMA wallets are not minted here).

## Owner plane — `API/wallets/...` (auth `owner`; the agent's API key CANNOT call these)

| Method + path | Purpose | Returns |
|---|---|---|
| `POST API/wallets` | provision wallet (body `{}`); **main only** | 201/200 `{…walletSummary, email, needs_funding}` |
| `GET API/wallets/me` | own wallet(s) by identity | 200 `{…main fields (null if SMA-only), email, sma_wallets:[{id,strategy_ref,status,sol_address,evm_address,policies_migrated}]}` / 404 if neither exists |
| `POST API/wallets/:id/api-keys` | create key | 201 `{api_key, key_id, scopes, caps}` (plaintext once) |
| `GET API/wallets/:id/api-keys` | list keys | `{api_keys:[{key_id,scopes,caps,last_used_at,revoked_at}]}` |
| `DELETE API/wallets/:id/api-keys/:key_id` | revoke (idempotent) | `{revoked:true}` |
| `GET API/wallets/:id/delegation/grant-spec` | per-chain inputs for client-side Privy grant | `{specs:[{chain,address,signerId,policyIds}]}` |
| `POST API/wallets/:id/delegation/enable` | enable delegated trading (409 if not granted) | `{app_enabled,signers}` |
| `POST API/wallets/:id/delegation/disable` | disable | `{app_enabled,signers}` |
| `POST API/wallets/:id/withdrawals` | build UNSIGNED withdrawal tx (owner signs client-side); gated by withdraw pins if any are set (below) | unsigned tx |
| `PUT API/wallets/:id/withdraw-pins` | main or SMA wallet: set/replace/clear (`{}`) destination pins + per-tx caps; every write that changes the pins (a `{}` clear included; an identical re-PUT changes nothing) must be signed via the existing policy-migration spec/apply flow before it takes effect | 200 `{pins, policies_migrated, next?}` (`next: "policy-migration"` while a signature is owed) |
| `GET API/wallets/:id/withdraw-pins` | read the stored pins and whether the bases have signed them | 200 `{pins, policies_migrated}` |

**Withdraw pins (W3a).** An owner may pin ≤ 4 EVM addresses and ≤ 4 Solana **owner** addresses (a wallet, or a PDA such as a Squads vault)
per wallet, plus an optional `per_tx_cap_usdc` and per-chain `per_tx_cap_native`. Until the pin write is
owner-signed, withdrawals answer `409 policies_stale`. Pre-check error codes an agent or client may see on
`POST .../withdrawals` (and its `/apply`, and the owner plane's copy-fund equivalent):

| Code | Meaning |
|---|---|
| `destination_not_pinned` (422) | the recipient is not one of the signed pins for that chain family |
| `withdraw_cap_exceeded` (422, `{cap:'per_tx'}`) | the amount exceeds the signed per-tx cap |
| `pins_required` (400, PUT only) | a cap was set with no destination pinned for that family |
| `policies_stale` (409) | the pin write exists but has not been owner-signed yet |
| `withdraw_pins_unavailable` (503) | the pins store could not be read — fails closed, never treated as unpinned |
| `withdraw_tx_invariant` (500, outside the refusal vocabulary) | the backend's own built transaction broke a pin the intent already passed — a backend fault, not a caller error |

⚠ **Solana pins are the destination's associated token account per stable mint, never the owner
address** (`TransferChecked.destination` is compared literally; an owner address in the pin list does
NOT match that owner's ATA). ⚠ **The per-tx cap is enforced per instruction in Privy's policy, not per
transaction** — the backend separately bounds the sum of transfer instructions in one transaction before
building it, but a client that signs its own bytes is bounded only by the per-instruction TEE rule.

Source of truth: `src/api/routes/api/v1/wallets.ts`, `src/api/routes/api/v1/onboarding-sessions.ts`,
`src/api/routes/public/v1/{trades,portfolio}.ts`; mounts in `src/api/index.ts`.
