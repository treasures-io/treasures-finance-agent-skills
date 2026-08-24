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
Omit `chain`/`protocol` → auto-route (single best cell). →
`{ chain, protocol, side, asset, max_amount_in, min_amount_out, route_type:"dex_aggregator" }` —
atomic strings; `chain`/`protocol` are the **resolved** cell. Errors: 400, 403 cap, 422
`quote_unavailable`/`asset_not_whitelisted`, 503 `routing_unavailable`.

A 422 `quote_unavailable` may carry an optional `reason` naming the finer cause. Two of them are
**permanent** for the asset and must not be retried: `reference_unavailable` (no market price exists to
price-check the venue against, so it isn't tradeable here) and `price_off_reference` (the on-chain price
sits too far from the underlying — a cap that is not caller-tunable, so raising `slippage_bps` cannot
clear it). Anything else — including a `quote_unavailable` with no `reason` — is a liquidity condition
worth retrying later.

### `POST API/wallets/:id/trades` — auth `key:trade` (or `owner`)
Headers: `Idempotency-Key` (required → 400 `missing_idempotency_key`; base64url/UUID charset `[A-Za-z0-9_-]`, ≤ 128 chars → 400 `invalid_idempotency_key`), `Content-Type: application/json`.
Body (`.strict`): `{ chain?, protocol?, side, asset, size:{notional_usdc}|{shares}, slippage_bps }`.
→ **202** job (see job shape). Route-first: no-route/cap/unwhitelisted are **sync 4xx** (no job).
Errors: 400, 401, 403 `key_cap_exceeded`/`policy_denied`, 404, 422, 503. The execute-time re-quote runs
the same price-check as `GET /quotes`, so a 422 here carries the same optional `reason` — treat
`reference_unavailable` and `price_off_reference` as permanent for the asset.

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
