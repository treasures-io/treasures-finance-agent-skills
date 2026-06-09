# Auth — ownership proof

Load when building the `ownership_proof` for `/quote/buy`, `/quote/sell`, or `/bridge/quote`. Challenge format, encoding rules, and validity window are in the entry doc ([`../SKILL.md`](../SKILL.md#auth-essentials)) — this file is the rules, code, and troubleshooting.

## When required & wallet rules

Required on `POST /quote/buy`, `POST /quote/sell`, `POST /bridge/quote` (bridge needs **both** signatures). Everything else is unauthenticated. `/trade/submit` is gated by the per-leg signed payload itself — a strictly stronger proof than `ownership_proof`.

At least one of `sol_wallet` / `eth_wallet` is required on `/quote/*`; `/bridge/quote` requires both. Each wallet you send must have a matching signature, verified cryptographically (Ed25519 for Solana, secp256k1 EIP-191 recovery for Ethereum).

- **All-or-nothing per proof.** The signed proof must match the wallets on the same request. If you signed over both wallets, you must send both wallets + both signatures on every request reusing that proof — you can't send one half. A proof over `{sol_wallet, eth_wallet}` is invalid for a request with only `sol_wallet` (the empty-string line for the absent side changes the challenge → recovery fails → `401 ownership_proof_sol_invalid` / `_eth_invalid`). For a single-wallet `/quote/*` request, sign a separate single-wallet proof.
- **No EIP-1271 / smart wallets.** The EVM verifier is EOA-only (`ecrecover` against EIP-191 `personal_sign`). Contract-account signatures are not accepted.
- **Reuse:** one proof works across `/quote/buy`, `/quote/sell`, and `/bridge/quote` within the skew window — no server-side nonce store. A single-wallet proof works only on the matching `/quote/*` calls (bridge needs both). Re-sign once it ages out.

```jsonc
// Both-wallets proof (works on all three endpoints)
"ownership_proof": {
  "sol_signature": "<base64 ed25519>",   // required iff sol_wallet present
  "eth_signature": "0x...",              // required iff eth_wallet present (EIP-191)
  "issued_at": 1730000000
}
// Single-wallet proof — OMIT the absent side's signature KEY entirely
// (not "" / null — schema is .strict() → 400). Only valid on /quote/*.
"ownership_proof": { "eth_signature": "0x...", "issued_at": 1730000000 }
```

## Reference signing

Two primitives against `@noble/curves` + `@solana/web3.js` (Solana) and `viem` (Ethereum):

```ts
// Solana: Ed25519 sign over the UTF-8 challenge bytes, base64-encode.
// @noble/curves because the challenge is raw bytes, not a tx — web3.js Keypair
// methods are tx-oriented. (Trade signing uses tx.sign([keypair]); see trading.md.)
import { ed25519 } from "@noble/curves/ed25519";
import { Keypair } from "@solana/web3.js";

function signOwnershipSol(challenge: string, keypair: Keypair): string {
  const seed = keypair.secretKey.subarray(0, 32); // noble signs from the 32-byte seed
  return Buffer.from(ed25519.sign(Buffer.from(challenge, "utf8"), seed)).toString("base64");
}

// Ethereum: EIP-191 personal_sign — viem applies the prefix for you.
import { privateKeyToAccount } from "viem/accounts";
const signOwnershipEvm = (challenge: string, account: ReturnType<typeof privateKeyToAccount>) =>
  account.signMessage({ message: challenge });
```

Driver — builds the challenge, signs whichever sides you hold keys for:

```ts
function buildChallenge(issuedAt: number, solWallet?: string, ethWallet?: string): string {
  return ["treasures-finance-quote-v1", String(issuedAt),
    solWallet ?? "", (ethWallet ?? "").toLowerCase()].join("\n"); // eth lowercased
}

const issuedAt = Math.floor(Date.now() / 1000);
const challenge = buildChallenge(issuedAt, solKeypair?.publicKey.toBase58(), ethAccount?.address);
const ownership_proof = {
  issued_at: issuedAt,
  ...(solKeypair ? { sol_signature: signOwnershipSol(challenge, solKeypair) } : {}),
  ...(ethAccount ? { eth_signature: await signOwnershipEvm(challenge, ethAccount) } : {}),
};
```

<a id="embedded-wallets"></a>
## Embedded / managed wallets (Solana) — read before you sign

The reference signer assumes you hold the raw secret key. Most agent platforms don't — they call a managed wallet's `signMessage(...)` (Privy, Turnkey, Crossmint, …). **This is where Solana proofs fail silently** with `401 ownership_proof_sol_invalid`. The server verifies a **raw Ed25519 signature over the exact UTF-8 challenge bytes — no envelope — base64-encoded**:

```ts
ed25519.verify(signatureBytes, utf8ChallengeBytes, walletPubkeyBytes); // must be true
```

Rule out each of these — every one yields a rejected signature:

- **Wrong encoding (most common).** Correct raw Ed25519 but encoded **base58** (Solana default) instead of **base64**. The server does `Buffer.from(sig, 'base64')`, so base58 decodes to garbage. → base64-encode the raw 64 bytes.
- **Off-chain envelope / SIWS.** Signer prepends Solana's off-chain domain (`\xffsolana offchain` + header) or wraps in Sign-In-With-Solana → signed bytes ≠ challenge bytes. → sign the challenge **verbatim**.
- **Transaction signing.** Wallet only exposes `signTransaction` — that signs a tx, not your message. → use a detached **message** signer.
- **Pre-hashing.** We sign the raw message (PureEdDSA, RFC 8032), not a hash.

> **Why EVM "just works" but Solana doesn't:** EIP-191 `personal_sign` is the one universal Ethereum message standard — every embedded EVM wallet implements it identically. Solana has no equivalent; embedded signers each differ. Same agent code can pass on EVM and fail on Solana. **Verify locally** (below).

### Privy (exact path)

Privy's `signMessage` signs the **raw bytes** you pass — correct. The only trap is encoding:

```ts
// React (@privy-io/react-auth/solana) — signature is a Uint8Array
const { signature } = await signMessage({
  message: new TextEncoder().encode(challenge), wallet: solWallet,
});
const sol_signature = Buffer.from(signature).toString("base64"); // base64 — NOT bs58.encode()

// Server / REST — message is base64 of the raw bytes; returned signature is ALREADY base64
const message = Buffer.from(challenge, "utf8").toString("base64");
const { signature: sol_signature } = await privy.wallets().solana()
  .signMessage(walletId, { message, encoding: "base64" }); // use as-is
```

> ⚠️ Privy's quickstart shows `bs58.encode(signature)` (base58, for display). **We require base64** — base58 is the single most common cause of `ownership_proof_sol_invalid`.

### Self-verify before you send (catches the mismatch offline)

```ts
import { ed25519 } from "@noble/curves/ed25519";
import { PublicKey } from "@solana/web3.js";

function assertSolProof(challenge: string, solSignatureBase64: string, solWallet: string): void {
  const ok = ed25519.verify(
    Buffer.from(solSignatureBase64, "base64"),
    Buffer.from(challenge, "utf8"),
    new PublicKey(solWallet).toBytes(),
  );
  if (!ok) throw new Error(
    "sol_signature does not verify as raw Ed25519 over the challenge — likely base58 " +
    "(use base64), an off-chain/SIWS envelope, a signed tx, or wrong bytes.");
}
```

If `assertSolProof` passes locally, the server accepts it. If it throws, no request shape fixes it — the signer is wrong.

## Test vectors

Deterministic — sign the listed challenge with the seed and you **must** reproduce these byte-for-byte (Ed25519 is deterministic; viem's ECDSA uses RFC 6979). Diff your signer's output to localize a mismatch.

> `issued_at` is fixed at `1730000000`, far in the past → live calls with these return `401 ownership_proof_skewed`. **Offline verification only.** Re-sign with current `issued_at` for real calls.

```
test seed / privkey: 0x0102030405060708090a0b0c0d0e0f101112131415161718191a1b1c1d1e1f20
```

```jsonc
// Solana (sol_wallet only — note the trailing \n for the empty eth line)
sol_wallet: "9C6hybhQ6Aycep9jaUnP6uL9ZYvDjUp1aSkFWPUFJtpj"
challenge:  "treasures-finance-quote-v1\n1730000000\n9C6hybhQ6Aycep9jaUnP6uL9ZYvDjUp1aSkFWPUFJtpj\n"
sol_signature (base64, 64 bytes): "zqbTPdEaJ+y4RULTNsnxjnG89bi2Mmf9YYMl2XvwP7NbGTpEUf9SnHbPYhgfnRUva5WwKdZKx6KkjANJcLVPCg=="

// Ethereum (eth_wallet only — note the empty sol line, \n\n)
eth_wallet: "0x6370ef2f4db3611d657b90667de398a2cc2a370c"
challenge:  "treasures-finance-quote-v1\n1730000000\n\n0x6370ef2f4db3611d657b90667de398a2cc2a370c"
eth_signature (EIP-191 personal_sign): "0xb9e2724af6d5016fb0141c384d63dbe7ead9225b4897c17eb953cc089f43fdbc6222b5b9b97c0544daa8091e3315474dde1f05ab57e12d798b3d80c4f85d270d1c"
```

## Auth error codes

| HTTP | `error` | Cause / fix |
| --- | --- | --- |
| 401 | `ownership_proof_sol_invalid` | `sol_signature` missing or fails Ed25519 verify — re-sign (base64 **not** base58; exact bytes incl. trailing `\n`) |
| 401 | `ownership_proof_eth_invalid` | `eth_signature` missing or EIP-191 recovery ≠ `eth_wallet` — re-sign |
| 401 | `ownership_proof_skewed` | `issued_at` outside `[now − 300s, now + 30s]` — re-sign with current time |
| 401 | `ownership_proof_invalid` | Neither wallet supplied (defensive; route schema usually rejects as `400 invalid_request` first) |
