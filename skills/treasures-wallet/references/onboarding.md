# Onboarding (RFC 8628 device grant)

A **two-actor** flow. The **agent** (headless) mints a session and polls for a key; a **human owner**
logs in via Privy and grants the swap delegation **in a browser**. The agent cannot do the human part —
provisioning and the delegation grant require a Privy login + in-browser signing.

```
agent: POST /onboarding-sessions ──► {request_id, device_secret, url, interval}
agent: send `url` to the human ─────► (human opens it, logs in via Privy)
human (hosted page): provision wallet + GET grant-spec → Privy useSigners().addSigners (both chains)
human (hosted page): POST /onboarding-sessions/:requestId/complete  → status "approved"
agent: POST /onboarding-sessions/poll (loop, Bearer device_secret) → {status:"approved", api_key, ...}
```

## Agent operating rules (do exactly this)

1. **Mint** the session (`POST /onboarding-sessions`). Keep `device_secret` private (it's your poll Bearer).
2. **Surface `url` to a human** and ask them to open it, log in, and approve. **You cannot do this part** —
   if no human is reachable, STOP here and hand off the `url`; do not loop pointlessly.
3. **Poll** `POST /onboarding-sessions/poll` every `interval` seconds (Bearer `device_secret`, body
   `{request_id}`). `{status:"pending"}` → keep polling. `{status:"approved", …, api_key}` → **capture +
   store the `api_key` immediately** (delivered once), plus `wallet_id` + `addresses`.
4. **Give up at `expires_at`** (~15 min). On `401 invalid_session` after approval you waited past the
   ~5 min claim window — **re-mint** and start over.
5. **Never call `/complete` yourself** — it needs the human's Privy access+identity token (their browser),
   which the agent does not have. A `/complete` attempt by the agent will 401.

## Endpoints (all under `API = {host}/api/v1`)

### `POST API/onboarding-sessions` — mint · auth `none` (agent)
Body (`.strict`, optional): `{ "api_key": { "scopes"?: ["trade"|"quote"], "caps"?: {…} } }` — or `{}`
for default scopes `["trade","quote"]`, uncapped. Caps shape: see [`api-keys.md`](api-keys.md).
→ **201** `{ request_id, device_secret, url, expires_at, interval }`.
- `device_secret` is shown **once** — the agent keeps it private and uses it as the poll Bearer. Never
  rendered in a browser; only its hash is stored.
- `url` is the hosted onboarding page (carries `request_id`); send it to the human.
- `interval` = suggested poll seconds (e.g. 3). Session TTL ~15 min; after `complete`, a ~5 min claim window.

### `GET API/onboarding-sessions/:requestId` — consent · auth `none` (the page)
→ `{ scopes, caps, status:"pending"|"approved", expires_at }`. **Never** returns the device_secret or
key. Used by the hosted page to show the human what's being requested.

### `POST API/onboarding-sessions/:requestId/complete` — auth `owner` (human, via page)
The hosted page calls this after the human logged in (Privy access + identity token) and the swap signer
was granted on **both chains** (grant-spec → `useSigners().addSigners`). Provisions (idempotent on the
verified Privy DID) and gates on the both-chains grant. → `{ wallet, status:"approved" }`. Errors: 403
`owner_mismatch` (an approved session re-completed by a different identity), 409 `delegation_not_granted`
(grant not on both chains yet), 503 `grant_check_unavailable` (Privy read failed — retryable), 502
`privy_provisioning_failed`.

### `POST API/onboarding-sessions/poll` — auth `device` (agent)
Header `Authorization: Bearer <device_secret>`. Body (`.strict`): `{ request_id }`.
- Before approval → `{ status:"pending" }` (keep polling every `interval`).
- First poll after approval → `{ status:"approved", wallet:{…walletSummary}, owner_email, api_key }`.
  **`api_key` (`twk_…`) is delivered exactly once** — store it immediately.
  - ⚠️ **Parse JSON — never string-match the body.** It carries **two** `status` fields: the **session**
    status (top-level, `"approved"`) and the nested **`wallet.status`** (`"active"`). Read the
    **top-level** `status` and pull `api_key` off the *parsed* object. A greedy regex / `sed` grabs the
    nested `"active"`, never matches `"approved"`, and silently discards the once-only key — the next
    poll then returns `410 already_claimed` and the key is gone.
- Subsequent polls → **410** `already_claimed` (single-use). Errors: 401 `invalid_session` (bad/unknown
  secret), 409/503 grant states, 502 provisioning.

## What the agent gets & must verify
`api_key`, `wallet_id`, `addresses.{solana,evm}`, `owner_email` (may be `null`). **Persist `api_key` +
`wallet_id` + addresses per the SKILL.md "Credential store" section** (secret store, or
`~/.config/treasures/credentials.json` mode 0600) — the key is **once-only**, so write + read-back in the
same turn; on failure, re-onboard.
Verify the wallet/owner out-of-band (e.g. against the expected owner) **before funding** — the
`owner_email` + addresses echo exists for this misbinding check.

## Funding (after onboarding)
Fund the wallet with **USDC** (the trade quote currency) + a little **native gas**: SOL on Solana
(fees + one-time Token-2022 ATA rent per new asset), a little ETH on Ethereum (one-time ERC-20 approve
on the first trade of a token; the Fusion fill itself is gasless). `GET /wallets/:id/balances` →
`needs_funding`.

Source: `src/api/routes/api/v1/onboarding-sessions.ts`, `src/services/wallet/onboarding-session.ts`.
