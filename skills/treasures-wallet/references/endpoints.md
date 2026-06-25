# Endpoint reference

All paths relative to `host` (`https://api.treasures.io`).
**Two base paths:** `API = {host}/api`, `READS = {host}/public/v1`.
Auth column: `key:quote`/`key:trade` = `X-API-Key` with that scope; `owner` = Privy access token
(`Authorization: Bearer`) + identity token (`Privy-Id-Token`); `device` = Bearer `device_secret`;
`none` = public, IP-rate-limited.

## Agent plane — `API/wallets/...`

### `GET API/wallets/:id` — wallet metadata · auth `none`
→ `{ wallet_id, status, addresses:{evm,solana}, owner_quorum_id, signers:[{role,app_enabled}],
policies:[{chain,scope,owner_quorum_id}], created_at }`. Use once to resolve + cache addresses. 404
`wallet_not_found`.

### `GET API/wallets/:id/balances` — auth `none`
→ `{ native:{sol,eth}, stablecoins:[{chain,asset,amount}],
positions:[{issuer,asset,chain,token_address,raw_token,shares,usd_per_token,usd_per_share,notional_usd}],
needs_funding, as_of }`. `chain` ∈ `sol|eth`, `issuer` ∈ `ondo|xstocks`. On-chain truth.

### `GET API/wallets/:id/delegation` — auth `none`
→ `{ app_enabled, signers:[{role,app_enabled}] }`. Is delegated trading enabled.

### `GET API/wallets/:id/quotes` — auth `key:quote` (or `owner`)
Query (`.strict`): `chain?` (`solana|ethereum`), `protocol?` (`ondo|xstocks`), `side` (`buy|sell`),
`asset`, **exactly one of** `notional_usdc` (buy) / `shares` (sell), `slippage_bps` (int, ≤ 500).
Omit `chain`/`protocol` → auto-route (single best cell). →
`{ chain, protocol, side, asset, max_amount_in, min_amount_out, route_type:"dex_aggregator" }` —
atomic strings; `chain`/`protocol` are the **resolved** cell. Errors: 400, 403 cap, 422
`quote_unavailable`/`asset_not_whitelisted`, 503 `routing_unavailable`.

### `POST API/wallets/:id/trades` — auth `key:trade` (or `owner`)
Headers: `Idempotency-Key` (required → 400 `missing_idempotency_key`), `Content-Type: application/json`.
Body (`.strict`): `{ chain?, protocol?, side, asset, size:{notional_usdc}|{shares}, slippage_bps }`.
→ **202** job (see job shape). Route-first: no-route/cap/unwhitelisted are **sync 4xx** (no job).
Errors: 400, 401, 403 `key_cap_exceeded`/`policy_denied`, 404, 422, 503.

### `GET API/wallets/:id/trades/:job_id` — auth `key:trade` (or `owner`)
Same credential as `POST /trades` — the poller is the trade's creator. → job. State machine
`routing → simulating → signing → broadcast → {confirmed|failed|rejected}`. Errors: 401, 403
`insufficient_scope`, 404 `job_not_found` (also if the job belongs to another wallet).

**Job shape:** `{ job_id, state, chain, protocol, side, asset, route_type, reject_reason, tx_sig,
attempts, result, created_at, updated_at }`. At `confirmed`: `result = { amount_in, amount_out, tx_sig }`
(atomic). EVM: `tx_sig` at `broadcast` = Fusion orderHash; `result.tx_sig` at `confirmed` = on-chain
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
| `POST API/wallets/:id/withdrawals` | build UNSIGNED withdrawal tx (owner signs client-side; MFA gate for USDC > $1000) | unsigned tx |
| `POST API/wallets/:id/mfa/enroll` | sync MFA-enrolled flag from Privy | `{mfa_enrolled}` |

Source of truth: `src/api/routes/wallets.ts`, `src/api/routes/onboarding-sessions.ts`,
`src/api/routes/public/v1/{trades,portfolio}.ts`; mounts in `src/api/index.ts`.
