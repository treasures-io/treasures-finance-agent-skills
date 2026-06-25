# API keys — scopes, caps, management

The API key (`twk_<key_id>_<secret>`) is the agent's **only** trade-plane credential. Format:
`X-API-Key: twk_…`. The plaintext is returned **once** (at onboarding `poll` or owner `create`) and
never persisted/logged — only `sha256(secret)` is stored. A lost key cannot be recovered; mint a new one.

## Scopes
`['trade', 'quote']` (default: both). Enforced at auth:
- `quote` → `GET /wallets/:id/quotes`. Missing → **403** `insufficient_scope`.
- `trade` → `POST /wallets/:id/trades` **and** `GET /wallets/:id/trades/:job_id` (the status poll is
  gated like its create sibling — the poller is the trade's creator). Missing → **403** `insufficient_scope`.

## Caps (per-key spend limits) — all optional
```json
{ "max_trade_notional_usd": "1000.50",   // decimal string; per-trade USD ceiling
  "daily_notional_usd":     "10000.00",  // decimal string; per-key per-UTC-day ceiling
  "asset_allowlist":        ["NVDA","TSLA"], // exact-ticker match; only these tradable
  "max_slippage_bps":       50 }          // positive int; caps the slippage_bps a trade may request
```
- **Intent-pure caps** (`asset_allowlist`, `max_slippage_bps`) are checked on **both** quote and trade
  (no I/O). **Notional caps** (`max_trade_notional_usd`, `daily_notional_usd`) are checked on **trade
  only** (a quote is a read-only preview); the daily cap is enforced atomically under a per-key lock.
- **Breach → 403** `{ error:"key_cap_exceeded", cap, reason }` where `cap` names the field and `reason`
  is `exceeded` or `price_unavailable` (a notional-capped trade that couldn't be priced → **fail
  closed**). **Don't retry a cap breach** — surface it.
- A buy prices off `notional_usdc`; a sell prices off `shares × spot`.

## Auth errors (on quote/trade)
| code | HTTP | when |
|---|---|---|
| `invalid_api_key` | 401 | missing/malformed/unknown key, wrong wallet, or bad secret (uniform — no existence leak) |
| `key_revoked` | 403 | key exists + secret valid, but revoked |
| `insufficient_scope` | 403 | valid key lacks the required scope |

## Management — OWNER-only (the agent's key cannot do these)
Create/list/revoke require an **owner Privy session** (`Authorization: Bearer <access>` +
`Privy-Id-Token: <identity>`), verified to own the wallet. An agent holding only a `twk_…` key **cannot**
mint or manage keys — its key is provisioned at onboarding (with the scopes/caps requested in the mint
body). To get a *new* scoped key headlessly, run another onboarding session.

| Method + path | Body | Returns |
|---|---|---|
| `POST API/wallets/:id/api-keys` | `{ scopes?, caps? }` (`{}` → default+uncapped) | 201 `{ api_key, key_id, scopes, caps }` — **plaintext once** |
| `GET API/wallets/:id/api-keys` | — | `{ api_keys:[{ key_id, scopes, caps, last_used_at, revoked_at }] }` (no secret) |
| `DELETE API/wallets/:id/api-keys/:key_id` | — | `{ revoked:true }` — idempotent tombstone; unknown/other-wallet keys also return 200 (no existence leak) |

Owner errors: 401 `unauthorized`, 403 `not_wallet_owner`, 404 `wallet_not_found`, 400 invalid scopes/caps.

Source: `src/services/wallet/api-key-types.ts` (scopes/caps), `key-caps.ts` (enforcement),
`api-keys.ts` (format/verify), `src/api/routes/wallets.ts` (routes).
