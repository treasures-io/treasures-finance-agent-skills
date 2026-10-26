# Bridging USDC across chains

Load when moving USDC between chains. Treasures fronts an intent-based bridge: it issues the quote and a privacy-safe status proxy. Two ways to run a quote, chosen by `delivery` on the quote:

- **`self_broadcast` (default):** you sign the bridge tx and **broadcast it yourself**. Unchanged from earlier skill versions.
- **`submit` (opt-in):** you sign and hand the result to `POST /bridge/{id}/submit`; nothing you hold can be broadcast by anyone else. From an EVM chain you sign **one typed-data message and need no ETH** (the upstream executes the deposit and takes its gas from the amount); from Solana you sign the deposit tx and Treasures broadcasts it. See [Treasures-submitted bridges](#submit).

> **Any two distinct chains:** `sol`, `eth`, `robinhood`, `base`, `arbitrum`, `bnb`. Each side settles in that chain's base currency: USDC everywhere except **USDG on `robinhood`** (so that bridge is also a USDC↔USDG swap) and 18-dp Binance-Peg USDC on `bnb`. `bnb` answers `400 unsupported_route` on a deployment without BNB Chain support. With `delivery: "submit"`, the origin must be `sol`, `eth`, `base` or `arbitrum` for now (others: `400 unsupported_route`).

## `POST /bridge/quote`

```jsonc
// request
{
  "from_chain": "sol" | "eth" | "robinhood" | "base" | "arbitrum" | "bnb",
  "to_chain":   "sol" | "eth" | "robinhood" | "base" | "arbitrum" | "bnb",  // must differ (else 400 same_chain_bridge)
  "amount_usdc": "100",                  // capped at 100,000 USDC; exceeding → 400 invalid_request
  "max_slippage_bps": 50,                // 10 ≤ value ≤ 500 (tighter than /quote/*)
  "eth_wallet": "0x...",                 // always required: your address on every EVM side
  "sol_wallet": "...",                   // required only when either side is "sol"
  "ownership_proof": { "eth_signature": "0x...", "sol_signature": "<base64 ed25519>", "issued_at": 1730000000 },
                                         // sol_signature only when sol_wallet is sent
  "delivery": "self_broadcast" | "submit" // optional, default "self_broadcast"
}

// response 200
{
  "bridge_quote_id": "brq_...",
  "expires_at": 1730000060,
  "amount_in_usdc": "100",
  "estimated_amount_out_usdc": "99.55",
  "max_slippage_bps": 50,                  // governs swap_slippage_bps only; gas + bridge fees sit on top
  "cost_breakdown_bps": {                  // every field carries _bps
    "swap_slippage_bps": 1,                // DEX fees + actual slippage + routing rebalancing
    "gas_fee_bps": 0,                      // origin-chain gas + flat execution fee (eats small notionals)
    "bridge_fee_bps": 45,
    "total_bps": 46
  },
  "cost_breakdown_usdc": {                  // same components in USDC; every field carries _usdc
    "swap_slippage_usdc": "0.010000", "gas_fee_usdc": "0.001000",
    "bridge_fee_usdc": "0.450000", "total_usdc": "0.461000"
  },
  "signable_payload": /* one of: */
    { "type": "evm_eip1559_tx",              // EVM origin, self_broadcast
      "from": "0x...",                       // expected signer (matches eth_wallet)
      "tx_base64": "...",                    // serialized unsigned tx (nonce=0 — see trap below)
      "tx_hex": "0x...",                     // same tx, hex
      "approval_spender": "0x..." | null }   // see below
    /* OR */
    { "type": "solana_versioned_tx",         // sol origin, either delivery
      "tx_base64": "...",
      "blockhash": "...",
      "last_valid_block_height": 123456789 } // re-quote past this height
    /* OR */
    { "type": "evm_permit_typed_data", ... } // EVM origin, delivery: "submit" — see below
}
```

<a id="approval-spender"></a>
**`approval_spender` (EVM-only, `string | null`).** Non-null when the calldata requires an allowance on the source chain's base token — the upstream pull-pattern deposit, i.e. the normal flow from an EVM chain. **Set the allowance to this address before broadcasting.** It mirrors the tx's `to`, so you never decode `tx_hex`. `null` when the calldata is self-spend (direct `transfer`/`approve`/value-only) and no external allowance is needed → skip the approval step.

**Workflow (`self_broadcast`):** (1) from an EVM chain with non-null `approval_spender`, approve the base token to it ([approvals](trading.md#first-time-approvals)). (2) sign the payload with the `from_chain` key. (3) **broadcast to chain yourself.** (4) poll `/bridge/{id}/status` until `completed` or `failed`. (5) for a follow-on trade, quote it with the **actual landed** amount from `/portfolio` — you receive `estimated_amount_out_usdc`, which is **less** than you sent, not the input amount.

<a id="evm-sign-broadcast"></a>
## Sign + broadcast — EVM (`evm_eip1559_tx`)

> ⚠️ **The serialized tx has `nonce=0`.** Treasures can't know your wallet's nonce, so it emits `nonce=0`. You MUST fetch the real nonce and inject it before signing, or broadcast is rejected ("nonce too low") on any wallet with history. **Most common EVM-bridge footgun.** Everything else — gas, `maxFeePerGas`/`maxPriorityFeePerGas`, `chainId` — is already populated in the serialized tx (`parseTransaction` + spread preserves it), so the **nonce is the only field you change**.

```ts
import { parseTransaction, createPublicClient, http } from "viem";
import { privateKeyToAccount } from "viem/accounts";
import { mainnet } from "viem/chains"; // the from_chain's viem chain

const publicClient = createPublicClient({ chain: mainnet, transport: http(FROM_CHAIN_RPC_URL) });
const account = privateKeyToAccount(ETH_PRIVATE_KEY);

async function signAndBroadcastEvmBridge(payload: { from: `0x${string}`; tx_hex: `0x${string}` }) {
  const parsed = parseTransaction(payload.tx_hex);
  // blockTag 'pending' counts any in-flight tx — else back-to-back broadcasts collide on nonce.
  const nonce = await publicClient.getTransactionCount({ address: payload.from, blockTag: "pending" });
  const signedHex = await account.signTransaction({ ...parsed, nonce });
  return publicClient.sendRawTransaction({ serializedTransaction: signedHex }); // cache this hash — see status note
}
```

## Sign + broadcast — Solana (`solana_versioned_tx`)

Signing is identical to the trade leg (`signSol` in [`trading.md`](trading.md)) — but here **you broadcast directly** instead of returning to `/trade/submit`:

```ts
import { Connection, Keypair, VersionedTransaction } from "@solana/web3.js";
const connection = new Connection(SOLANA_RPC_URL, "confirmed");

async function signAndBroadcastSolanaBridge(payload: { tx_base64: string }, keypair: Keypair) {
  const tx = VersionedTransaction.deserialize(Buffer.from(payload.tx_base64, "base64"));
  tx.sign([keypair]);
  return connection.sendRawTransaction(tx.serialize(), { maxRetries: 3 }); // cache this hash
}
```

If signing/broadcast slips past `last_valid_block_height`, the network drops the tx — re-quote on a fresh `bridge_quote_id`.

<a id="submit"></a>
## Treasures-submitted bridges (`delivery: "submit"`)

Quote with `"delivery": "submit"`, sign what it returns, and send the result to `POST /bridge/{bridge_quote_id}/submit` before `expires_at`. A `submit` quote is executable **only** through `/submit`: there is no transaction for you (or anyone) to broadcast twice.

**EVM origin — `evm_permit_typed_data`.** You sign typed data, not a transaction: no nonce, no gas, **no ETH needed** on the origin chain. The upstream executes the deposit and takes its gas from the amount (already inside `cost_breakdown_usdc.gas_fee_usdc` and `estimated_amount_out_usdc`).

```jsonc
"signable_payload": {
  "type": "evm_permit_typed_data",
  "chain_id": 8453,
  "approval_tx": null,           // bnb only (not yet offered): a one-time approve your wallet sends first
  "signatures": [                // sign each, in this order; usually one, sometimes two
    { "typed_data": { "domain": { ... }, "types": { "EIP712Domain": [ ... ], ... },
                      "primaryType": "ReceiveWithAuthorization", "message": { ... } } }
  ]
}
```

Each `typed_data` is exactly what `eth_signTypedData_v4` takes (`types` includes `EIP712Domain`). It authorizes moving `amount_in_usdc` of your base token, once, to the bridge's settlement contract, for about ten minutes. Sign it **as given**; the domain or type names may name the upstream settlement contract, which is expected.

```ts
import { privateKeyToAccount } from "viem/accounts";
const account = privateKeyToAccount(ETH_PRIVATE_KEY);

async function signPermitBridge(bridgeQuoteId: string, payload: { signatures: { typed_data: any }[] }) {
  const signatures = [];
  for (const { typed_data } of payload.signatures) {
    const { EIP712Domain, ...types } = typed_data.types; // viem derives the domain type itself
    signatures.push(await account.signTypedData({ ...typed_data, types }));
  }
  return submitBridge(bridgeQuoteId, { signatures }); // POST /bridge/{id}/submit
}
```

**Solana origin — `solana_versioned_tx`.** Sign it exactly as with `self_broadcast` (do not add instructions), then submit the signed tx instead of broadcasting it.

### `POST /bridge/{bridge_quote_id}/submit`

```jsonc
// request — exactly one of:
{ "signatures": ["0x..."] }                 // EVM origin: one per signable_payload.signatures[], same order
{ "signed_tx": "<base64 signed versioned tx>" }  // sol origin

// response 200
{ "bridge_quote_id": "brq_...", "status": "submitted",
  "source_tx_hash": null /* EVM: the deposit lands moments later, see status */ | "<sol signature>" }
```

| Response | Meaning | Do |
| --- | --- | --- |
| `400 quote_not_submittable` | the quote was `self_broadcast` | broadcast it yourself, or re-quote with `delivery: "submit"` |
| `400 signed_tx_mismatch` + `field` | not signed by your wallet over the quoted payload (`signatures[i]`, `signed_tx`, `message`, …) | sign the payload as given, with the quoted wallet |
| `404 bridge_quote_not_found` | unknown id | re-quote |
| `409 bridge_already_submitted` | this quote already holds a submission (or its one-hour retry window passed) | poll status; never sign a second deposit for one quote |
| `410 bridge_quote_expired` | quote TTL (or Solana blockhash) lapsed, or the bridge refused an EVM permit that can no longer run | re-quote |
| `422 broadcast_rejected` | Solana origin: refused before anything moved (e.g. not enough balance) | fix, then re-quote |
| `502 broadcast_unavailable` | outcome unknown; it may have gone through | resubmit the **same** body (idempotent); if it keeps answering `502`, poll status |

Resubmitting the same body is always safe for up to an hour after the quote: it answers `200` once the deposit is (or already was) sent, and `502` while it can't tell. Never re-quote and sign again because of a `502`; poll status.

## `GET /bridge/{bridge_quote_id}/status`

The upstream bridge's destination fill **cannot be looked up from the source tx hash** — it needs the upstream's private request id, which Treasures stores and translates at the boundary.

```jsonc
{
  "bridge_quote_id": "brq_...",
  "status": "pending" | "completed" | "failed",
  "source_tx_hash": "...",      // null until source tx indexed; may stay null on failed if it reverted/never broadcast
  "destination_tx_hash": "...", // null until destination fill lands; stays null on failed
  "updated_at": 1730000123
}
```

Cached 2s; safe to poll at 1 Hz. Don't expect `source_tx_hash` immediately — the upstream indexes the source side asynchronously.

> **Self-broadcast: always cache the hash returned by your own broadcast call** (from `sendRawTransaction`). When `source_tx_hash` is `null`, that locally-cached hash is the ONLY way to look up the deposit tx on-chain.

<a id="missing-allowance-trap"></a>
> ⚠️ **Missing allowance = silent forever-pending (`self_broadcast`).** From an EVM chain, if the base token isn't approved to the bridge deposit address ([approvals](trading.md#first-time-approvals)), the deposit tx reverts on-chain, the upstream never picks up the funds, and `status` stays `pending` indefinitely with both hashes `null` and **no error surfaced** — the endpoint can't tell "reverted" from "still indexing". If a bridge sits `pending` > ~2 min with no `source_tx_hash`, check the deposit tx on-chain (via your cached hash), set the allowance if missing, and re-quote on a fresh `bridge_quote_id`. `delivery: "submit"` has no allowance step.

> ⚠️ **`status: "failed"` recovery.** Two terminal states collapse into `failed`: **failed-without-refund** (source funds taken, destination didn't land, no auto-refund — rare) and **refunded** (upstream returned source-chain USDC, e.g. destination liquidity dried up). The API doesn't distinguish them — inspect `source_tx_hash` on-chain (refund events, or the user's USDC balance pre/post). Either way, **re-quote on a fresh `bridge_quote_id`** — never reuse the failed id.

Bridge quote error codes: `400 same_chain_bridge`, `400 unsupported_route`, `404 bridge_quote_not_found`, `422 insufficient_liquidity` (transient — retry/smaller size). Submit codes: table above. Full table: [`errors.md`](errors.md).
