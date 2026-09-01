---
name: treasures-wallet
description: >
  Operate a Treasures delegated wallet over the HTTP API: onboard (provision a wallet + mint a
  scoped API key), get quotes, execute buys/sells (async, server-signed — the agent NEVER signs),
  read balances/portfolio/trade-history, and manage API keys. Trigger whenever an agent needs to
  trade tokenized equities (xStocks / Ondo) vs USDC on a Treasures wallet, check a Treasures wallet
  balance or P&L, or set up Treasures wallet access. The agent needs only HTTPS + an API key — no
  web3 libraries, no private keys, no RPC.
metadata:
  version: "1.2.0"
tags:
  - treasures
  - delegated-wallet
  - trading
  - solana
  - ethereum
  - xstocks
  - ondo
  - tokenized-equities
  - api-key
---

# Treasures Wallet

Drive a Treasures **delegated** wallet over HTTP. Treasures custodies the swap signer and executes
on-chain; the agent only submits **intents** with a scoped API key. ⇒ No web3 libs, no keys, no RPC.

## When to use

- Buy/sell tokenized equities (e.g. NVDA, TSLA) as **xStocks** or **Ondo** tokens vs **USDC** on
  Solana or Ethereum, from a Treasures wallet.
- Preview a trade price (quote), check a wallet's balances/positions, or read its trade history / P&L.
- Onboard an agent: provision a wallet + mint a scoped (`trade`/`quote`) API key with optional caps.
- Manage API keys for a wallet (owner-side: create / list / revoke).

Do **not** use for: generic Solana/EVM RPC work, non-Treasures wallets, or anything requiring the
agent to hold a private key — the whole point of this design is that it doesn't.

## Mental model (read this — most of it is non-obvious)

- **Delegated custody. The agent never signs.** It POSTs intents; Treasures holds the swap signer
  and executes. The skill needs only HTTPS + the API key.
- **TWO base paths** (this is the #1 gotcha), both off the host `https://api.treasures.io`:
  - **`API = {host}/api/v1`** — wallet plane + onboarding (`/api/v1/wallets/...`, `/api/v1/onboarding-sessions/...`).
  - **`READS = {host}/public/v1`** — address-scoped reads only (`/public/v1/trades`, `/public/v1/portfolio`).
- **Two identifier types:**
  - `wallet_id` (`wlt_…`) → wallet-scoped endpoints (`API/wallets/:id/...`).
  - sol + eth **addresses** → address-scoped reads (`READS/trades`, `READS/portfolio`). Resolve them
    once from `GET API/wallets/:id` (`addresses.{solana,evm}`) and cache.
- **Auth is split:**
  - **Writes + quotes + trade-status poll** require `X-API-Key: twk_…` (scopes `quote` / `trade`,
    optional caps). The poll (`GET /wallets/:id/trades/:job_id`) takes the **same `trade` key** as the
    `POST` that created the job — it exposes the trade intent + outcome, so it is not keyless.
  - **Reads** (balances, history, portfolio, `GET /wallets/:id`, delegation state) need **no key** —
    public + IP-rate-limited, scoped by the address/id in the request.
- **Chain/protocol naming maps between layers:** balances/positions report `chain: "sol"|"eth"` and
  `issuer: "ondo"|"xstocks"`; quote/trade params take `chain: "solana"|"ethereum"` and
  `protocol: "ondo"|"xstocks"`. **Map `sol→solana`, `eth→ethereum`** when feeding a held position
  back into a quote/trade. (Responses echo `solana`/`ethereum`.)
- **Amounts are atomic integer strings — never JS `number`.** USDC = 6 dp. Stock tokens vary
  (reads expose both `shares` (human) and `raw_token` (on-chain)). Parse with a big-decimal lib.
- **Trading is async + route-first.** `POST /trades` → **202 + job**, then **poll** to terminal.
  Routing runs *before* the 202, so no-route / cap breach / unwhitelisted come back **synchronously**
  as 4xx (no job row).
- **Auto-routing is single-cell for a BUY, multi-leg for a SELL.** Omit `chain`/`protocol` and a buy
  resolves to the *one* best cell by net deliverable across `{solana,ethereum} × {ondo,xstocks}` — it
  never splits. A sell fans out across every venue the wallet holds and returns N legs in one
  response (see the **Sell playbook**). Pinning `chain`+`protocol` narrows either side to one cell.

## Config (the agent supplies)

| Key | Example | Source |
|---|---|---|
| `host` | `https://api.treasures.io` | default; override via `TREASURES_HOST` env |
| `api_key` | `twk_…` | onboarding (returned **once** — store in secret/env, never log/inline) |
| `wallet_id` | `wlt_XD825…` | onboarding poll (`response.wallet.wallet_id`) |
| `sol_address`, `eth_address` | `3HW73Fk…`, `0xaF51…` | onboarding poll (`response.wallet.addresses.{solana,evm}`); cache. Fallback if you only have a `wallet_id`: `GET API/wallets/:id` → `addresses` |

Derived: `API = https://api.treasures.io/api/v1`, `READS = https://api.treasures.io/public/v1`
(generally `API = ${host}/api/v1`, `READS = ${host}/public/v1`).

## Credential store — named profiles (multi-wallet)

An agent can hold **multiple Treasures wallets**, each a named **profile**. A name is a **client-side
label only** — Treasures has no wallet-name field (the canonical id is `wallet_id`), so the name lives
only in this store. The `api_key` is the only **secret** (`host`/`wallet_id`/addresses are not).

**File schema** — `${TREASURES_CONFIG:-~/.config/treasures/credentials.json}`, **`chmod 600`**:
```json
{
  "default": "trading",
  "wallets": {
    "trading":  { "api_key": "twk_…", "wallet_id": "wlt_…", "sol_address": "…", "evm_address": "0x…" },
    "treasury": { "api_key": "twk_…", "wallet_id": "wlt_…", "sol_address": "…", "evm_address": "0x…" }
  }
}
```
`host` is optional per profile — **omit to use the default** (`https://api.treasures.io`); set it only
to pin a profile to a different host.

**Resolve (every session, read in precedence):**
1. **Env vars** (a single profile named `env`): `TREASURES_API_KEY`, `TREASURES_WALLET_ID`, optional
   `TREASURES_HOST` (defaults to `https://api.treasures.io`),
   `TREASURES_SOL_ADDRESS`/`TREASURES_EVM_ADDRESS`. (Secret manager / CI.)
2. **Config file** above — the named `wallets` map + a `default` name.

**Select which wallet to act on (per request):**
- The user named one ("…in **treasury**") → use it (unknown name → list the available names, ask again).
- Else exactly one profile exists → use it.
- Else a `default` is set → use it **and say which one** you're acting on.
- Else (multiple, none specified, no default) → **ask the user which wallet**, listing names + addresses
  (offer a quick `/balances` per wallet if it helps them choose).

**Add a wallet** (after onboarding `poll`, or a pasted `api_key` + `wallet_id`):
- **Ask the user to name it** (suggest a default — the `owner_email` local-part or `wallet-N`). Names are
  unique; on collision, confirm overwrite or pick another. The **first** wallet added becomes `default`.
- The `api_key` is delivered **once** → write the profile **atomically in the same turn**, read it back to
  confirm. Prefer a runtime **secret store** for the `api_key` if one exists; **never** commit/log it.

**"Has a wallet"** ⇔ at least one profile resolves with `api_key` + `wallet_id`; none → **onboard**.
Reads-only (balances/history/portfolio) need just a profile's `wallet_id`/addresses, not the key.

## Step 0 — Preflight (BLOCKING)

Before any quote/trade, **resolve credentials via the Credential store above** (env → file). **Do not skip.**
- **`host`** is `https://api.treasures.io` — use it; don't ask. (Only deviate if a profile or
  `TREASURES_HOST` pins a different `host`.)
- **One or more profiles in the store?** → **select the target wallet** per the Credential store rules
  (named / single / `default` / ask) — then go to the Intent router. Don't silently act on the wrong wallet.
- **No profiles? Don't jump to onboarding — ASK the user first:** *"Do you already have a Treasures wallet?"*
  - **Yes** → ask them to paste **both** their **API key** (`twk_…`) **and** `wallet_id` (`wlt_…`). The
    Treasures app shows them together on the API-keys screen. ⚠️ **The key alone is NOT enough** — it's
    verified against the `wallet_id` in the URL path and there is **no key→wallet lookup endpoint**, so an
    agent given only a `twk_…` cannot discover its wallet. Validate with `GET API/wallets/:id` (confirms the
    id; the key verifies on the first quote/trade), then **ask the user to name it** and **persist the
    profile** (see Credential store → Add a wallet).
  - **No** → offer to **begin onboarding** (next bullet). On completion, **ask the user to name the wallet**
    before persisting.
- Reads-only (balances/history/portfolio) need just a `wallet_id` or addresses, not a key.
- **Onboarding REQUIRES a human and cannot be done headlessly.** The agent only **mints** the session
  and **polls** for the key; a human must open the returned `url`, log in via Privy, and grant the swap
  delegation in their browser. **If you have no way to reach a human, STOP** and surface the onboarding
  `url` with a request to complete it — do not spin or report a generic failure. See
  [`references/onboarding.md`](references/onboarding.md) for the exact loop. After onboarding the wallet
  is **empty** — it must be funded (USDC + a little native gas) before any trade.

## Intent router

| Intent | Capability | First call |
|---|---|---|
| Onboard / get a key | [Onboarding](references/onboarding.md) | `POST API/onboarding-sessions` → poll |
| Price preview | Quote | `GET API/wallets/:id/quotes` (X-API-Key, scope `quote`) |
| Buy | Trade | `POST API/wallets/:id/trades` (X-API-Key, scope `trade`) → poll |
| Sell (any size) | **Sell playbook** (below) | one `POST API/wallets/:id/trades` → poll every `legs[].job_id` |
| Balances / funding | Balances | `GET API/wallets/:id/balances` (no key) |
| Trade history / P&L | History / Portfolio | `GET READS/trades`, `GET READS/portfolio` (no key) |
| Manage keys (owner) | [API keys](references/api-keys.md) | owner Privy session — agent key can't |

Full per-endpoint contracts: [`references/endpoints.md`](references/endpoints.md). Validated curl
examples: [`references/examples.md`](references/examples.md).

## Quote — `GET API/wallets/:id/quotes`

Auth `X-API-Key` (scope `quote`). Query: `chain?`,`protocol?`,`side`,`asset`,(`notional_usdc` XOR
`shares`),`slippage_bps` (≤ 500). Omit `chain`/`protocol` to auto-route (single best cell). 200 on a
**buy** → `{chain,protocol,side,asset,max_amount_in,min_amount_out,route_type}` — **atomic strings**
in the input/output asset; `chain`+`protocol` echo the **resolved** cell. 200 on a **sell** →
`{side,asset,route_type,legs:[{chain,protocol,shares_consumed,max_amount_in,min_amount_out}]}` — one
entry per cell the sell draws from. Advisory only — the trade re-quotes route-first at submit.

**Check `tradability`/`warn_reason` before submitting.** A buy carries them (plus `thin_since`) at the
top level for the resolved cell; a sell carries them on each `legs[]` entry, side-neutral reasons only.
A cell can price normally and still be warned (`settlement_failure` = quotes fine, settles never). Absent
means unmeasured or unwarned — never `null`. Per-reason action table in
[`references/endpoints.md`](references/endpoints.md#tradability).

## Buy — `POST API/wallets/:id/trades`

Auth `X-API-Key` (scope `trade`). **Headers:** `Idempotency-Key: <uuid>` (**required** — 400
`missing_idempotency_key` if absent; base64url/UUID charset `[A-Za-z0-9_-]`, ≤ 128 chars → 400
`invalid_idempotency_key` otherwise), `Content-Type: application/json`. Body (`.strict()`):

```json
{ "chain":"solana", "protocol":"ondo", "side":"buy",
  "asset":"NVDA", "size":{"notional_usdc":"10"}, "slippage_bps":100 }
```

`chain`/`protocol` optional (auto-route). A **buy** → **202 + one flat job** (`job_id` at the top
level); a **sell** → **202 + `{order_status, legs[]}`** (see the sell playbook — `job_id` is per leg).
Branch on the `side` you sent, never on inspecting the body. Then poll `GET API/wallets/:id/trades/:job_id`
(**same `X-API-Key`, scope `trade`** — the poll is gated like the create). State machine:
`routing → simulating → signing → broadcast → {confirmed | failed | rejected}`.
- `confirmed` → `result = { amount_in, amount_out, tx_sig }` (atomic in/out). **EVM:** the `tx_sig`
  shown at `broadcast` is the **order hash**; `result.tx_sig` at `confirmed` is the real
  **on-chain fill** hash.
- `failed` / `rejected` → **no funds moved**; `reject_reason` says why.

**Timing:** Solana confirms in **~2 s**; **EVM is ~20–40 s** (resolver-settled). Bound the poll
(~2 min) and treat a long-wedged `broadcast` as **unknown, not confirmed** — see Behaviors.

**Buys route the FULL notional to ONE cell — do NOT split a buy.** This mirrors B2B `planBuy`, which
quotes the full amount at each cell, keeps the best cell per chain, and executes exactly one (the
cheapest). Buys aren't holdings-constrained, so single-best-cell *is* best execution — there's no
greedy/additive buy fill (that's sell-only). A single buy is one swap on one chain and **cannot combine
cross-chain USDC**: it needs the full notional on the resolved chain, or it fails on insufficient funds.
(Only if you must deploy more USDC than sits on any single chain do you split client-side into per-chain
buys — ranked by *most shares per dollar* — but that's a deliberate cross-chain-funding case, not the default.)

## Sell playbook — one call, N legs

**The server splits a sell.** `POST /trades` with `side:"sell"` plans across every venue the wallet
holds and executes N independent jobs, returning ONE `202`:

```json
{ "order_status": "pending",
  "legs": [ { "job_id": "job_s0", "chain": "sol", "protocol": "ondo",     "state": "broadcast",  ... },
            { "job_id": "job_s1", "chain": "eth", "protocol": "xstocks", "state": "confirmed", ... } ] }
```

- **`job_id` lives on each leg** — `legs[i].job_id`, never at the top level. This is the one place
  the sell shape differs from a buy, and reading a top-level `job_id` here is the most common way to
  mis-parse a sell.
- **Poll every leg** to terminal. Legs settle at different speeds (Solana ~2 s, EVM ~20–40 s), so
  the order is not finished when the first one lands. Poll them concurrently, not in sequence.
- **`order_status`** rolls the legs up: `pending` while any leg is non-terminal, else `confirmed`
  (all legs confirmed), `failed` (none), or `partially_filled` (some). It is a snapshot taken at
  submit — re-derive it from the polled legs rather than trusting the 202's copy.
- **`partially_filled` is a real outcome, not an error.** A leg that cannot re-quote lands as a
  terminal `failed` job and stays visible; nothing is hidden. Sum `result.amount_out` over the
  confirmed legs for what you actually received.
- **One `Idempotency-Key` for the whole order**, not one per leg — the split happens server-side of
  that key.

**Pinning still narrows to a single venue.** Send `chain`+`protocol` and the sell routes to that one
cell; larger than it holds → **`422 quote_unavailable`** (`holdings_insufficient`). The response
shape does not change — still `{order_status, legs}`, with a single entry. Pin only when you want
that specific venue: the default fans out and fills more.

**Dust:** selling exact share amounts leaves tiny remainders. To fully empty a position, sell its
reported `shares` value (or document a small tolerance).

## Reads (no key)

- **Balances** — `GET API/wallets/:id/balances` → `{native,stablecoins,positions,needs_funding,as_of}`.
  On-chain truth; positions carry `shares` (human) + `raw_token` (atomic) + `notional_usd`.
- **Trade history** — `GET READS/trades?sol_wallet=&eth_wallet=&source=&limit=&offset=` (≥1 address).
  `source=all` (default) vs `internal` does **not filter** — it only toggles whether P&L columns
  (`usd_per_share`, `avg_entry_price_per_share`, `realized_pnl`) are populated. `internal` rows are the
  agent's executed trades (carry `side`/`tx_hash`/`usdc_amount`); `external` rows are on-chain transfers
  Treasures didn't execute.
- **Portfolio (P&L)** — `GET READS/portfolio?sol_wallet=&eth_wallet=&source=` → positions + `usd_value`;
  `source=internal` adds `shares_internal_only`, `avg_entry_price_per_share`, `unrealized_pnl`. A
  position may also carry `tradability`/`warn_reason`/`thin_since` for its own cell — side-neutral
  reasons only (`low_volume`, `no_price_feed`, `no_settlements`); it never marks a holding unsellable.

## Onboarding & API keys (summary — detail in references)

- **Onboard (agent + human, RFC 8628 device grant):** agent `POST API/onboarding-sessions` (optional
  `{api_key:{scopes,caps}}`) → `{request_id, device_secret, url, …}`; send the human the `url` (they log
  in via Privy + grant delegation in-browser); agent polls `POST API/onboarding-sessions/poll` (Bearer
  `device_secret`) until `{status:"approved", wallet, owner_email, api_key}` — **the key is delivered
  once**. See [`references/onboarding.md`](references/onboarding.md).
- **API-key management is owner-only** (Privy access+identity session — the agent's key **cannot** mint
  keys). `POST/GET/DELETE API/wallets/:id/api-keys`; scopes `['trade','quote']`; caps
  `{max_trade_notional_usd, daily_notional_usd, asset_allowlist, max_slippage_bps}`. See
  [`references/api-keys.md`](references/api-keys.md).

## Behaviors the skill MUST implement

1. **One `Idempotency-Key` per attempt.** Re-POSTing the **same** key returns the **same** job (safe to
   retry a network blip). A new attempt needs a **new** key.
2. **Retry transients with a FRESH key.** `503 routing_unavailable` (provider/RPC blip) → retry w/
   backoff. `422 quote_unavailable` (genuine no-route / thin-liquidity no-fill — eth cells can
   intermittently no-fill) → bounded retry (couple), then surface. A terminal `failed`/`rejected` (esp.
   Solana Ondo RFQ) can also be transient. All of these = **no funds moved** → retry with a **new**
   `Idempotency-Key` (old key just replays the dead job). Cap ~3 attempts. Treasures does **not**
   auto-retry server-side.
3. **Poll to terminal; bound it.** Solana ~2 s, EVM ~20–40 s. Treat `confirmed|failed|rejected` as
   terminal; otherwise keep polling up to ~2 min. **A job can wedge in `broadcast`** (e.g. a tx that
   never landed) — after the bound, treat it as **unknown** (don't assume confirmed); the server-side
   reconciler force-fails wedged jobs. Re-check `/balances` + `/trades?source=internal` for truth.
4. **Honor caps without blind-retry.** A cap breach is **403** `{error:"key_cap_exceeded", cap, reason}`
   (`asset_allowlist`, `max_slippage_bps`, `max_trade_notional_usd`, `daily_notional_usd`). Surface it;
   don't retry.
5. **Big-decimals + decimals.** Quote/job amounts are atomic; convert with the asset's decimals (USDC 6;
   reads expose `shares` vs `raw_token`). Never use floats.
6. **Gas / funding:** Solana needs SOL (fees + one-time Token-2022 ATA rent per new asset); the first
   Ethereum trade of a token needs a little ETH for a one-time ERC-20 approve (the fill itself is
   gasless). Surface `needs_funding` from `/balances`.
7. **Read the preview's `tradability`/`warn_reason` before every trade.** They are advisory — a warned
   cell still quotes and still executes — so the agent decides: pin another `chain`+`protocol` on
   `settlement_failure`/`no_fill_window`/`no_settlements`, skip the asset on `no_price_feed`, keep sizes
   small on `low_volume`. `warn_reason` is an open vocabulary — treat an unlisted value as "warned,
   reason unknown" rather than an error. Table in
   [`references/endpoints.md`](references/endpoints.md#tradability).

## Reusable helpers (TypeScript; adapt to your runtime — `big.js` for decimals)

```ts
import Big from 'big.js';
const USDC_DECIMALS = 6;
// `apiKey`, `walletId` come from config; never log apiKey.
// `host` defaults to the Treasures API; override via TREASURES_HOST if set.
const host = process.env.TREASURES_HOST ?? 'https://api.treasures.io';
const API = `${host}/api/v1`, READS = `${host}/public/v1`;

const SKILL_NAME = 'treasures-wallet';
const SKILL_VERSION = '1.2.0'; // = SKILL.md metadata.version

async function tFetch(url: string, init: RequestInit = {}): Promise<any> {
  const res = await fetch(url, {
    ...init,
    // Sending the version opts into the gate: deprecation warnings while this skill ages, and a
    // clean 426 with upgrade instructions once it is past sunset, instead of a silent break.
    headers: { 'X-Treasures-Skill': SKILL_NAME,
               'X-Treasures-Skill-Version': SKILL_VERSION, ...init.headers },
  });
  const body = await res.json().catch(() => ({}));
  if (res.status === 426) throw { status: 426, ...body };  // STOP: relay body.upgrade, do NOT retry
  if (res.headers.get('Deprecation'))                      // non-blocking — finish, then warn
    console.warn(`skill deprecated — sunset ${res.headers.get('Sunset') ?? 'not yet set'}; ` +
                 `${res.headers.get('Warning') ?? 'update the Treasures skills'}`);
  notifyIfNewer(res);
  if (!res.ok) throw { status: res.status, ...body };   // {status, error, cap?, reason?}
  return body;
}

// `X-Treasures-Skill-Latest: treasures-b2b-api=…; treasures-wallet=…` rides every response from the
// wallet endpoints, whether or not you send a skill header. Informational only — never block, retry
// or alter a trade because of it. Warn once.
let updateNotified = false;
function notifyIfNewer(res: Response): void {
  if (updateNotified) return;
  const header = res.headers.get('X-Treasures-Skill-Latest');
  const latest = header?.split(';')
    .map((pair) => pair.trim().split('='))
    .find(([skill]) => skill === SKILL_NAME)?.[1];
  if (!latest || latest === SKILL_VERSION) return;
  updateNotified = true;
  console.warn(
    `A newer ${SKILL_NAME} skill is available (${latest}; you have ${SKILL_VERSION}). ` +
    `What's new: https://github.com/treasures-io/treasures-finance-agent-skills/blob/main/CHANGELOG.md`
  );
}
// Submit a trade intent. `idempotencyKey` MUST be fresh per attempt.
const trade = (intent: object, idempotencyKey: string) =>
  tFetch(`${API}/wallets/${walletId}/trades`, {
    method: 'POST',
    headers: { 'x-api-key': apiKey, 'content-type': 'application/json',
               'Idempotency-Key': idempotencyKey },
    body: JSON.stringify(intent),
  });

const TERMINAL = new Set(['confirmed', 'failed', 'rejected']);
async function pollJobToTerminal(jobId: string, ms = 120_000): Promise<any> {
  const deadline = Date.now() + ms;
  for (;;) {
    const job = await tFetch(`${API}/wallets/${walletId}/trades/${jobId}`,
      { headers: { 'x-api-key': apiKey } }); // same trade key as the POST
    if (TERMINAL.has(job.state)) return job;
    if (Date.now() > deadline) return { ...job, state: 'unknown' };          // wedged → not confirmed
    await new Promise((r) => setTimeout(r, 2000));
  }
}

// 503 / 422 / terminal-fail all mean "no funds moved" → retry with a FRESH key.
async function withFreshKeyRetry<T>(run: (idk: string) => Promise<T>, tries = 3): Promise<T> {
  for (let attempt = 1; ; attempt++) {
    try { return await run(crypto.randomUUID()); }       // new Idempotency-Key each attempt
    catch (err: any) {
      const retryable = err?.status === 503 || err?.status === 422;
      if (!retryable || attempt === tries) throw err;
      await new Promise((r) => setTimeout(r, 500 * 2 ** attempt));
    }
  }
}

const fromAtomic = (atomic: string, decimals: number) => new Big(atomic).div(new Big(10).pow(decimals));

// A sell is ONE POST; the server splits it. Poll every leg — they settle at different speeds.
async function sell(asset: string, shares: string, slippageBps: number) {
  const order = await withFreshKeyRetry((idk) => trade({
    side: 'sell', asset, size: { shares }, slippage_bps: slippageBps,
  }, idk));
  const legs = await Promise.all(
    order.legs.map((leg: { job_id: string }) => pollJobToTerminal(leg.job_id))
  );
  const filled = legs.filter((leg: any) => leg.state === 'confirmed');
  const received = filled.reduce(
    (sum: Big, leg: any) => sum.plus(fromAtomic(leg.result.amount_out, USDC_DECIMALS)),
    new Big(0)
  );
  return {
    // Re-derived from the polled legs; the 202's order_status was a submit-time snapshot.
    order_status: filled.length === legs.length ? 'confirmed'
      : filled.length === 0 ? 'failed' : 'partially_filled',
    usdc_received: received.toString(),
    legs,
  };
}
```

## Error catalog

| HTTP | body | action |
|---|---|---|
| 400 | `invalid_request_body` / `missing_idempotency_key` / `invalid_idempotency_key` | malformed — fix, don't retry |
| 401 | `invalid_api_key` / `invalid_session` | missing/bad key (or onboarding secret) |
| 403 | `{error:"key_cap_exceeded", cap, reason}` | cap breach — surface, don't retry. Also `policy_denied` = delegation off |
| 404 | `wallet_not_found` / `job_not_found` / `session_not_found` | bad id |
| 410 | `already_claimed` | onboarding key already claimed (single-use) |
| 422 | `asset_not_whitelisted` | ticker not in catalog / not routable on the pinned cell |
| 422 | `quote_unavailable` / `quote_failed` | genuine no-route, thin-liquidity no-fill, **or a PINNED sell bigger than that one cell holds** (drop the pin and let it fan out) → bounded retry |
| **503** | `routing_unavailable` / `grant_check_unavailable` | **transient → retry w/ backoff (fresh key)** |
| 202 → `failed`/`rejected` | `reject_reason` | no funds moved → retry (fresh key) for transient reasons |

> `503 routing_unavailable` = retryable infra blip; `422 quote_unavailable` = genuine no-route. Both
> quote (GET) and trade (POST) use this split.
