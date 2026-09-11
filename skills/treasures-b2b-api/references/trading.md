# Trading — quote, submit, status

Load when quoting/submitting trades or polling status. Pre-flight traps (approvals, gas, slippage, TTLs) and the buy-one/sell-all rule are summarized in [`../SKILL.md`](../SKILL.md); this file is the schemas, signing code, and status contracts.

> **`tx_hash` vs `order_hash` (defined once, applies everywhere).** Exactly one is non-null at a time. **Solana legs** and **`completed` Eth / Robinhood / Base** legs: `tx_hash` carries the on-chain hash, `order_hash = null`. **Any EVM venue in-flight** (Eth, Robinhood, Base — all gasless): `tx_hash = null`, the gasless EVM order hash sits on `order_hash`; once it fills, `tx_hash` flips to the settlement hash and `order_hash → null`. **The one exception is a [speed-route](#speed) leg** (`priority:"speed"` on robinhood/base): it is an on-chain transaction from the start, so `tx_hash` is populated the moment it is accepted, `order_hash` is always `null`, and there is no flip. External `/trades` rows: both null.

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

**`base_asset` + `amount_base` are on _every_ quote leg** — the base currency and its human amount (spent on a buy, received on a sell): `usdc` on sol/eth/base, `usdg` on Robinhood. To detect the Robinhood venue, test `base_asset === "usdg"`; **do not** treat the field's mere presence as the signal (it's present on the other legs too, as `"usdc"`).

> ⚠️ **`base_asset:"usdc"` does not identify a contract.** Solana, Ethereum and Base each settle in *their own* USDC — same symbol, three different mints/contracts. Always pair `base_asset` with the leg's **`chain`** before you resolve an address, size an allowance, or aggregate balances.

## `POST /quote/buy`

USDC → shares. Returns up to one quote per chain the ticker lists (or per chain in your `chain` array, when you send one) — sol/eth, plus robinhood (and base, where minted) — sorted best-rate-first. Mixed-currency candidates are ranked on **comparable** cost (a USDG leg is marked up by what acquiring USDG costs), never on nominal per-share parity — so an unpinned request can return a **USDG-denominated robinhood leg** when it genuinely prices best. Always read each leg's `chain` + `base_asset` before interpreting amounts.

```jsonc
// request
{
  "ticker": "AAPL",
  "amount_usdc": "100",                  // exactIn, string, 6 decimals max
  "chain": "sol" | "eth" | "robinhood" | "base" | ["sol", "eth"] | null,  // one chain = pin; an array (1–4, unique) = auto-route within those chains only (["x"] ≡ "x"); null = every chain the ticker lists, robinhood/base included (see #robinhood, #base)
  "protocol": "ondo" | "xstocks" | "robinhood" | "coinbase" | ["ondo", "xstocks"] | null, // one protocol = pin; an array (1–4, unique) = auto-route within those protocols only (["x"] ≡ "x"); null = every protocol the ticker lists. To exclude one, send the other three. "robinhood" only pairs with chain:"robinhood", "coinbase" only with chain:"base"
  "preferred_chain": "sol" | null,       // put this chain first — a PREFERENCE, not a filter: it reorders, it never narrows. Buy: that chain's leg becomes quote_index 0. Sell: it is filled from first, then the rest. Must be one of the chains in `chain` when you send `chain` (else 400); a chain with nothing to offer falls back to normal ordering. Sell guarantee: preferred-chain holdings that cannot be priced on the first attempt rejoin the rest of the fill, so the preference can never leave a sell short — see POST /quote/sell
  "priority": "speed" | null,            // null/omitted = today's routing. "speed" asks for the speed route on robinhood/base — see below
  "max_slippage_bps": 50,                // 10 ≤ value ≤ 5000
  "sol_wallet": "...", "eth_wallet": "0x...",   // at least one
  "ownership_proof": { /* see auth.md */ },
  "quote_only": false                    // true = price-only preview (see below); buy-only
}

// response 200
{
  "quote_id": "qte_<nanoid>",
  "side": "buy",
  "ticker": "AAPL",
  "expires_at": 1730000040,              // unix seconds
  "tradfi_reference": { "price_usd": "232.45", "market_status": "open", "as_of": 1730000000 },  // always present on a 200; market_status: opaque string — don't branch on specific values
  "quotes": [
    {
      "quote_index": 0,                  // stable handle referenced at /trade/submit
      "chain": "sol",
      "protocol": "ondo",
      "price_usdc_per_share": "234.20",
      "estimated_output_shares": "0.4988",
      "estimated_output_tokens": "0.4988",
      "base_asset": "usdc",              // spend currency: "usdc" on sol/eth, "usdg" on robinhood
      "amount_base": "100",              // base spent, human-decimal (= amount_usdc on a buy)
      "cost_breakdown_bps": {
        "treasures_fee_bps": 30,
        "dex_swap_fee_bps": 8,
        "estimated_slippage_bps": 12,
        "slippage_vs_tradfi_bps": 75     // signed; positive = unfavorable. Always present on a 200
                                         // (a quote with no reference is refused, not returned).
      },
      "signable_payloads": [{ "type": "solana_versioned_tx", "tx_base64": "..." }],
      "tradability": "thin",             // optional: probe-tier warning for this leg's venue
      "warn_reason": "low_volume",       // optional: why — see the action table below
      "thin_since": "2026-08-24T16:30:00.025Z"
    },
    {
      "quote_index": 1, "chain": "eth", "protocol": "xstocks",
      "price_usdc_per_share": "235.10", "estimated_output_shares": "0.4969", "estimated_output_tokens": "0.4969",
      "base_asset": "usdc", "amount_base": "100",
      "cost_breakdown_bps": { "treasures_fee_bps": 30, "dex_swap_fee_bps": 5, "estimated_slippage_bps": 18, "slippage_vs_tradfi_bps": 114 },
      "gasless": true,                   // who pays this leg's gas — see "Gas on a quote leg" below
      // Eth legs return EIP-712 typed data for the gasless order, NOT a raw tx:
      "signable_payloads": [{ "type": "evm_eip712_typed_data", "typed_data": { /* domain, types, primaryType, message */ } }]
    }
  ]
}
```

**Buy submission rule.** Each `quote_index` is a **mutually-exclusive alternative** — sign + submit one index only. >1 leg → `400 quote_index_mismatch`. No chain preference → just take `quotes[0]` (best rate) or re-quote with the `chain` you want. To favour a chain **without** excluding the others, send `preferred_chain`: `quotes[0]` is then that chain's leg when the plan has one, and the remaining legs stay in price order behind it — so you can still fall back to a cheaper chain by reading `quotes[1]`.

<a id="gas-on-a-leg"></a>
**Gas on a quote leg (`gasless`, `estimated_gas_usd`).** Every leg carries `gasless`. `true` = you sign an order, a settlement network broadcasts it, and the trade costs you **no native gas** — that is every `eth`, `robinhood` and `base` leg by default. `false` appears only on a [**speed-route** leg](#speed) (`priority:"speed"`), which you sign as a whole transaction that is then broadcast against **your own** native balance; those legs also carry `estimated_gas_usd`, the estimated USD cost of that broadcast (`null` when no USD gas price was available). The key can be **absent** on a Solana leg, where nothing upstream reports it — **absent is not `false`**; read `signable_payloads[0].type` when you need certainty. Note `price_impact_bps` reads `0` on a speed-route leg: that route reports no impact, and `0` is already this field's "favorable or unreported" value, not a measured zero.

**Price preview (`quote_only: true`).** Returns prices without an executable quote — for an estimate before committing. The wallet need **not** hold the funds, but you still pass `ownership_proof`. The response **omits** `quote_id`, `expires_at`, and every leg's `signable_payloads`; nothing is persisted, so there's no `/quote/{quote_id}/status` to poll and nothing to submit. Buy-only — sending `quote_only` to `/quote/sell` is rejected (`400 invalid_request`). Omit it (or set `false`) for a normal executable quote.

<a id="warnings"></a>
### `warnings[]` and the advisory acknowledgement

Every quote response — buy, sell and `quote_only` preview — carries a top-level `warnings[]`. It is **always present**, `[]` when there is nothing to disclose, so an empty list never means "an older build". On an executable quote a non-empty set is accompanied by `warnings_ack_token`, a 32-hex-character digest bound to that `quote_id` and that exact set. **A `quote_only` preview never carries the token**, warned or not: it has no `quote_id` to bind one to and nothing to submit it against — read its `warnings[]`, then re-quote executably when you want to trade.

```jsonc
{
  "quote_id": "qte_...",
  "tradfi_reference": null,
  "warnings": [
    { "code": "no_reference" },
    { "code": "low_volume", "thin_since": "2026-08-24T16:30:00.000Z" }
  ],
  "warnings_ack_token": "3f2a...c91b",
  "quotes": [ /* ... */ ]
}
```

Each entry has a `code` plus whatever material values that code carries. Two codes come from the price check; every other value is the routed venue's own `warn_reason` from the table below, so you learn **one** vocabulary rather than two — and, as there, an unrecognised code means "warned, reason unknown", not a schema violation.

| `code` | What it means | What to do |
|---|---|---|
| `no_reference` | No reference market price could be obtained for this asset, so nothing independent checked the quote's price. Permanent for assets our market-data provider does not cover; transient during a feed outage | Decide on the venue price alone, or skip the ticker. `tradfi_reference` is `null` on these quotes |
| `off_reference` | A reference exists and the quote sits beyond its band, in the direction that costs you | Re-quote in 30–60s; the band is not caller-tunable, so a bigger `max_slippage_bps` will not clear it |

**Warned quotes are executable — this API discloses, it does not gate.** `POST /trade/submit` accepts an **optional** `acknowledged_warnings`: echo the quote's `warnings_ack_token` and the acknowledgement is recorded beside the trade; omit it and the submit proceeds exactly as before. Sending a value that does not match the warnings served with that `quote_id` is refused with `422 warnings_not_acknowledged` rather than recorded — clear it by omitting the field, or by re-quoting and echoing the fresh token. On a clean quote (`warnings: []`) the field is ignored.

**Whether your own end users are shown any of this is your call and your record.** Treasures records what it served you on every trade, echo or not; it records nothing about what you told anyone downstream.

Beyond a plausibility bound the price check stops disclosing and refuses outright — `422 no_routes` with `reason: "price_implausible"`, no quote issued, no acknowledgement possible. See [`errors.md`](errors.md).

<a id="tradability"></a>
### Tradability warnings on quote legs

Any leg may carry three optional advisory fields describing **that leg's own venue** (its `ticker` + `protocol` + `chain`), raised by a background probe that quotes a ladder of buy sizes against every listed venue and watches whether real orders settle:

| Field | Values | Meaning |
| --- | --- | --- |
| `tradability` | `"thin"` \| `"untradable"` | `thin` = the venue fills, but not at every size. `untradable` = a full measurement window passed with no fill at any size. |
| `warn_reason` | `"settlement_failure"` \| `"no_price_feed"` \| `"no_fill_window"` \| `"no_settlements"` \| `"low_volume"` | Why the warning was raised — act per the table below. |
| `thin_since` | ISO-8601 string | When the warning was first raised. Present exactly when `tradability` is. |

**Advisory, never a block.** A warned leg is still planned, still priced, and still submittable — no leg is dropped, no quote is refused, and there is no request param to exclude warned venues. `untradable` means "nothing filled across a full measurement window", **not** "disabled". Decide with the reason:

| `warn_reason` | What it means | What to do |
|---|---|---|
| `settlement_failure` | A real order on this venue recently expired without settling — quotes here can look fine and still never fill | Prefer another `quote_index` / venue; if this is the only venue, warn the user before submitting |
| `no_price_feed` | The venue holds no price for this pair at all | Do not submit to this venue at any size; pick another venue or skip the ticker |
| `no_fill_window` | No probe size has filled here for weeks | Treat as likely-to-fail; prefer another venue |
| `no_settlements` | No value-bearing on-chain settlement observed for this venue in the census window | Treat as likely-to-fail; prefer another venue |
| `low_volume` | Measured on-chain trading volume is below a floor | Expect poor pricing on entry AND exit; keep sizes small or prefer another venue |

- **Absent ≠ null.** All three keys are omitted, never sent as `null`. An absent `tradability` means the venue is **unmeasured or unwarned** — never "healthy, value null". `warn_reason` can also be absent beside a *present* `tradability`: that warning predates the reason field, so read it as "reason not recorded", never "no reason".
- **`warn_reason` is an open vocabulary.** New values are added over time; an unrecognised one means "warned, reason unknown" — treat it like any reason above rather than ignoring it.
- **Sell legs publish side-neutral reasons only.** `/quote/sell` legs carry the same three fields, but only when the reason is `low_volume`, `no_price_feed` or `no_settlements`. A venue warned for `no_fill_window` or `settlement_failure` — both buy-sided — emits **nothing at all** on a sell, and so does a warning whose reason was never recorded. Nothing here ever marks a holding unsellable.
- **The value is not stable over a venue's lifetime.** A `low_volume` warning flips to `settlement_failure` once a real order dies there. Re-read it per quote; don't cache it.
- Same fields, venue-by-venue and per ticker before you quote: [`data.md`](data.md#tradability-warnings). A whole-quote refusal is a different tier — see `422 no_routes` in [`errors.md`](errors.md).

### Signable payload shapes

A leg's `chain` determines the variant:

| `chain` | `type` | Payload field | Sign / submit as |
| --- | --- | --- | --- |
| `sol` | `solana_versioned_tx` | `tx_base64` — unsigned `VersionedTransaction` | sign with Solana key → `{ type: "solana_versioned_tx", signed_tx_base64 }` |
| `eth` | `evm_eip712_typed_data` | `typed_data` — full EIP-712 object | sign typed data (`eth_signTypedData_v4`) → `{ type: "evm_eip712_signature", signature: "0x..." }` |
| `robinhood` | `evm_eip712_typed_data` (one per leg, chain 4663) | `typed_data` — full EIP-712 object (gasless order) | sign typed data (`eth_signTypedData_v4`) → `{ type: "evm_eip712_signature", signature: "0x..." }` — **same as `eth`** (see [Robinhood](#robinhood)) |
| `base` | `evm_eip712_typed_data` (one per leg, chain 8453) | `typed_data` — full EIP-712 object (gasless order) | sign typed data (`eth_signTypedData_v4`) → `{ type: "evm_eip712_signature", signature: "0x..." }` — **same as `eth`** (see [Base](#base)) |
| `robinhood` / `base` **with `priority:"speed"`** | `evm_eip1559_tx` (8453) or `evm_legacy_tx` (4663) — **one or two of them** | `tx_hex` — a **complete** unsigned transaction (nonce, gas, fees already set), plus `role` (`approve` \| `swap`), `approval_spender` (null on the approve), `chain_id`, `nonce`, `gas` and the fee fields | sign each payload's bytes as-is (`signTransaction`) → one `{ type: "evm_signed_tx", signed_tx_hex: "0x..." }` per payload, **all of them**; **Treasures broadcasts them** — see [speed route](#speed) |
| `base` **with `priority:"speed"` + `execution:"user_operation"`** | `evm_calls` (exactly one per leg) | `calls[]` — one or two `{ role, to, data, value:"0" }` (`approve` first when present, then `swap`), plus `sender`, `chain_id`, `entry_point` (v0.7) and `approval_spender` | pack the calls into ONE ERC-4337 v0.7 UserOperation from your MAv2-delegated wallet (`execute` / `executeBatch` in order), sponsor with your paymaster, sign, return `{ type: "evm_user_operation", user_operation }`; **Treasures submits it to a bundler** — see [user operation](#user-operation) |

> By default the `eth`, `robinhood` and `base` quote payloads are **all** `evm_eip712_typed_data` — off-chain order signatures you sign but **never broadcast**. The `typed_data.domain.chainId` tells you which network the order settles on (`1` / `4663` / `8453`) — sign it as given, never rewrite it. A [speed-route](#speed) leg is the one trade payload that is a real transaction (`evm_eip1559_tx` / `evm_legacy_tx`): you still don't broadcast it — you sign the given bytes and Treasures sends them. **The only EVM payload you broadcast yourself is the bridge payload** (`evm_eip1559_tx`, see [`bridging.md`](bridging.md)) — and unlike a speed-route payload, the bridge one carries **no nonce**, which you must set. Branch on `type`, never on `chain` alone.

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

Shares → USDC. Server reads holdings across every chain × protocol — sol/eth from indexed holdings, robinhood/base sized live against your own on-chain balance — then greedy-fills `amount_shares` from the best-priced pools. Robinhood (and base) positions **join the unpinned fill**: a `sell all` no longer needs a `chain:"robinhood"` pin to reach a robinhood position. Mixed-currency pools are ranked on comparable USD proceeds (net of what converting USDG back costs), never nominal parity. A pinned `chain` still narrows the sell to **that one venue**; an array narrows it to **those venues** (see [Robinhood](#robinhood) / [Base](#base)). **Same response shape as `/quote/buy`**, with these differences:

- request: `amount_shares` (18 decimals max) replaces `amount_usdc`.
- `chain` and `protocol` accept the same one-value / array / null shapes as `/quote/buy`.
- `preferred_chain` names the chain to sell from **first** — a preference, not a filter: the remaining chains still fill whatever is left, in the usual best-price order. Must be one of the chains in `chain` when you send `chain`. The preferred chain's holdings are priced first; whatever it does not fill continues over the remaining chains **and** over any preferred-chain holding that could not be priced on that first attempt, so the preference costs at most one extra pricing attempt and can never leave a sell short of what the same sell would have covered without it.
- `priority: "speed"` works the same way as on a buy — robinhood/base legs come back as [speed-route](#speed) transactions you sign whole, `eth` holdings are never part of the fill, and a chain the speed route can't serve is dropped from it (`speed_route_unavailable`). On a sell the spent token is the **stock token**; if it is not yet approved to the router, the leg's first payload is the bundled `approve` for it — sign both.
- each quote carries `shares_consumed` + `tokens_consumed` + `estimated_output_usdc` instead of `estimated_output_*`.
- response adds `"totals": { "shares_total": "0.5", "usdc_total_estimated": "117.10" }`.
- `signable_payloads` shapes are identical (sol → `solana_versioned_tx`, eth / robinhood / base → `evm_eip712_typed_data`).
- all other buy fields carry over unchanged: `quote_index`, `chain`, `protocol`, `price_usdc_per_share`, `base_asset`, `amount_base`, `cost_breakdown_bps`, `tradfi_reference`. On a sell, `amount_base` equals `estimated_output_usdc` base-denominated (USDG on robinhood).

**Sell submission rule.** Sell legs are **additive** — sign + submit **every** `quote_index` returned. Subset → `400 incomplete_submit`. To sell from one chain only, re-quote with `chain` pinned; to sell from a subset, send an array.

<a id="preview"></a>
## `POST /quote/preview` — indicative prices, no wallet

Prices a buy with **no wallet and no ownership proof** — the route for a screen, a watchlist, or "what would this cost?". Unlike every other quote route it **requires your integrator key**: send `X-API-Key: tik_…`. No key is `401 invalid_api_key`; a read-only `trk_` reporting key is `403 insufficient_scope` (it reaches the reporting routes and nothing else).

- **Body = the `/quote/buy` body minus `sol_wallet`, `eth_wallet`, `ownership_proof` and `quote_only`.** All four are rejected as unknown fields (`400 invalid_request`) — there is no wallet to bind, so there is nothing to prove and nothing to execute. `ticker`, `amount_usdc` and `max_slippage_bps` are required; `chain`, `protocol` and `preferred_chain` behave exactly as on `/quote/buy`.
- **Response = the `/quote/buy` shape minus `quote_id`, `expires_at`, every leg's `signable_payloads`, and `warnings_ack_token`.** Everything else is there, including `warnings[]` (always present, `[]` when clean) and each leg's `chain`, `protocol`, `base_asset` and `cost_breakdown_bps` — so read a preview leg's amounts with the same `chain` + `base_asset` care as a real one.
- **Nothing is stored and nothing is executable.** No `/quote/{quote_id}/status` to poll, nothing to submit, and no `expires_at` to race. **Use this for indicative prices; use `/quote/buy` for anything you intend to sign** — the price you preview is not a price you hold.
- **10-second cache + a shared ceiling.** Identical requests inside a 10-second window are answered from one upstream fan-out; a refusal is never cached. "Identical" is judged on the request's meaning, not its bytes: `amount_usdc` is matched **exactly**, so a one-cent change is a different request, while `chain` and `protocol` are compared as **sets** — a single value and its one-element array (`"sol"` and `["sol"]`) are the same request. On top of the usual per-IP and per-organisation limits this route carries a **global ceiling shared by every caller**, so a `429` here can be someone else's traffic rather than your own quota — honour `Retry-After` and re-check, don't assume you are throttled for the minute.
- **`priority: "speed"` is accepted here** and prices what an executable speed request would route. There is no wallet, so the conditions that depend on one — your token balance, and whether the leg will carry a bundled `approve` — can't be known; they surface only on `/quote/buy` or `/quote/sell`.
- `422 no_routes` and `422 basket_price_unavailable` mean exactly what they mean on `/quote/buy`.

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

// A speed-route leg (priority:"speed") replies with the whole signed transaction — see #speed.
// A leg that issued TWO payloads (approve + swap) gets both back, IN THE ORDER THEY WERE ISSUED.
// The server matches each by nonce, but idempotency hashes the array — a replay that reorders the
// entries is treated as a new submit (409 leg_already_submitted), so keep the issued order.
{ "quote_id": "qte_...", "signed": [
  { "quote_index": 0, "signed_payloads": [
    { "type": "evm_signed_tx", "signed_tx_hex": "0x02f8..." },   // role: approve (nonce n)
    { "type": "evm_signed_tx", "signed_tx_hex": "0x02f8..." }    // role: swap    (nonce n+1)
  ] }
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
| robinhood | `broadcast` \| `broadcast_unknown` \| `broadcast_failed` | same gasless model as `eth` — the server relayed your signed order to the 4663 settlement network | `pending` until the order fills → `completed`/`failed` | poll `/status` (server also backfills; see [Robinhood](#robinhood)) |
| base | `broadcast` \| `broadcast_unknown` \| `broadcast_failed` | same gasless model as `eth` — the server relayed your signed order to the 8453 settlement network | `pending` until the order fills → `completed`/`failed` | poll `/status` (server also backfills; see [Base](#base)) |
| robinhood / base with `priority:"speed"` | `broadcast` \| `broadcast_unknown` \| `broadcast_failed` | your signed transaction was broadcast (`tx_hash` set — on a two-payload leg the approve went out and `tx_hash` is the swap's, sent once the approve mines), or refused before it was sent (`broadcast_failed` — bind, balance, native gas, simulation revert, nonce, `approve_failed`; nothing reached the chain except, on `approve_failed` after the approve was sent, the approve itself — and on `approve_failed` `tx_hash` is `null`, because the swap was never broadcast) | `pending` until the receipt lands, then `completed`/`failed` | poll; on `broadcast_failed` act per `error_code`, then re-quote |

`/status` legs **never** carry `broadcast`, `broadcast_unknown`, or `in_progress` (the latter is an `aggregate_status` value only). Only `broadcast_failed` survives untouched as a terminal failure.

- **Idempotency.** `(quote_id, quote_index)` is the dedup key, cached **24h**. Same `(quote_id, quote_index)` + same signed payload → cached prior result (200, same `tx_hash`). Different payload for an already-broadcast leg → `409 leg_already_submitted` carrying the original handle — `tx_hash` for a landed sol / completed EVM leg, or `order_hash` for an EVM leg still in-flight (read whichever is non-null). Safe to retry on network glitch.
- **`error_code` at submit** (when a leg is `failed`/`broadcast_failed`): same per-leg enum as `/status` below. `failed_legs[]` uses `internal_error` only.
- **Atomicity.** Validation (shape, signer recovery, payload-type) is all-or-nothing pre-broadcast → any leg fails to validate, none broadcast (`400 incomplete_submit` / specific error). After validation, broadcast is best-effort per leg: legs that landed stay irreversibly on-chain even if later ones fail. Re-quote any `failed_legs[]` indices on a fresh `quote_id`.

## `GET /quote/{quote_id}/status`

Aggregate + per-leg view. Poll at `poll_after_ms`: `1250` while a [speed-route](#speed) leg is in flight (it settles by receipt within a block or two), `10250` when only gasless or Solana legs remain (their upstream status is synced at most every 10 s per leg, so polling faster only returns the cached view) or a speed-route leg has sat unmined past 5 min. On a cached response (`is_cached: true`) the value is the time left on that cached view rather than the full window, so a client that polls late is not asked to wait another one — it is still never below 1000. Never faster than 1 Hz.

```jsonc
{
  "quote_id": "qte_...",
  "aggregate_status": "in_progress" | "completed" | "partial_failed" | "all_failed",
  "is_cached": false,
  "poll_after_ms": 1250,                 // while in_progress: the window, or what is left of a cached one; null once terminal (stop polling)
  "legs": [{
    "quote_index": 0, "trade_id": "trd_...", "ticker": "AAPL",
    "chain": "sol" | "eth" | "robinhood" | "base", "protocol": "ondo" | "xstocks" | "robinhood" | "coinbase", "side": "buy" | "sell",
    "tx_hash": "...", "order_hash": null,   // see tx_hash/order_hash split at top of file
    "status": "pending" | "completed" | "failed" | "broadcast_failed",
    "error_code": null,
    "filled_shares": "0.3", "filled_tokens": "0.3", "filled_usdc": "70.26",  // "0" unless status=="completed" — a terminal failed/broadcast_failed leg also reads "0"
    "last_synced_at": 1730000050
  }]
}
```

Per-leg `error_code` (when `status` is `failed`/`broadcast_failed`):

| Code | Meaning | Action |
| --- | --- | --- |
| `invalid_signature` | Payload malformed or doesn't recover to the quote-bound wallet | re-sign with the correct wallet |
| `quote_expired` | Cached quote / order / blockhash no longer valid | re-quote |
| `not_enough_balance_or_allowance` | (any EVM venue) wallet lacks token balance, or allowance to that chain's settlement contract — on a single-payload [speed-route](#speed) leg the target is the payload's own `approval_spender`, checked before anything is sent (a two-payload leg checks balance only: its approve is what fixes the allowance) | set the allowance **on that chain** ([approvals](#first-time-approvals)) or fund, then re-quote |
| `provider_error` | Other upstream rejection — and, on a [speed-route](#speed) leg, a pre-flight read or simulation that never came back. Nothing was sent in that case | re-quote; retry on persistent failure |
| `status_check_exhausted` | Eth order sat > 1h unresolved, or a speed-route transaction > 6h (operator handoff) | re-quote |
| `nonce_conflict` | Speed route only: another transaction from this wallet took the nonce, and yours never mined | Wait for your earlier trade on that chain to reach a terminal status, then re-quote (only a new quote issues a new nonce) |
| `insufficient_native_gas` | Speed route only: the signed gas × fee exceeds the wallet's native balance on that chain | Fund native ETH on that chain, then re-quote |
| `swap_reverted` | Speed route only: the swap reverted — in simulation (nothing sent, no gas spent) or on chain. The market moved past the quote's minimum return | re-quote |
| `tx_dropped` | Speed route only: the transaction was accepted, then evicted without mining, and a re-send did not recover it | re-quote and re-sign |
| `approve_failed` | Speed route only, two-payload leg: the bundled `approve` reverted (in simulation — nothing sent — or on chain), or did not land within 10 minutes; the swap was never sent | re-quote — if the approve did mine, the new quote is single-payload |
| `sponsorship_rejected` | [`execution:"user_operation"`](#user-operation) only: your paymaster refused to sponsor the op (deposit / stake / rate limit / policy signature / paymaster revert) — nothing was submitted | fix the sponsorship under your Gas Manager policy, re-sign, re-submit (the quote is still valid inside its window) |

> `internal_error` is `/trade/submit`-only (in `failed_legs[]`); never appears here. Re-quote those indices on a fresh `quote_id`.

Stop polling once `aggregate_status` ∈ `{completed, partial_failed, all_failed}` — equivalently, once `poll_after_ms` is `null`:

| Condition | `aggregate_status` |
| --- | --- |
| Any leg still `pending` | `in_progress` |
| All terminal **and all** `completed` | `completed` |
| All terminal, ≥1 `completed` + ≥1 failed | `partial_failed` |
| All terminal, **none** `completed` | `all_failed` |

`completed` is reachable **only** when every leg is `completed` — one `broadcast_failed` always demotes to `partial_failed`/`all_failed`. Don't trust aggregate alone for "did all legs land"; check per-leg too.

<a id="robinhood"></a>
## Robinhood Chain (`chain:"robinhood"`) — sign-only, gasless, priced in USDG

A venue that trades **Robinhood Stock Tokens** on **Robinhood Chain** (chain id **4663**), priced in **USDG**. **Execution is byte-identical to the [`eth` venue](#signable-payload-shapes):** the quote returns one `evm_eip712_typed_data` order — sign it with `eth_signTypedData_v4` (the same `signEvm` helper; strip `EIP712Domain`), submit `{ type:"evm_eip712_signature", signature }`, then poll `/status` (`order_hash` while in-flight → `tx_hash` on fill; the server also backfills stuck orders). **You broadcast nothing, pay no gas, and need no 4663 RPC.** (Short version: pre-flight trap #6 in [`../SKILL.md`](../SKILL.md).)

Only **four things differ from `eth`** — everything else carries over unchanged:

1. **Priced in USDG, not USDC.** Read `base_asset:"usdg"` + `amount_base`; the `_usdc`-named fields (`amount_usdc`, `price_usdc_per_share`, `estimated_output_usdc`, …) all carry **USDG** here. Detect the venue with `base_asset === "usdg"` — never the field's mere presence (it's on sol/eth legs too, as `"usdc"`).
2. **Auto-routed — no longer pin-only.** A `chain:null` request co-ranks robinhood against sol/eth (on comparable cost, so a USDG leg never wins on a nominal artifact), and an unpinned sell greedy-fills robinhood positions alongside the rest. Pass `chain:"robinhood"` only to force the venue. **Consequence: code written when this venue was pin-only may receive a USDG leg it never expects — branch on `base_asset`, not on whether you sent `chain`.** Your `eth_wallet` (the same 0x EOA) is the signer on 4663.
3. **Liquid names only.** Only the marquee tickers (AAPL/TSLA/NVDA/AMD) fill; other Robinhood tokens have no settlement liquidity and return `422 no_routes` even though they list on `/stocks/tickers`. Treat `no_routes` as "not fillable right now".
4. **No external-row reconciler — you MUST submit to get a `/trades` row.** For `sol`/`eth`, tokens acquired outside Treasures reconcile in as a synthetic `external` row; Robinhood positions do **not** (holdings read live from your 4663 balance), so an unsubmitted trade is **absent from `/trades`** with **no cost basis** no matter what. `/portfolio` does not wait for submission, though: your 4663 Stock Tokens are read for **any** wallet when you send your `tik_` integrator key (and otherwise for every wallet Treasures knows on any chain) — snapshotted for 30 s and invalidated the moment a trade settles — so a token transferred in from outside shows exactly like one bought here. `X-API-Key` is OPTIONAL on `/portfolio`, so an ANONYMOUS call for an `eth_wallet` Treasures has never seen gets an empty `positions` list and is not flagged `partial` — **send the key and that case disappears**; `usdg.robinhood` cash is never gated either way — it still reads live even for an unknown wallet. Submitted trades get **on-chain-observed** cost basis + P&L (realized from the fill, same fidelity as `eth`).

**Funding is out of scope:** hold **USDG** on 4663 to buy, or **Stock Tokens** to sell — getting them onto 4663 (bridge/transfer) is out-of-band. Buy and sell are both supported; a sell trades Stock Token → USDG, sized against your on-chain 4663 balance.

The single quote leg (`chain`/`protocol` both `"robinhood"`):

```jsonc
{
  "quote_index": 0,
  "chain": "robinhood",
  "protocol": "robinhood",
  "base_asset": "usdg",            // spend currency — the _usdc-named fields carry USDG on this venue
  "amount_base": "100",            // USDG spent (read this, not amount_usdc)
  "price_usdc_per_share": "325.98",
  "estimated_output_shares": "0.3067", "estimated_output_tokens": "0.3067",
  "cost_breakdown_bps": { "treasures_fee_bps": 10, "dex_swap_fee_bps": 30, "estimated_slippage_bps": 45 },
  "expires_at": 1730000060,        // signing deadline — sign + submit before it
  "signable_payloads": [
    { "type": "evm_eip712_typed_data", "typed_data": { "domain": {}, "types": {}, "primaryType": "Order", "message": {} } }
  ]
}
```

<a id="base"></a>
## Base (`chain:"base"`) — sign-only, gasless, priced in Base-native USDC

A co-ranked venue that trades **Coinbase B20 tokenized equities** on **Base** (chain id **8453**), priced in **Base-native USDC**. **Execution is byte-identical to the [`eth` venue](#signable-payload-shapes):** the quote returns one `evm_eip712_typed_data` order — sign it with `eth_signTypedData_v4` (the same `signEvm` helper; strip `EIP712Domain`), submit `{ type:"evm_eip712_signature", signature }`, then poll `/status` (`order_hash` while in-flight → `tx_hash` on fill; the server also backfills stuck orders). **You broadcast no trade, pay no trade gas, and need no 8453 RPC to trade.** (Short version: pre-flight trap #7 in [`../SKILL.md`](../SKILL.md).)

Five things differ from `eth` — everything else carries over unchanged:

1. **`base_asset` is `"usdc"`, but not *that* USDC.** Base settles in **Base-native USDC `0x833589fCD6eDb6E08f4c7C32D4f71b54bdA02913`** — a different contract from mainnet's `0xA0b8…eB48`. The `_usdc`-named fields are genuine USDC amounts, so no unit translation is needed (unlike Robinhood's USDG), but **resolve the contract from the leg's `chain`, never from `base_asset`**. Balances on the two chains are not fungible and do not aggregate.
2. **Joins `chain:null` selection once minted + priceable.** Like robinhood, a base cell is co-ranked in the unpinned selection — but only for tickers with minted supply (point 3), and a listing that can't be priced drops out silently, so unpinned base legs are rare in practice today. Pass `chain:"base"` to force the venue. Your `eth_wallet` (the same 0x EOA) is the signer on 8453. A `protocol` pin is optional, but the only valid value is `"coinbase"`; any other → `422 no_routes`.
3. **Supply-gated listings — trust `/stocks/tickers`, not a hardcoded list.** Coinbase deployed its B20 contracts **unminted**, and Treasures hides a cell until real supply exists. Until a ticker is minted it is absent from `available_chains`, its `coinbase` listing block carries `address: null`, and it contributes no price — a buy returns `422 no_routes`. The set grows as Coinbase mints, so **discover it per-ticker at runtime**. (Sells are never supply-gated: an existing holder can always exit.)
4. **One-time allowance on 8453 — and a small ETH float to send it.** The order is filled by a settlement contract that pulls your input token, so the wallet needs an ERC-20 allowance **on Base**: USDC to buy, the B20 token to sell. The spender is `0x111111125421ca6dc452d289314280a0f8842a65` — textually the same address as Ethereum's, but **an Ethereum approval grants nothing on Base**; allowances are per-chain. Sending that `approve()` is an ordinary Base transaction, so keep a **small ETH float on 8453** (Base runs sub-gwei — a few dollars covers many approvals). Skipping it surfaces as `not_enough_balance_or_allowance`. The trade itself remains gasless.
5. **On `/portfolio` for any wallet when you send your key; cash reads live regardless — but no reconciler, so you MUST submit to get a `/trades` row.** `/portfolio` reports B20 positions (`chain:"base"`, `protocol:"coinbase"`) and your Base USDC cash under **`usdc.base`** — Base-native USDC, a *different contract* from mainnet USDC. Positions are read for **any** wallet when you send your `tik_` integrator key (and otherwise for every wallet Treasures knows on any chain) — snapshotted for 30 s and invalidated when a trade settles — so a token transferred in directly shows exactly like one bought here; an ANONYMOUS call for an `eth_wallet` Treasures has never seen gets an empty `positions` list and is not flagged `partial`, so **send the key**. **Cash is never gated** — it still reads live even for an unknown wallet. Native ETH on 8453 is not reported — read that from your own RPC. A Base trade you never submit is still absent from `/trades` and `/settlements` with no cost basis, regardless of what `/portfolio` shows. Submitted trades do get on-chain-observed cost basis + P&L and appear in both.

**Funding is out of scope:** hold **USDC on Base** to buy, or **B20 tokens** to sell. `/bridge/*` moves USDC between **Solana and Ethereum only** — it cannot reach 8453, so getting funds onto Base is out-of-band. Buy and sell are both supported; a sell trades B20 → USDC, sized against your on-chain 8453 balance.

The single quote leg (`chain:"base"`, `protocol:"coinbase"`):

```jsonc
{
  "quote_index": 0,
  "chain": "base",
  "protocol": "coinbase",
  "base_asset": "usdc",            // Base-native USDC — resolve the contract from `chain`, not this
  "amount_base": "100",            // USDC spent on 8453
  "price_usdc_per_share": "325.98",
  "estimated_output_shares": "0.3067", "estimated_output_tokens": "0.3067",
  "cost_breakdown_bps": { "treasures_fee_bps": 10, "dex_swap_fee_bps": 30, "estimated_slippage_bps": 45 },
  "expires_at": 1730000060,        // signing deadline — sign + submit before it
  "signable_payloads": [
    { "type": "evm_eip712_typed_data", "typed_data": { "domain": { "chainId": 8453 }, "types": {}, "primaryType": "Order", "message": {} } }
  ]
}
```

<a id="speed"></a>
## `priority: "speed"` — one on-chain swap on Robinhood Chain or Base

Send `priority: "speed"` on `/quote/buy`, `/quote/sell` or `/quote/preview` and the `robinhood` (4663) and `base` (8453) legs come back as the **speed route**: a single on-chain swap that settles in one block, instead of the gasless order those venues return by default. `sol` legs are unchanged — Solana already settles in seconds. **`eth` legs are never returned under `priority: "speed"`**: eth has no fast route (its relayed order can take minutes to fill), so it is not a candidate.

It is a **filter, not a preference**. Under `"speed"` the only legs you can receive are `sol` and speed-route `robinhood`/`base`. A `robinhood`/`base` leg the speed route can't serve is **absent** from `quotes[]` — it is never handed back on its gasless route instead — and a `speed_route_unavailable` entry in `warnings[]` says which chain and why. Every leg you do receive is executable as returned. Pinning `chain: "eth"` (or `["eth"]`) together with `"speed"` is a contradiction and is refused up front as `400 invalid_request` (`priority: "speed" is not available on eth`); a `chain` set that also names other chains simply loses `eth`. **`gasless` is present on every leg of every quote**, speed or not, so you can branch on it without inspecting the payload type.

**What you take on, in exchange for one-block settlement:**

1. **You fund the gas.** The leg is `gasless: false` and carries `estimated_gas_usd`. The wallet needs native ETH on that chain — 4663 or 8453 — or the leg fails at submit with `insufficient_native_gas`. Base runs sub-gwei; the estimate is over a live gas market, and the transaction you sign is what bounds the spend.
2. **No allowance to set up front — the quote bundles it.** The swap pulls your input token (USDC/USDG to buy, the stock token to sell) through the router named in the swap payload's `approval_spender`. If the wallet has not approved that router on that chain yet, the leg's `signable_payloads` has **two** entries: `role: "approve"` first (nonce n — an ERC-20 `approve(router, max-uint256)` of the spent token, `approval_spender: null`) and `role: "swap"` second (nonce n+1). An approved wallet gets the single `swap` payload. Branch on `role`, never on position or count alone — and **sign and return every payload the leg issued**; sending only the swap for a two-payload leg is `incomplete_submit` (400) before anything is sent. The approve is max-uint, so it happens once per wallet, token and chain; your next speed quote there is single-payload.
3. **You sign whole transactions — and you do not broadcast them.** Each payload is a complete unsigned transaction (`evm_eip1559_tx`, or `evm_legacy_tx` on 4663): nonce, gas limit and fees are already in the bytes. Sign each `tx_hex` **as-is** with `signTransaction` and send back one `{ "type": "evm_signed_tx", "signed_tx_hex": "0x..." }` per payload. Do **not** re-estimate gas, set your own nonce, or re-serialize — the server checks each signed transaction against the one it issued (matching by nonce) and refuses a modified one as `invalid_signature`. It then re-checks your balance and native gas (summed over both transactions on a pair), simulates, and broadcasts: on a pair it sends the approve, waits for its receipt (a block or two), then sends the swap. `tx_hash` is the swap's from the moment the leg is accepted, so on a pair it becomes visible on chain only after the approve mines.
4. **One in-flight speed trade per wallet per chain.** The nonce is read when the quote is built, so a second speed trade signed while the first is still in flight collides: the loser fails with `nonce_conflict`. Wait for the first trade's status to go terminal, then re-quote — a fresh nonce is only issued by a new quote.
5. **The signing window is shorter (~40 s)** and the price is pinned at that moment. A `swap_reverted` late in that window means the market moved past the quote's minimum return before the transaction was sent — nothing was spent, so re-quote. On a two-payload leg, `approve_failed` means the approve reverted or did not land within 10 minutes and the swap was never sent — re-quote (if the approve did mine, the new quote is single-payload). On `tx_dropped` re-quote and re-sign — that code is only written after the transaction was accepted, re-sent once, and then missing from both the chain and the mempool on **two consecutive** checks, so a transaction merely held in a private mempool is never called dropped.

**Status differs from the gasless venues.** A speed-route leg reports `tx_hash` from the moment it is accepted (there is no order hash and no `order_hash → tx_hash` flip); `/status` moves it to `completed` or `failed` once the receipt lands, and the server keeps polling stuck rows on its own. On a two-payload leg the swap is sent by the server once the approve has mined — your `/status` polls drive that (keep polling at `poll_after_ms`), and `tx_hash` shows on chain a block or two after submit.

**`speed_route_unavailable`** entries carry `reason` and, except when the whole route is switched off, the `chain` whose leg was dropped:

| `reason` | What to do |
|---|---|
| `insufficient_balance` | Fund the spent token on that chain, then re-quote |
| `no_routes` | No speed route for this pair/size right now — take another returned leg, or re-quote without `priority` for that chain's gasless route |
| `provider_error` | Transient upstream failure — retry, or take another returned leg |
| `disabled` | The speed route is off server-side (no `chain` — it is not per-leg); only `sol` legs can be returned. Re-quote without `priority` for the gasless routes |

When the drop leaves nothing to return, the quote is `422 no_routes` instead — with `reason: "insufficient_balance"` when your own wallet is what refused it, otherwise bare (a 422 carries no `warnings[]`). A sell whose holdings sit only on `eth` is `422 holdings_insufficient` under `"speed"`: nothing on the speed chains can cover it.

**`speed_route_unpriced_gas`** is the opposite disclosure and a separate code for that reason: the named `chain`'s leg **was** served on the speed route, but no USD gas price was available, so its `estimated_gas_usd` is `null` and the route ranking ignored gas for it. It carries `chain` and no `reason`. Price the gas yourself before choosing that leg over a gasless one — the leg is fully executable either way.

If you pinned `chain` to a single venue and the *only* problem is your own wallet's balance, the request can refuse with `422 no_routes` and `reason: "insufficient_balance"` instead — same fix, one round trip earlier. A missing allowance never refuses on either shape: the quote carries the approve.

<a id="user-operation"></a>
### `execution: "user_operation"` — the speed route as a sponsored UserOperation

Add `execution: "user_operation"` beside `priority: "speed"` (it is meaningless without it — `400 invalid_request`) and a `robinhood` / `base` speed leg comes back as **one `evm_calls` payload** instead of raw transactions: the same one or two calls (`approve` then `swap`, or just `swap`), with `sender` (your wallet), `chain_id`, `entry_point` (ERC-4337 EntryPoint v0.7) and `approval_spender`. The leg is `gasless: true` with `estimated_gas_usd: "0"` — **your paymaster pays**, not the wallet, so no native ETH float is needed and there is no nonce to collide on. `execution: "transaction"` (or omitting it) is the raw-transaction shape above, unchanged. The flag scopes the **speed-route legs only**: an unpinned fan-out still ranks `sol` alongside, so a `user_operation` response can contain a `sol` leg with its usual `solana_versioned_tx` payload beside the `evm_calls` one — sign each payload as its own `type` says.

**Requirements.** (1) The `eth_wallet` must be **EIP-7702-delegated to Alchemy Modular Account v2** (v1.0.0 `0x69007702764179f14F51cdce752f4f775d74E139` or v1.1.0 `0x77021100bD87b7008E5E1989d0eB38555d0d0000`) **on that chain** — checked at quote time via `eth_getCode`. Pinned and not delegated: `400 wallet_not_delegated` naming the chain; unpinned: that chain's leg is dropped with `speed_route_unavailable` / `reason: "wallet_not_delegated"`. Delegating for the first time is your side (`eip7702Auth`); the quote never carries an `initCode`/`factory`. (2) The op must be **sponsored** — a `paymaster` is required; an unsponsored op is refused as `invalid_signature`. (3) **`base` only today**: `robinhood` has no bundler on this lane yet — pinned it is `422 no_routes`, unpinned it drops with `reason: "disabled"` naming the chain.

**Build, sign, submit.** From `evm_calls`, build the UserOperation with your account SDK (e.g. `@account-kit`): `execute(to, 0, data)` for one call, `executeBatch([{target, value:0, data}, …])` **in the order given** for two; run your paymaster's sponsorship; sign under `chain_id` and the v0.7 EntryPoint; return `{ "type": "evm_user_operation", "user_operation": { sender, nonce, callData, callGasLimit, verificationGasLimit, preVerificationGas, maxFeePerGas, maxPriorityFeePerGas, paymaster, paymasterVerificationGasLimit, paymasterPostOpGasLimit, paymasterData, signature } }` (unpacked v0.7 fields, numerics as decimal or hex strings, ≤ 16 KB, no `factory`/`factoryData`). Treasures decodes `callData` and binds each call **byte-for-byte** to the quote (plus `sender`, zero `value`, a non-empty `paymaster`); a mismatch is a per-leg `invalid_signature` and nothing is submitted. It then re-checks your balance (and the allowance, on a single-call leg), **simulates the op on the bundler** (`eth_estimateUserOperationGas`) and submits it. A simulation refusal never reached the mempool: `nonce_conflict` (the account nonce moved — re-quote and re-sign), **`sponsorship_rejected`** (your paymaster declined: deposit, stake, rate limit, policy signature or a paymaster revert — fix the sponsorship, re-sign), `swap_reverted` (re-quote), `provider_error` (retry).

**Status.** The accepted leg is `status: "broadcast"` with **`user_op_hash`** set and `tx_hash: null`; `/status` polls the bundler for the op's receipt (keep polling at `poll_after_ms`) and flips the leg to `completed` — `tx_hash` then carries the **bundle transaction** the op was included in — or `failed` + `swap_reverted` if the op reverted on chain. An op the bundler drops from its pool without mining is `failed` + `tx_dropped` after 10 minutes; Treasures never re-sends or replaces a UserOperation — that is yours. `user_op_hash` is `null` on every other kind of leg. It is a `/status` key: the reporting reads (`GET /trades`, `GET /settlements`) key on `tx_hash`, and because a bundle can carry more than one of your ops, two rows there may legitimately share one `tx_hash` with different outcomes — `/status` is where the two are told apart.

**Idempotency** is unchanged (a hash over the submitted array), and the op's own hash is deduped across quotes — the same signed op under a second `quote_id` is `409 leg_already_submitted` carrying `user_op_hash` — as is a re-submit of the same leg under a **different signature** (the op hash does not cover the signature, so it is the same op you already own).
