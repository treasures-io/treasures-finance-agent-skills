---
name: treasures-b2b-api
description: Use to build an AI agent on the Treasures public B2B API — discover tokenized stocks, quote/execute trades on Solana, Ethereum, Robinhood Chain and Base, bridge USDC across Solana and Ethereum, and read portfolio + trade history for a single end-user wallet pair. Covers endpoint selection, ownership-proof signing (incl. embedded wallets), trade/bridge execution, and error handling.
metadata:
  version: "1.14.0"
tags:
  - treasures
  - b2b-api
  - tokenized-stocks
  - solana
  - ethereum
  - base
  - trading
  - bridge
  - usdc
  - ownership-proof
---

# Treasures B2B API — Agent Skill

Guide for AI agents (and any non-human caller) using the Treasures public B2B API to discover tokenized stocks, quote + execute trades, bridge USDC across chains, and read portfolio + trade history on behalf of a single end-user wallet pair.

- **Base URL:** `https://api.treasures.io/public/v1`
- **Network:** trades execute against **Ethereum mainnet + Solana mainnet-beta — real funds, not a testnet**, plus two more venues: **Robinhood Chain (4663)** — now co-ranked in unpinned quotes, not just pinned — and **Base (8453)**. Point your own RPCs at mainnet; the token/contract addresses in this skill are mainnet.
- **Required headers:** set `Content-Type: application/json` on every request carrying a body — a body sent with a **non-JSON** content-type (e.g. `text/plain`, which some HTTP clients default to when you don't set headers) fails `415` before any schema check. Also send `X-Treasures-Skill: treasures-b2b-api` and `X-Treasures-Skill-Version: 1.14.0` (this skill's `metadata.version`): today they're informational (the fund-moving endpoints echo `X-Treasures-Api-Revision` + `X-Min-Skill-Version` back), but once the API floor rises, an enrolled caller below it gets a clean `426 skill_version_unsupported` with upgrade instructions — plus `Deprecation`/`Sunset` warning headers during the grace window — instead of a silent break. Omitting the version header opts out of that early warning.
- **Wire format:** all token amounts, USDC, shares, prices, and bps-derived decimals are **strings** (avoid JS float drift). Integer fields (`expires_at`, `*_bps`, `quote_index`) are JSON numbers. Never round-trip a money value through JS `number`.

This entry doc is the map + the footguns. **It is not enough on its own to execute a trade** — full schemas, signing code, and error tables live in the references below. **Load the reference for the task before you act; you don't need to read them all.**

> **Schema-of-record:** an OpenAPI spec (`openapi.yaml`) is published alongside this skill in the same repository. This skill is self-sufficient for normal use — consult the spec only for exhaustive field-level validation or client codegen; you don't need to read it to operate.

## Reference index

| Load this                                          | When you're…                                                                                                |
| -------------------------------------------------- | ----------------------------------------------------------------------------------------------------------- |
| [`references/auth.md`](references/auth.md)         | building the `ownership_proof` (esp. on embedded/managed Solana wallets)                                    |
| [`references/trading.md`](references/trading.md)   | quoting/submitting a buy or sell, signing trade legs, polling quote status, or setting first-time approvals |
| [`references/bridging.md`](references/bridging.md) | bridging USDC between chains (incl. the `nonce=0` EVM trap)                                                 |
| [`references/data.md`](references/data.md)         | reading `/stocks/*`, `/portfolio`, `/trades`, or `/settlements` (your own settled trades)                     |
| [`references/errors.md`](references/errors.md)     | handling an error code, rate limits, or simulating a tx before broadcast                                    |

## TL;DR — happy path

```
1. GET   /stocks/tickers              → supported tickers + token addresses
2. GET   /stocks/prices?tickers=AAPL  → live price snapshot
3. ONE-TIME: set ERC-20 allowances on Ethereum (see traps below)  ← CRITICAL
4. POST  /quote/buy   (or /sell)      → quote(s) + signable payload(s)
5. Sign each signable_payload with the matching wallet key
6. POST  /trade/submit                → broadcast
7. GET   /quote/{quote_id}/status     → poll until terminal
8. GET   /portfolio                   → reconciled holdings
```

Need cross-chain USDC first? Insert a bridge between steps 2 and 4: `POST /bridge/quote` → sign + broadcast it yourself → `GET /bridge/{id}/status` until `completed` or `failed`. See [`references/bridging.md`](references/bridging.md).

## Before your first call — pre-flight traps

These cause silent or first-time-only failures. Handle them up front.

**1. Ethereum ERC-20 approvals are required before any transfer.** The API does **not** broadcast approvals for you; set them once per `(token, spender)` from the user's wallet, or your first Eth sell/bridge fails (`not_enough_balance_or_allowance` / silent forever-pending bridge). Solana has no allowances — skip on `sol` legs.

| Flow                    | Approve                                                         | Spender                                                                                 |
| ----------------------- | --------------------------------------------------------------- | --------------------------------------------------------------------------------------- |
| Eth buy (USDC → stock)  | USDC `0xA0b86991c6218b36c1d19D4a2e9Eb0cE3606eB48`               | settlement `0x111111125421ca6dc452d289314280a0f8842a65`                                 |
| Eth sell (stock → USDC) | each protocol's stock token (per ticker — a sell may span both) | settlement (same)                                                                       |
| Base buy (USDC → stock) | **Base-native** USDC `0x833589fCD6eDb6E08f4c7C32D4f71b54bdA02913` | settlement `0x111111125421ca6dc452d289314280a0f8842a65` — same address as Ethereum's, but allowances are **per-chain**: an Ethereum approval grants nothing on 8453 |
| Base sell (stock → USDC)| the ticker's B20 token (`coinbase` block `address`)             | settlement (same)                                                                       |
| `eth → sol` bridge      | USDC                                                            | `signable_payload.approval_spender` from the bridge quote (non-null on the normal flow) |
| Robinhood / Base **speed route** (`priority:"speed"`) | **none** — the API bundles the approve | the quote itself carries a `role:"approve"` payload before the `swap` when the wallet has no allowance yet; sign both ([trap 9](#trap-9)) |

Set `allowance = max-uint256` once. Approve code + the `approval_spender` null/non-null rule: [`references/trading.md`](references/trading.md#first-time-approvals).

**2. Hold the input token on the chain you trade.** A buy spends **USDC on the chosen chain**; a sell spends the **stock token**. The server pre-checks _sell_ holdings (insufficient → `422 holdings_insufficient` at quote time) but does **NOT** pre-check _buy_ USDC — an underfunded buy passes `/quote/buy` **and** `/trade/submit`, then the on-chain swap fails at the end (`broadcast_failed` / `not_enough_balance_or_allowance`), wasting the whole flow. **Verify USDC before quoting**: read the wallet, or call `GET /portfolio` (`usdc.{sol, eth, base}`). Need USDC on the other chain? Bridge first.

**3. Native gas: Treasures does not subsidize it.** Any operation your key broadcasts needs native balance on that chain. Eth, Robinhood and Base buys/sells are gasless **by default** (the settlement network pays), but approvals + bridge broadcasts cost native gas; all Solana txs cost SOL. Keep ≥ **0.02 SOL**, ≥ **0.01 ETH** on mainnet, and ≥ **0.001 ETH on Base** (approvals only). Robinhood needs no native float at all — **unless you send `priority:"speed"`** (trap 9), which makes that leg's gas yours to fund on 4663 or 8453. Detail: [`references/errors.md`](references/errors.md#gas).

**4. The EVM bridge tx ships with `nonce=0`.** You MUST inject the wallet's real nonce before signing or broadcast is rejected ("nonce too low"). Most common bridge footgun — see [`references/bridging.md`](references/bridging.md#evm-sign-broadcast).

**5. Solana ownership proof is base64, NOT base58.** Privy/Turnkey quickstarts show base58 (`bs58.encode`) — that's the #1 cause of `ownership_proof_sol_invalid`. See [`references/auth.md`](references/auth.md#embedded-wallets).

**6. `chain:"robinhood"` is auto-routed (no longer pin-only), priced in USDG, and liquid-names-only.** The Robinhood Chain venue (Robinhood Stock Tokens, chain id **4663**, priced in **USDG**) is **co-ranked in `chain:null` quotes** — an unpinned buy can return a USDG leg, and an unpinned sell greedy-fills robinhood positions with the rest, so **branch on each leg's `base_asset`, never on whether you sent `chain`**. Pass an array — `chain:["sol","eth"]` — to restrict the auto-route to a subset (e.g. USDC-only legs); you get up to one leg per chain in the set, and `["x"]` is the same as `"x"`. The venue uses the **same sign-only, gasless model as `eth`**: `/quote/buy` **or `/quote/sell`** with `chain:"robinhood"` returns one `evm_eip712_typed_data` order — sign it (`eth_signTypedData_v4`) and submit `{ type:"evm_eip712_signature", signature }`. You **broadcast nothing and pay no gas** — no 4663 RPC and no native float needed. Three things to know: (a) amounts are **USDG** — read `base_asset` + `amount_base`, not the `_usdc`-named fields; (b) only the **liquid** marquee names (AAPL/TSLA/NVDA/AMD) fill — others return `422 no_routes` even though they list on `/stocks/tickers`; (c) you still must fund the input on 4663 (USDG to buy, Stock Tokens to sell) and **submit** each trade for it to enter `/trades` — there is no external-row reconciler here, so an unsubmitted trade never appears there regardless of your balance. `/portfolio` does not wait for that submission, though: your 4663 Stock Tokens are read for **any** wallet when you send your `tik_` integrator key (and otherwise for every wallet Treasures knows on any chain), snapshotted for 30 s and invalidated the moment a trade settles — a token transferred in from outside shows exactly like one bought here. `X-API-Key` is OPTIONAL on `/portfolio`, so an ANONYMOUS call for an `eth_wallet` Treasures has never seen gets an empty `positions` list and is not flagged `partial` — **send the key and that case disappears**; `usdg.robinhood` cash is never gated either way — it still reads live even for an unknown wallet. Buy and sell are both supported, pinned or unpinned (a sell is sized against your on-chain 4663 balance). Full flow: [`references/trading.md`](references/trading.md#robinhood).

**7. `chain:"base"` is priced in Base-native USDC and listed only once a token is minted.** The Coinbase venue (**B20 tokenized equities**, Base, chain id **8453**) is co-ranked in unpinned quotes once a ticker is minted and priceable — rare today, so treat an unpinned base leg as possible, not expected. It uses the **same sign-only, gasless model as `eth`**: `/quote/buy` or `/quote/sell` with `chain:"base"` returns one `evm_eip712_typed_data` order — sign it (`eth_signTypedData_v4`) and submit `{ type:"evm_eip712_signature", signature }`. Four things to know: (a) `base_asset` is **`"usdc"`**, but it is **Base-native USDC** (`0x8335…2913`), a *different contract* from mainnet USDC — branch on the leg's `chain`, never on `base_asset` alone; (b) the venue is **supply-gated** — Coinbase deployed its tokens unminted, and a ticker appears on `base` only once real supply exists, so **read `available_chains` from `/stocks/tickers`** instead of assuming a ticker trades there; (c) you need a **one-time allowance on 8453** (see the table above) and therefore a **small ETH float on Base** to send that approval — the trade itself is gasless; (d) `/portfolio` **does** report Base — B20 positions (`chain:"base"`, `protocol:"coinbase"`) and your Base USDC cash under **`usdc.base`**, a sibling of `usdc.sol`/`usdc.eth` but a *different contract* from mainnet USDC. Positions are read for **any** wallet when you send your `tik_` integrator key (and otherwise for every wallet Treasures knows on any chain) — snapshotted for 30 s and invalidated when a trade settles — so a token transferred in directly shows exactly like one bought here; an ANONYMOUS call for an `eth_wallet` Treasures has never seen gets an empty `positions` list and is not flagged `partial`, so **send the key**. **Cash is never gated** — it still reads live even for an unknown wallet. Still **submit** every trade: `/trades` has no external-row reconciler on this venue, so an unsubmitted trade never appears there. Full flow: [`references/trading.md`](references/trading.md#base).

**8. `MAG7X` is a basket, and it is the one ticker nothing independent price-checks.** The Ondo Intelligent Portfolio lists on `eth` only and holds **nine** constituents — AAPL, AMZN, **ETHA**, GOOGL, **IBIT**, META, MSFT, NVDA, TSLA. `IBIT` is a spot-Bitcoin ETF and `ETHA` a spot-Ether ETF, so it is **part crypto-ETF, not a pure Magnificent-7 basket** — say that to a user before they buy it, because the ticker implies otherwise. Three consequences: (a) `tradfi` is **permanently `null`** on `/stocks/prices` — a basket has no market-data profile, so this is not an outage and there is no premium to compute; (b) every other ticker's quote is checked against its reference market price, and this one **cannot be** — the only check left is the issuer's own published price; (c) when that price is stale or non-positive the **buy** is refused with **`422 basket_price_unavailable`** (transient — retry in a few minutes; no size or slippage change clears it) — **`/quote/sell` is never gated this way**, so a holder can always exit. Detail: [`references/data.md`](references/data.md#mag7x).

<a id="trap-9"></a>**9. `priority:"speed"` trades one-block settlement for your own gas and nonce — and may hand you TWO transactions to sign.** Sending `priority:"speed"` on `/quote/buy`, `/quote/sell` or `/quote/preview` asks for the **speed route** on `robinhood` and `base`: instead of a gasless order, those legs come back as **complete unsigned transactions** you sign whole (`evm_signed_tx`) and Treasures broadcasts. Consequences, all yours: the wallet must hold **native ETH on that chain** (`insufficient_native_gas` otherwise); **no pre-approval is needed** — a wallet that has not approved the router gets `signable_payloads: [ {role:"approve"}, {role:"swap"} ]` with consecutive nonces, and that approve grants the router an **unlimited allowance (`max-uint256`)** on the token being spent — you are signing that consent, so surface it to your user — and you must sign and return **both, in the order issued** (signing only `[0]` is a shape error, `incomplete_submit`; reordering them on a replay is read as a new submit, `leg_already_submitted`); and only **one speed trade per wallet per chain** can be in flight (a second collides as `nonce_conflict`; wait for the first to go terminal, then re-quote). Never re-nonce, re-estimate or re-serialize a payload — the server compares the signed bytes with the ones it issued, matching each by nonce. On a pair `tx_hash` is the swap's from the start but appears on chain only after the approve mines (seconds); `approve_failed` means the approve reverted or did not land in 10 min and the swap was never sent (`tx_hash` is then `null`, as it also is when the node refuses the approve outright with `nonce_conflict` or `provider_error`) — re-quote. It is a filter, not a preference: `eth` is never quoted under `priority:"speed"` (`chain:"eth"` + speed is `400 invalid_request`), and a robinhood/base leg the speed route can't serve is **absent** from `quotes[]` with a `speed_route_unavailable` warning saying why — never handed back on its gasless route. Only `sol` and speed-route legs can come back, so a `gasless: true` leg under speed is always `sol` — **unless you also send `execution:"user_operation"`**: then a `base` speed leg comes back `gasless: true` as ONE `evm_calls` payload you pack into a sponsored ERC-4337 UserOperation from an EIP-7702-delegated (Modular Account v2) wallet, sign, and return as `evm_user_operation`; the gas is your paymaster's, there is no nonce to collide on, and a refusal to sponsor is `sponsorship_rejected`. An undelegated wallet is `400 wallet_not_delegated`; `robinhood` is not served on that lane yet. Full rules: [`references/trading.md`](references/trading.md#speed) and [`#user-operation`](references/trading.md#user-operation).

## Auth essentials

Three endpoints require an `ownership_proof`: `POST /quote/buy`, `POST /quote/sell`, and `POST /bridge/quote` (**both** `sol_signature` + `eth_signature` mandatory on bridge — it spans both chains). Three more require your integrator key in `X-API-Key`: `GET /settlements`, `POST /quote/preview` and `GET /stocks/{ticker}`. Everything else is unauthenticated; `/trade/submit` is gated by the per-leg signed payload itself.

Sign this exact UTF-8 byte string (lines joined with `\n`) with each wallet's key:

```
treasures-finance-quote-v1
{issued_at}
{sol_wallet|""}
{eth_wallet|""}
```

- `eth_wallet` **must be lowercased** in the challenge. Empty string (not `null`/omitted) for a side you didn't supply. No extra fields — schema is `.strict()`.
- `sol_signature`: **base64** raw Ed25519 over the challenge bytes. `eth_signature`: `0x` 65-byte **EIP-191 `personal_sign`** (not EIP-712).
- **Validity:** `issued_at` is unix seconds; accepted in `[now − 300s, now + 30s]`, else `401 ownership_proof_skewed`. One proof is reusable within that window (no nonce store) — a both-wallets proof works on all three endpoints; a single-wallet proof on `/quote/*` only (bridge needs both signatures).

Signing code, all-or-nothing per-proof rules, embedded-wallet troubleshooting, and deterministic test vectors: [`references/auth.md`](references/auth.md).

## Critical contracts — read before trading

- **`broadcast_unknown` → do NOT retry.** Lost upstream response; the order may have landed. Poll `/quote/{id}/status` (or `/trades`) to confirm on-chain outcome before re-quoting. Retrying risks a double-fill.
- **Slippage bounds:** `max_slippage_bps` ∈ `[10, 5000]` for `/quote/*`, `[10, 500]` for `/bridge/*`. Out of range → `400 invalid_slippage` (response echoes the cap).
- **Quote TTLs are tight** (~25–55s for trades, ~30s bridge). If sign+submit takes longer than `expires_at − 5s`, expect `410 quote_stale` — re-quote.
- **Buy = pick ONE `quote_index`; sell = submit ALL of them.** Buy with >1 leg → `400 quote_index_mismatch`; sell subset → `400 incomplete_submit`.
- **Check `tradability`/`warn_reason` before submitting.** Quote legs, `/stocks/tickers` and `/stocks` carry them; a leg can price normally and still be warned (`settlement_failure` = quotes fine, settles never). Action table in [`references/trading.md`](references/trading.md).
- **Read `warnings[]` on every quote.** Always present (`[]` when clean); a non-empty set comes with `warnings_ack_token`. A warned quote is still executable — this API discloses and does not gate — and `acknowledged_warnings` on `/trade/submit` is **optional**, recording your acknowledgement when echoed and refusing (`422 warnings_not_acknowledged`) only when you send a token that matches nothing we served. What you tell your own end user, and the evidence that you did, is yours to decide and to keep. Codes and the ack contract: [`references/trading.md`](references/trading.md#warnings).
- **Must-handle errors** (full matrix + backoff policy in [`references/errors.md`](references/errors.md)):

| HTTP      | `error`                                                                       | Action                                                        |
| --------- | ----------------------------------------------------------------------------- | ------------------------------------------------------------- |
| 426       | `skill_version_unsupported`                                                   | STOP — skill too old for the current API; do **not** trade or retry. Relay the response's `upgrade` command (see Version & compatibility) |
| 403       | `address_blocked`                                                             | Wallet tied to a sanctioned entity — do not retry that wallet |
| 410 / 404 | `quote_stale` / `quote_not_found`                                             | Re-quote                                                      |
| 422       | `holdings_unknown` / 502 `provider_unavailable` / 503 `screening_unavailable` | Transient — backoff policy B, cap 5 attempts                  |
| 429       | `Too many requests`                                                           | Honor `Retry-After` (delta-seconds)                           |

## Endpoint index

| Endpoint                                         | Auth                                | Method |
| ------------------------------------------------ | ----------------------------------- | ------ |
| `/stocks/tickers` · `/stocks/prices?tickers=…`   | none                                | GET    |
| `/stocks/{ticker}`                               | `X-API-Key` (required)              | GET    |
| `/quote/buy` · `/quote/sell`                     | `ownership_proof`                   | POST   |
| `/quote/preview`                                 | `X-API-Key` (required)              | POST   |
| `/quote/{quote_id}/status`                       | none                                | GET    |
| `/trade/submit`                                  | signed payload (per leg)            | POST   |
| `/bridge/quote`                                  | `ownership_proof` (both signatures) | POST   |
| `/bridge/{bridge_quote_id}/status`               | none                                | GET    |
| `/portfolio?sol_wallet=&eth_wallet=&source=`     | none                                | GET    |
| `/trades?sol_wallet=&eth_wallet=&limit=&offset=&source=` | none                        | GET    |
| `/settlements?limit=&cursor=&chain=&protocol=&ticker=&side=&token_out_address=&settled_from=&settled_to=` | `X-API-Key` (required) | GET |

## Version & compatibility

Sending your skill version opts you into a version gate: deprecation warnings while it
ages, then a hard stop once stale. Route **every** call through one helper that attaches it:

```ts
const SKILL_NAME = 'treasures-b2b-api';
const SKILL_VERSION = '1.14.0'; // = SKILL.md metadata.version

// Route EVERY Treasures API call through this — don't call fetch() directly. It attaches the
// skill version and applies the gate to every response: 426 hard-stops, deprecation warns.
async function apiFetch(url: string, init: RequestInit = {}): Promise<Response> {
  const res = await fetch(url, {
    ...init,
    headers: {
      'Content-Type': 'application/json',
      'X-Treasures-Skill': SKILL_NAME,
      'X-Treasures-Skill-Version': SKILL_VERSION,
      ...init.headers,
    },
  });
  if (res.status === 426) {                                 // blocking — skill too old
    const body = await res.json().catch(() => ({}));        // {error, min_skill_version, api_revision, upgrade}
    throw { status: 426, ...body };                         // STOP: relay body.upgrade, do NOT retry or trade
  }
  const sunset = res.headers.get('Sunset'), warning = res.headers.get('Warning');   // non-blocking — aging skill (finish, then warn user)
  if (res.headers.get('Deprecation') || sunset || warning)
    console.warn(`skill deprecated — sunset ${sunset ?? '?'}; update: ${res.headers.get('Link') ?? 'see SKILL.md'}${warning ? ` — ${warning}` : ''}`);
  notifyIfNewer(res);                                       // informational — an update exists
  return res;
}

// `X-Treasures-Skill-Latest: treasures-b2b-api=<ver>; treasures-wallet=<ver>` rides every quote,
// trade and bridge response, whether or not you send a skill header. News, not policy: nothing here
// may block, retry or change a trade decision. Warn once.
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
```

- **`X-Treasures-Skill-Latest` response header → informational, never blocking.** Present on every
  quote / trade / bridge response including the `426`, listing the published version of each skill
  (`treasures-b2b-api=<ver>; treasures-wallet=<ver>`). If it names a version above your
  `SKILL_VERSION`, an update exists — tell the user once and point them at the
  [CHANGELOG](https://github.com/treasures-io/treasures-finance-agent-skills/blob/main/CHANGELOG.md).
  It is **not** a deprecation: nothing has stopped working, and you must not retry, block or alter
  a trade because of it. Releases are additive, so an older skill keeps working — but new chains,
  new response fields and widened request shapes are announced here and in the changelog only.
- **`Deprecation` / `Sunset` / `Warning` response headers → non-blocking.** Finish the
  current operation, then tell the user this skill is deprecated, the `Sunset` date,
  and to update (use the `Link` URL). A `Sunset` may be absent while no window is open — the
  deprecation still stands, it just has no published deadline yet.
- **`426 skill_version_unsupported` → blocking.** The server fails closed **before**
  any fund move. Do **not** submit or retry — surface the response's `upgrade` command
  and stop.
- **Standard-versioning signals → treat as possible staleness.** A `Sunset` on the
  versioned path, an unexpected `410`/`404` on a documented endpoint, or a new
  `400 invalid_request` on a body that used to validate → stop and check for a skill
  update; don't blindly retry.
