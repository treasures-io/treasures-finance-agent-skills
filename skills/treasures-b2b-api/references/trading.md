# Trading — quote, submit, status

Load when quoting/submitting trades or polling status. Pre-flight traps (approvals, gas, slippage, TTLs) and the buy-one/sell-all rule are summarized in [`../SKILL.md`](../SKILL.md); this file is the schemas, signing code, and status contracts.

> **`tx_hash` vs `order_hash` (defined once, applies everywhere).** Exactly one is non-null at a time. **Solana legs** and **Eth `completed`** legs: `tx_hash` carries the on-chain hash, `order_hash = null`. **Eth in-flight**: `tx_hash = null`, the gasless EVM order hash sits on `order_hash`; once it fills, `tx_hash` flips to the settlement hash and `order_hash → null`. External `/trades` rows: both null.

<a id="first-time-approvals"></a>
## First-time approvals (before first Eth trade or bridge)

Ethereum requires ERC-20 `approve()` allowances **before** any transfer; the API does not broadcast them for you. Set once per `(token, spender)` from the user's wallet. Solana has no allowances — skip on `sol` legs. (Spender addresses + the summary table are in [`../SKILL.md`](../SKILL.md#before-your-first-call--pre-flight-traps); the `eth → sol` bridge spender comes from `signable_payload.approval_spender` — see [`bridging.md`](bridging.md#approval-spender).)

Set `allowance = max-uint256` once, then never re-approve unless the user revokes:

```ts
import { createWalletClient, http, maxUint256 } from "viem";
import { privateKeyToAccount } from "viem/accounts";
import { mainnet } from "viem/chains";

const ERC20_APPROVE_ABI = [{
  type: "function", name: "approve", stateMutability: "nonpayable",
  inputs: [{ name: "spender", type: "address" }, { name: "amount", type: "uint256" }],
  outputs: [{ type: "bool" }],
}] as const;

const walletClient = createWalletClient({
  account: privateKeyToAccount(ETH_PRIVATE_KEY), chain: mainnet, transport: http(ETH_RPC_URL),
});

// writeContract auto-fetches the nonce (unlike the bridge tx — see bridging.md).
const approveErc20Max = (token: `0x${string}`, spender: `0x${string}`) =>
  walletClient.writeContract({ address: token, abi: ERC20_APPROVE_ABI,
    functionName: "approve", args: [spender, maxUint256] });
```

**If you skip it:** Eth buy/sell of an unapproved token → `/trade/submit` returns 200 with `status: "broadcast_failed"`, `error_code: "not_enough_balance_or_allowance"` (gasless, so the wallet pays no gas) — re-quote after approving. `eth → sol` bridge without USDC allowance → silent forever-pending (see [`bridging.md`](bridging.md#missing-allowance-trap)).

**Eth sells can span both protocols.** A sell greedy-fills across both protocols, each a **distinct token address**, and the *quote* (not you) decides which fill. Quote TTLs (~25–55s) are too short to land an on-chain `approve()` between quote and submit — so **pre-approve the stock token for every protocol the ticker lists** (read both `eth_address`es from `/stocks/tickers`) before your first Eth sell of that ticker. Approving only one protocol's token risks `not_enough_balance_or_allowance` on the other leg.

## Shares vs tokens

Quantity responses emit both raw `tokens` and equity-exposure `shares` side-by-side (`estimated_output_*`, `*_consumed`, `filled_*`, and `tokens`/`shares` on portfolio + trades). Use `shares` for "how much equity", `tokens` for "on-chain units". The server converts; you never need to. The per-protocol multiplier (Ondo ~0.5, xStocks 0.5–10 after corporate actions) is invisible to the contract — read `share_multiplier` from `/stocks/tickers` only if you need exact-amount `approve()` instead of max-uint256.

## `POST /quote/buy`

USDC → shares. Returns up to 2 quotes (one per chain), sorted by USDC-per-share (best first).

```jsonc
// request
{
  "ticker": "AAPL",
  "amount_usdc": "100",                  // exactIn, string, 6 decimals max
  "chain": "sol" | "eth" | null,         // null = best per chain
  "protocol": "ondo" | "xstocks" | null, // null = either
  "max_slippage_bps": 50,                // 10 ≤ value ≤ 5000
  "sol_wallet": "...", "eth_wallet": "0x...",   // at least one
  "ownership_proof": { /* see auth.md */ }
}

// response 200
{
  "quote_id": "qte_<nanoid>",
  "side": "buy",
  "ticker": "AAPL",
  "expires_at": 1730000040,              // unix seconds
  "tradfi_reference": { "price_usd": "232.45", "market_status": "open", "as_of": 1730000000 } | null,  // market_status: opaque string — don't branch on specific values
  "quotes": [
    {
      "quote_index": 0,                  // stable handle referenced at /trade/submit
      "chain": "sol",
      "protocol": "ondo",
      "price_usdc_per_share": "234.20",
      "estimated_output_shares": "0.4988",
      "estimated_output_tokens": "0.4988",
      "cost_breakdown_bps": {
        "treasures_fee_bps": 30,
        "dex_swap_fee_bps": 8,
        "estimated_slippage_bps": 12,
        "slippage_vs_tradfi_bps": 75     // signed; positive = unfavorable. KEY IS ABSENT
                                         // (not null) when tradfi_reference is null.
      },
      "signable_payloads": [{ "type": "solana_versioned_tx", "tx_base64": "..." }]
    },
    {
      "quote_index": 1, "chain": "eth", "protocol": "xstocks",
      "price_usdc_per_share": "235.10", "estimated_output_shares": "0.4969", "estimated_output_tokens": "0.4969",
      "cost_breakdown_bps": { "treasures_fee_bps": 30, "dex_swap_fee_bps": 5, "estimated_slippage_bps": 18, "slippage_vs_tradfi_bps": 114 },
      // Eth legs return EIP-712 typed data for the gasless order, NOT a raw tx:
      "signable_payloads": [{ "type": "evm_eip712_typed_data", "typed_data": { /* domain, types, primaryType, message */ } }]
    }
  ]
}
```

**Buy submission rule.** Each `quote_index` is a **mutually-exclusive alternative** — sign + submit one index only. >1 leg → `400 quote_index_mismatch`. No chain preference → just take `quotes[0]` (best rate) or re-quote with the `chain` you want.

### Signable payload shapes

A leg's `chain` determines the variant:

| `chain` | `type` | Payload field | Sign / submit as |
| --- | --- | --- | --- |
| `sol` | `solana_versioned_tx` | `tx_base64` — unsigned `VersionedTransaction` | sign with Solana key → `{ type: "solana_versioned_tx", signed_tx_base64 }` |
| `eth` | `evm_eip712_typed_data` | `typed_data` — full EIP-712 object | sign typed data (`eth_signTypedData_v4`) → `{ type: "evm_eip712_signature", signature: "0x..." }` |

> The quote EVM payload (`evm_eip712_typed_data`, off-chain order signature) is **different** from the bridge EVM payload (`evm_eip1559_tx`, a signed-and-broadcast on-chain tx — see [`bridging.md`](bridging.md)).

```ts
// Solana: deserialize, sign in place, re-serialize.
import { Keypair, VersionedTransaction } from "@solana/web3.js";
function signSol(payload: { tx_base64: string }, keypair: Keypair) {
  const tx = VersionedTransaction.deserialize(Buffer.from(payload.tx_base64, "base64"));
  tx.sign([keypair]);
  return { type: "solana_versioned_tx" as const,
    signed_tx_base64: Buffer.from(tx.serialize()).toString("base64") };
}

// Ethereum: EIP-712 over the order typed data. Strip EIP712Domain — viem auto-injects it.
import { privateKeyToAccount } from "viem/accounts";
async function signEvm(payload: { typed_data: any }, account: ReturnType<typeof privateKeyToAccount>) {
  const { domain, types, primaryType, message } = payload.typed_data;
  const { EIP712Domain: _omit, ...typesNoDomain } = types;
  const signature = await account.signTypedData({ domain, types: typesNoDomain, primaryType, message });
  return { type: "evm_eip712_signature" as const, signature };
}
```

> **Embedded / managed wallets:** the EVM path works unchanged — `signTypedData` is universal across embedded EVM wallets. For **Solana** you won't hold a raw `Keypair`: pass the deserialized `VersionedTransaction` to your wallet's **transaction signer** — **not** `signMessage` (that's only for the ownership proof). Most managed signers (incl. Privy `signTransaction`) return the **signed transaction** — re-serialize it to `signed_tx_base64`. The `addSignature` path — `tx.addSignature(new PublicKey(walletPubkey), sigBytes)` before `serialize()` — is the fallback **only** for signers that hand back a raw 64-byte signature instead of a signed tx. Sign **without** broadcasting; `/trade/submit` does the broadcast. The wallet handle is the same one you use in [`auth.md`](auth.md#embedded-wallets).

Both output shapes drop straight into `/trade/submit` as `signed[i].signed_payloads`.

## `POST /quote/sell`

Shares → USDC. Server reads holdings across both chains × both protocols, then greedy-fills `amount_shares` from the best-priced pools. **Same response shape as `/quote/buy`**, with these differences:

- request: `amount_shares` (18 decimals max) replaces `amount_usdc`.
- each quote carries `shares_consumed` + `tokens_consumed` + `estimated_output_usdc` instead of `estimated_output_*`.
- response adds `"totals": { "shares_total": "0.5", "usdc_total_estimated": "117.10" }`.
- `signable_payloads` shapes are identical (sol → `solana_versioned_tx`, eth → `evm_eip712_typed_data`).
- all other buy fields carry over unchanged: `quote_index`, `chain`, `protocol`, `price_usdc_per_share`, `cost_breakdown_bps`, `tradfi_reference`.

**Sell submission rule.** Sell legs are **additive** — sign + submit **every** `quote_index` returned. Subset → `400 incomplete_submit`. To sell from one chain only, re-quote with a `chain` filter.

## `POST /trade/submit`

Atomic submission of signed payloads. No `ownership_proof` — the signed payloads are the auth.

```jsonc
// BUY — exactly ONE quote_index (>1 → 400 quote_index_mismatch)
{ "quote_id": "qte_...", "signed": [
  { "quote_index": 0, "signed_payloads": [{ "type": "solana_versioned_tx", "signed_tx_base64": "..." }] }
] }

// SELL — ALL quote_index values (subset → 400 incomplete_submit)
{ "quote_id": "qte_...", "signed": [
  { "quote_index": 0, "signed_payloads": [{ "type": "solana_versioned_tx", "signed_tx_base64": "..." }] },
  { "quote_index": 1, "signed_payloads": [{ "type": "evm_eip712_signature", "signature": "0x..." }] }
] }

// response 200
{
  "results": [
    { "quote_index": 0, "trade_id": "trd_...", "status": "completed", "tx_hash": "<sol sig>", "order_hash": null, "error_code": null },
    { "quote_index": 1, "trade_id": "trd_...", "status": "broadcast",  "tx_hash": null, "order_hash": "<eth order hash>", "error_code": null }
  ],
  "failed_legs": []   // populated if a sibling leg's internal invariant broke;
                      // each entry: { quote_index, error_code: "internal_error" }
}
```

### Submit-time leg statuses → how they map to `/quote/{id}/status`

| Chain | Submit `status` | Meaning | `/status` leg `status` | Agent action |
| --- | --- | --- | --- | --- |
| sol | `completed` | confirmed on-chain | `completed` (terminal) | none |
| sol | `failed` | upstream rejected (bad sig, expired, or declined) | `failed` (terminal) | re-quote |
| sol | `broadcast_unknown` | lost upstream response — may or may not have landed | `pending` | **DO NOT retry.** Poll status |
| eth | `broadcast` | gasless order accepted and live | `pending` | poll until fill/fail |
| eth | `broadcast_unknown` | submit transport failed but order may be live | `pending` | poll; upstream resolves either way |
| eth | `broadcast_failed` | upstream 4xx reject (bad allowance/sig) | `broadcast_failed` (terminal) | re-quote after fixing root cause |

`/status` legs **never** carry `broadcast`, `broadcast_unknown`, or `in_progress` (the latter is an `aggregate_status` value only). Only `broadcast_failed` survives untouched as a terminal failure.

- **Idempotency.** `(quote_id, quote_index)` is the dedup key, cached **24h**. Same `(quote_id, quote_index)` + same signed payload → cached prior result (200, same `tx_hash`). Different payload for an already-broadcast leg → `409 leg_already_submitted` with the original `tx_hash`. Safe to retry on network glitch.
- **`error_code` at submit** (when a leg is `failed`/`broadcast_failed`): same per-leg enum as `/status` below. `failed_legs[]` uses `internal_error` only.
- **Atomicity.** Validation (shape, signer recovery, payload-type) is all-or-nothing pre-broadcast → any leg fails to validate, none broadcast (`400 incomplete_submit` / specific error). After validation, broadcast is best-effort per leg: legs that landed stay irreversibly on-chain even if later ones fail. Re-quote any `failed_legs[]` indices on a fresh `quote_id`.

## `GET /quote/{quote_id}/status`

Aggregate + per-leg view. Poll ≤ 1 Hz (cached 10s server-side while any leg is `in_progress`).

```jsonc
{
  "quote_id": "qte_...",
  "aggregate_status": "in_progress" | "completed" | "partial_failed" | "all_failed",
  "is_cached": false,
  "legs": [{
    "quote_index": 0, "trade_id": "trd_...", "ticker": "AAPL",
    "chain": "sol" | "eth", "protocol": "ondo" | "xstocks", "side": "buy" | "sell",
    "tx_hash": "...", "order_hash": null,   // see tx_hash/order_hash split at top of file
    "status": "pending" | "completed" | "failed" | "broadcast_failed",
    "error_code": null,
    "filled_shares": "0.3", "filled_tokens": "0.3", "filled_usdc": "70.26",  // "0" until terminal
    "last_synced_at": 1730000050
  }]
}
```

Per-leg `error_code` (when `status` is `failed`/`broadcast_failed`):

| Code | Meaning | Action |
| --- | --- | --- |
| `invalid_signature` | Payload malformed or doesn't recover to the quote-bound wallet | re-sign with the correct wallet |
| `quote_expired` | Cached quote / order / blockhash no longer valid | re-quote |
| `not_enough_balance_or_allowance` | (Eth) wallet lacks token balance or allowance to settlement | set allowance ([approvals](#first-time-approvals)) or fund, then re-quote |
| `provider_error` | Other provider rejection | re-quote; retry on persistent failure |
| `status_check_exhausted` | Eth order sat > 1h unresolved (operator handoff) | re-quote |

> `internal_error` is `/trade/submit`-only (in `failed_legs[]`); never appears here. Re-quote those indices on a fresh `quote_id`.

Stop polling once `aggregate_status` ∈ `{completed, partial_failed, all_failed}`:

| Condition | `aggregate_status` |
| --- | --- |
| Any leg still `pending` | `in_progress` |
| All terminal **and all** `completed` | `completed` |
| All terminal, ≥1 `completed` + ≥1 failed | `partial_failed` |
| All terminal, **none** `completed` | `all_failed` |

`completed` is reachable **only** when every leg is `completed` — one `broadcast_failed` always demotes to `partial_failed`/`all_failed`. Don't trust aggregate alone for "did all legs land"; check per-leg too.
