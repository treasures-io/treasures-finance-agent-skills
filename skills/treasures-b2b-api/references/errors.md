# Errors, rate limits, simulation, gotchas

Load when handling an error, hitting a rate limit, or simulating a tx before broadcast.

> **Backoff policy B** (referenced below): exponential — 1s start, ×2 per attempt, ±20% jitter, **cap at 5 attempts (~30s total)**. Still failing → surface to the user; do **not** loop indefinitely.

## HTTP error matrix

Top-level HTTP `error` codes. (Per-leg `error_code` values live inside `/quote/{id}/status` legs — a **separate namespace** documented in [`trading.md`](trading.md#get-quotequote_idstatus) — and never appear as top-level HTTP errors. Ownership-proof 401s are in [`auth.md`](auth.md#auth-error-codes).)

| HTTP | `error` | Meaning | Retry |
| --- | --- | --- | --- |
| 400 | `invalid_request` | Schema/Zod validation failed | Permanent — fix payload |
| 400 | `invalid_slippage` | `max_slippage_bps` outside bounds (`[10,5000]` quote, `[10,500]` bridge — response echoes cap) | Permanent — adjust |
| 400 | `incomplete_submit` | Sell missing legs, or `signed[]` empty | Permanent — sell all legs; buy submits exactly 1 |
| 400 | `quote_index_mismatch` | Buy submitted >1 leg, unknown index, or duplicate | Permanent — re-quote with chain filter, or dedupe |
| 400 | `same_chain_bridge` | Bridge `from_chain == to_chain` | Permanent |
| 400 | `unsupported_route` | Bridge route not enabled | Permanent |
| 403 | `address_blocked` | A wallet is tied to a sanctioned entity | Permanent — do not retry that wallet |
| 404 | `quote_not_found` | `quote_id` unknown or expired | Permanent — re-quote |
| 404 | `bridge_quote_not_found` | `bridge_quote_id` unknown | Permanent — re-quote |
| 409 | `leg_already_submitted` | `(quote_id, quote_index)` already broadcast with a different payload | Permanent — read returned `tx_hash` |
| 410 | `quote_stale` | `expires_at` passed | Permanent — re-quote |
| 415 | *(`error` is a message, not a code)* | Request body sent with a **non-JSON** `Content-Type` — a *missing* header is accepted (body is parsed as JSON regardless); a *wrong* one (e.g. `text/plain`) is rejected before schema validation. Body: `{ "error": "Unsupported Media Type; expected application/json" }` | Permanent — set `Content-Type: application/json` |
| 422 | `holdings_insufficient` | Sell shares exceed on-chain holdings | Permanent — reduce `amount_shares` |
| 422 | `holdings_unknown` | On-chain balance read failed (RPC/indexer blip) | Transient — **policy B**, then re-quote with a `chain`/`protocol` filter to skip the unreadable cell. Failing across both chains = degradation; surface to user |
| 422 | `insufficient_base_balance` | (`/quote/*`, chain `robinhood` only) executable swap pre-build failed — the wallet holds too little of the **input** asset on 4663 to fund the trade: a **buy** needs USDG; a **sell** needs the Stock Token (if on-chain balance drifted below the sized exit) | Permanent — fund the input asset on Robinhood Chain (4663), then re-quote |
| 422 | `insufficient_liquidity` | Bridge route has no quote at this size | Transient — retry / smaller size |
| 422 | `no_routes` | (`/quote/*` only) all candidate legs unavailable; body may carry `reason: "slippage_exceeded" \| "amount_too_small"` | Permanent — act per `reason`, or relax slippage / change chain / protocol |
| 426 | `skill_version_unsupported` | `X-Treasures-Skill-Version` below the server's `X-Min-Skill-Version` floor; fails closed **before** any fund move. **Inert today** (floor = `1.0.0`); activates on a future breaking change. Body: `{error, min_skill_version, api_revision, upgrade}` | Permanent — STOP, do not trade/retry; relay `upgrade` to the user. See [skill-version-compatibility](https://github.com/treasures-io/treasures-finance-agent-skills/blob/main/docs/skill-version-compatibility.md) |
| 429 | `Too many requests` | Per-IP or per-(IP, wallet) limit. Body is the literal string `Too many requests` (not an enum) | Transient — honor `Retry-After` |
| 502 | `provider_unavailable` | Upstream 5xx/429/network (quote venue, bridge, price feed, or RPC) | Transient — **policy B** |
| 503 | `screening_unavailable` | Sanctions screening couldn't complete; request fails closed | Transient — **policy B** |

## Rate limits

Per-IP and per-(IP, wallet) buckets, 60s windows. **The numbers below are conservative floors** — the server's real per-endpoint ceilings are equal or higher — **but every caller is additionally capped by a global 300 requests/min per IP across _all_ endpoints** (see below), which is the binding limit for high-frequency polling. Pace to the 300/min aggregate, not the per-endpoint row:

| Endpoint group | Per-IP / min | Per-wallet / min |
| --- | --- | --- |
| `/quote/buy` | 60 | 60 |
| `/quote/sell` | 20 | 20 |
| `/quote/{id}/status` | 600 | — |
| `/trade/submit` | 60 (IP only) | — |
| `/bridge/quote` | 30 | 60 |
| `/bridge/{id}/status` | 180 | — |
| `/stocks/tickers`, `/stocks/prices` | 60 each (separate buckets) | — |
| `/portfolio`, `/trades` | 60 | 60 |

**Global ceiling.** A blanket **300 requests / minute per IP** applies across *all* endpoints combined (every route except health), on top of the per-endpoint buckets above. High-frequency polling (`/status` at 1 Hz across many quotes, plus `/portfolio` / `/trades`) is bounded by this aggregate, not the per-endpoint number — keep your **total** request rate under 300/min/IP, or spread across IPs.

429 returns `{ "error": "Too many requests" }` + a `Retry-After` header in **delta-seconds** (integer, e.g. `12` — never an HTTP-date). Sleep exactly that long; don't retry sooner.

**Mid-flow 429 on a multi-leg sell.** `/trade/submit` is IP-only rate-limited (60/min). If a 429 hits between signing and submitting, the signed payloads stay valid until `quote.expires_at` — wait out `Retry-After` and submit unchanged. If `Retry-After` would push you past `expires_at − 5s`, re-quote on a fresh `quote_id`.

## Simulate signed txs before broadcast (recommended)

Optional but strongly advised. Dry-run every signed tx against the live RPC first — a free way to catch reverts, slippage violations, missing allowances, and stale blockhashes before burning gas. Inspect logs / return data to confirm recipient, token, amount, and program/contract match the quote — a passing sim that moves the wrong funds is still a fail.

| Payload | From | Simulate with |
| --- | --- | --- |
| `solana_versioned_tx` (signed) | `/quote/*`, `/bridge/quote` | `connection.simulateTransaction(tx)` (@solana/web3.js) |
| `evm_eip1559_tx` (signed, real nonce) | `/bridge/quote` | `publicClient.call({...})` (viem `eth_call`) |
| `evm_legacy_tx` (signed, real nonce) | `/quote/*` chain `robinhood` | same — `publicClient.call(...)` on a client pointed at your **4663** RPC (define a custom viem chain for 4663, not `mainnet`; sim both the approve and the swap) |
| ERC-20 `approve()` you built | first-time approvals | same — `publicClient.call(...)` against the token |

> `evm_eip712_typed_data` from `/quote/*` is an **off-chain order signature, not an on-chain tx** — nothing to simulate. Trust the quote and poll `/quote/{id}/status`.

```ts
// Solana
import { Connection, VersionedTransaction } from "@solana/web3.js";
const connection = new Connection(SOLANA_RPC_URL, "confirmed");
async function simulateSol(signedTxBase64: string) {
  const tx = VersionedTransaction.deserialize(Buffer.from(signedTxBase64, "base64"));
  // sigVerify:false skips re-checking the sig; replaceRecentBlockhash dodges expired-blockhash false negatives.
  const { value } = await connection.simulateTransaction(tx, { sigVerify: false, replaceRecentBlockhash: true });
  if (value.err) throw new Error(`sim failed: ${JSON.stringify(value.err)} — ${value.logs?.join(" | ")}`);
}

// EVM — eth_call dry-runs at head; throws the decoded revert reason on failure.
import { parseTransaction, createPublicClient, http } from "viem";
import { mainnet } from "viem/chains";
const publicClient = createPublicClient({ chain: mainnet, transport: http(ETH_RPC_URL) });
async function simulateEvm(signedTxHex: `0x${string}`, from: `0x${string}`) {
  const p = parseTransaction(signedTxHex);
  await publicClient.call({ account: from, to: p.to!, data: p.data, value: p.value });
}
```

A passing simulation is **not a guarantee** the broadcast lands — chain state shifts (front-running, MEV, blockhash expiry, gas spikes, allowance drift). Strong pre-flight check, not a contract.

<a id="gas"></a>
## Native gas

Treasures does not subsidize gas. Any operation your key broadcasts needs native balance on that chain.

| Operation | Pays gas in | Notes |
| --- | --- | --- |
| Solana buy/sell via `/trade/submit` | **SOL** | user pays network + tiny priority fee; ~0.001 SOL/tx |
| Eth ERC-20 `approve()` | **ETH** | one-time per (token, spender); mainnet gas, no subsidy |
| Eth `/bridge/quote` broadcast | **ETH** | you submit the deposit tx; reverts (e.g. missing allowance) also cost gas |
| Eth buy/sell | **gasless** | the execution venue pays settlement gas; ETH needed only for the prior approval |
| Solana `/bridge/quote` broadcast | **SOL** | you broadcast the signed tx; user pays network fee |
| Robinhood (`chain:"robinhood"`) approve + swap | **ETH on 4663** | you broadcast both txs yourself on chain 4663; native ETH *there* — a separate float from mainnet ETH |

Rough floats (mainnet, conservative — tune to live gas): Solana ≥ **0.02 SOL** (dozens of trades + a bridge), Ethereum ≥ **0.01 ETH** (a handful of approvals + one bridge; raise during congestion), plus — if you use the Robinhood venue — a **separate ETH float on chain 4663** (covers the approve + swap you broadcast there). Zero balance → Solana: leg `failed`/`provider_error`; Eth / 4663: wallet RPC rejects (`insufficient funds for gas`), no on-chain effect — fund, then proceed. **This API does not expose native balances** — `/portfolio` returns only USDC + token positions, so check the native float yourself via your own RPC (`getBalance` on Solana, `eth_getBalance` on Ethereum, `eth_getBalance` on your 4663 RPC for Robinhood) before broadcasting.

## Gotchas & contract notes

- **All request bodies are `.strict()`.** Unknown/misspelled fields → `400 invalid_request`. Don't add unknown `null` placeholders, leftover B2C fields (`user_id`), or future-reserved fields. (Fields documented as nullable — e.g. `chain`/`protocol`, where `null` means "best/either" — do accept `null`; the rule targets *undocumented* fields only.) Applies to the POST bodies of `/quote/buy`, `/quote/sell`, `/trade/submit`, `/bridge/quote` — **and** to GET query strings on `/portfolio`, `/trades`, `/stocks/tickers`, `/stocks/prices` (an unknown query param → `400 invalid_request`).
- **Decimals.** USDC = 6, shares = up to 18. Always send/parse as strings; never via JS `number` (server uses Big.js / Postgres `numeric` end-to-end).
- **Tickers.** Canonical IEX/Nasdaq dot-notation everywhere: `BRK.A`, `BF.B` — never `BRK-A`/`BF-B`. Quote-body regex `/^[A-Z][A-Z0-9.]{0,9}$/` (uppercase, starts with a letter, hyphen rejected). The laxer `/stocks/prices` regex `/^[A-Z0-9.-]{1,10}$/i` works only because that endpoint normalizes to dot-notation first — always send dot-notation; don't depend on normalization elsewhere.
- **`tradfi_reference` may be `null`** (tradfi-feed outage) — never block on it; use `quotes[i].price_usdc_per_share` for execution decisions.
- **Privacy.** Treasures hides which upstream provider executed a leg. Don't expect or parse provider-specific fields — they aren't there.
- **`source: "external"` trades have null `usdc_amount`** — token-balance drift from reconciliation; `tokens` **and** `shares` are both signed (positive = inflow, negative = outflow), and `side` is always `null` (the reconciler never classifies drift as buy/sell).
- **`expires_at` is unix seconds, not millis.**
- **The `chain` field is functional, not cosmetic** — each chain has a different signing scheme; sign with the wallet whose key matches the leg's `chain`.
- **No `/quote/buy` USDC balance pre-check.** Sell does pre-check share holdings.
- **xStocks on Ethereum rebases internally** — `balanceOf` fluctuates with dividends/splits. Trust the `/portfolio` `shares` field, not raw `balanceOf`.
- **Sanctions screening** runs on the fund-moving endpoints (`/quote/buy`, `/quote/sell`, `/trade/submit`, `/bridge/quote`) against both wallets: sanctioned tie → `403 address_blocked` (permanent), screening can't complete → `503 screening_unavailable` (transient, policy B). Read-only endpoints are not screened.
