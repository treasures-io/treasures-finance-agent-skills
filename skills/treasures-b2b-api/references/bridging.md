# Bridging USDC across chains

Load when moving USDC between Solana and Ethereum. Treasures fronts an intent-based bridge: it issues the quote and a privacy-safe status proxy; **the agent signs and broadcasts the bridge tx themselves.** Both wallets + both signatures are mandatory on the quote (a bridge spans both chains).

## `POST /bridge/quote`

```jsonc
// request — both wallets + both signatures required
{
  "from_chain": "sol" | "eth",
  "to_chain": "sol" | "eth",            // must differ from from_chain (else 400 same_chain_bridge)
  "amount_usdc": "100",                  // capped at 100,000 USDC; exceeding → 400 invalid_request
  "max_slippage_bps": 50,                // 10 ≤ value ≤ 500 (tighter than /quote/*)
  "sol_wallet": "...", "eth_wallet": "0x...",   // both required
  "ownership_proof": { "sol_signature": "<base64 ed25519>", "eth_signature": "0x...", "issued_at": 1730000000 }
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
    { "type": "evm_eip1559_tx",
      "from": "0x...",                       // expected signer (matches eth_wallet)
      "tx_base64": "...",                    // serialized unsigned tx (nonce=0 — see trap below)
      "tx_hex": "0x...",                     // same tx, hex
      "approval_spender": "0x..." | null }   // see below
    /* OR */
    { "type": "solana_versioned_tx",
      "tx_base64": "...",
      "blockhash": "...",
      "last_valid_block_height": 123456789 } // resubmit/re-quote past this height
                                             // (no approval_spender — Solana has no allowances)
}
```

<a id="approval-spender"></a>
**`approval_spender` (EVM-only, `string | null`).** Non-null when the calldata requires a USDC allowance — the upstream pull-pattern deposit, i.e. the normal `eth → sol` flow. **Set the USDC allowance to this address before broadcasting.** It mirrors the tx's `to`, so you never decode `tx_hex`. `null` when the calldata is self-spend (direct `transfer`/`approve`/value-only) and no external allowance is needed → skip the approval step.

**Workflow:** (1) on `eth → sol` with non-null `approval_spender`, approve USDC to it ([approvals](trading.md#first-time-approvals)). (2) sign the payload with the `from_chain` key. (3) **broadcast to chain yourself.** (4) poll `/bridge/{id}/status` until `completed` or `failed`. (5) for a follow-on trade, quote it with the **actual landed** USDC from `/portfolio` (`usdc.{sol,eth}`) — you receive `estimated_amount_out_usdc`, which is **less** than you sent, not the input amount.

<a id="evm-sign-broadcast"></a>
## Sign + broadcast — EVM (`evm_eip1559_tx`)

> ⚠️ **The serialized tx has `nonce=0`.** Treasures can't know your wallet's nonce, so it emits `nonce=0`. You MUST fetch the real nonce and inject it before signing, or broadcast is rejected ("nonce too low") on any wallet with history. **Most common EVM-bridge footgun.** Everything else — gas, `maxFeePerGas`/`maxPriorityFeePerGas`, `chainId` — is already populated in the serialized tx (`parseTransaction` + spread preserves it), so the **nonce is the only field you change**.

```ts
import { parseTransaction, createPublicClient, http } from "viem";
import { privateKeyToAccount } from "viem/accounts";
import { mainnet } from "viem/chains";

const publicClient = createPublicClient({ chain: mainnet, transport: http(ETH_RPC_URL) });
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

> **Always cache the hash returned by your own broadcast call** (from `sendRawTransaction`). When `source_tx_hash` is `null`, that locally-cached hash is the ONLY way to look up the deposit tx on-chain.

<a id="missing-allowance-trap"></a>
> ⚠️ **Missing USDC allowance = silent forever-pending.** On `eth → sol`, if USDC isn't approved to the bridge deposit address ([approvals](trading.md#first-time-approvals)), the deposit tx reverts on-chain, the upstream never picks up the funds, and `status` stays `pending` indefinitely with both hashes `null` and **no error surfaced** — the endpoint can't tell "reverted" from "still indexing". If a bridge sits `pending` > ~2 min with no `source_tx_hash`, check the deposit tx on-chain (via your cached hash), set the allowance if missing, and re-quote on a fresh `bridge_quote_id`.

> ⚠️ **`status: "failed"` recovery.** Two terminal states collapse into `failed`: **failed-without-refund** (source funds taken, destination didn't land, no auto-refund — rare) and **refunded** (upstream returned source-chain USDC, e.g. destination liquidity dried up). The API doesn't distinguish them — inspect `source_tx_hash` on-chain (refund events, or the user's USDC balance pre/post). Either way, **re-quote on a fresh `bridge_quote_id`** — never reuse the failed id.

Bridge error codes: `400 same_chain_bridge`, `400 unsupported_route`, `404 bridge_quote_not_found`, `422 insufficient_liquidity` (transient — retry/smaller size). Full table: [`errors.md`](errors.md).
