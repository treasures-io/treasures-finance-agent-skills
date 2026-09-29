# Changelog

Two skills ship from this repo, each with its own independent `metadata.version`:

- **`treasures-b2b-api`** — the public B2B API (discover tokenized stocks, quote, trade, bridge, read portfolio and history).
- **`treasures-wallet`** — delegated wallets (onboard, quote, async buys/sells, balances, API keys).

A skill's `metadata.version` is what the API's version gate reads. The **plugin** version
(`.claude-plugin/plugin.json`) tracks the bundle and moves on every published release.

**To update:** `npx skills add treasures-io/treasures-finance-agent-skills`, or in Claude Code
`/plugin marketplace update`. See the [README](README.md) for all install options.

Every release so far has been **additive on the request** — a caller on an older version keeps
working unchanged. Entries that need action from you are marked **⚠ Action**.

---

## 2026-09-29: b2b `1.16.0`

Folds in b2b `1.15.0`, which did not ship standalone, and publishes integrator fees and payouts for
the first time. Additive on the request: a caller that sends none of the new fields gets the same
routing it got on `1.14.0`. The one **⚠ Action** is for strict response parsers.

### treasures-b2b-api `1.16.0`: sell and receive USDC on another chain

- **`payout_chain` on `POST /quote/sell`** (`"sol"`, `"eth"` or `"base"`; absent or `null` keeps the
  old behaviour). The sale's USDC lands on that chain, in your own wallet there. A `sol/xstocks`,
  `base/coinbase` or `eth/ondo` position held on another chain sells cross-chain as **one leg**,
  marked by `payout_chain` on the leg, whatever the `priority`. A position that cannot pay out there
  is left out with a `cross_chain_route_unavailable` warning; it is never sold on its own chain
  instead. Sell-only: `/quote/buy` and `/quote/preview` answer `400 invalid_request`.
- **You fund the gas on a cross-chain sell leg**, on the position's own chain (`gasless: false`
  whatever the `priority`), unless you send `execution: "user_operation"`, where your paymaster pays. On `eth` / `base` it is always two `evm_signed_tx` payloads, an exact-amount
  `approve` then a `role: "deposit"`; under `execution: "user_operation"` it is one `evm_calls`
  payload holding the same two calls; on `sol` it is a `solana_versioned_tx` that uses
  address-lookup tables.
- **`/quote/{quote_id}/status` legs** gain `payout_chain` and `payout_tx_hash`. On `completed`,
  `payout_chain` names where the USDC actually landed, which is the sale's own chain if the payout
  was returned there.
- **`/settlements`** reports such a sale as two transactions: the sale (`sending`) and the payout
  (`receiving`).
- New refusals: `400 invalid_request` with a `payout_chain: ` message prefix (the payout wallet is
  missing, or nothing in scope can pay out there), and `422 no_routes` with
  `reason: "payout_route_unavailable"`.
- `/quote/{quote_id}/status` asks for `poll_after_ms: 3250` (was `10250`) when only gasless or
  Solana legs remain; their upstream status now syncs every 3 s.
- `/settlements` `dex_fee` on a speed-route leg the server broadcast is measured from the settled
  transaction rather than taken from the quote.

### treasures-b2b-api `1.16.0`: integrator fees and payouts (integrator key required)

- **`integrator_fee_bps` on `/quote/buy`, `/quote/sell` and `/quote/preview`**: your own fee on the
  quote, in net bps, overriding the default Treasures configured for your key, up to your ceiling.
  Omit it to use the default. It is stacked into the same on-wire fee as Treasures' and already
  reflected in `estimated_output_*`; on Solana the venue keeps half of that fee, so the trader is
  charged twice the figure for you to net it.
- Every quote leg reports **`cost_breakdown_bps.integrator_fee_bps`** (`0` on an anonymous quote).
- **A quote carrying your fee must be submitted with the key that quoted it**: anything else is
  `403 quote_integrator_mismatch` and nothing is broadcast. Zero-fee quotes are unaffected.
- New refusals: `400 invalid_integrator_fee` (above your ceiling, or a venue cannot carry it; `reason`
  says which), `400 integrator_fee_requires_api_key`, and `503 integrator_fee_misconfigured`.
- **`/settlements`** gains an `integrator_fee` entry in `fee_costs`, carrying the `payout_id` it went
  out in, and two filters: `payout_id` and `payout_status` (`unpaid` / `processing` / `paid`).
- **Payouts**, general `tik_` key only: `GET /payouts/accrued` (what you are owed now, where it will
  be paid, the limits and `next_eligible_at`), `POST /payouts` (claim everything owed to your
  registered address; empty body, `Idempotency-Key` required), `GET /payouts` and
  `GET /payouts/{payoutId}`. The payout address is set by Treasures only. Refusals include
  `409 payout_in_progress` / `payout_cooldown` / `address_hold` / `payout_address_missing` /
  `nothing_to_pay`, `422 below_minimum` / `requires_review` and `503 payouts_unavailable`.

### treasures-b2b-api `1.15.0`: cross-chain buy legs under `priority:"speed"`

- Under `priority: "speed"`, `/quote/buy` and `/quote/preview` may quote a cell you are **not
  funded on** as a cross-chain leg instead of dropping it: one signature set moves your stable off
  its origin chain and into the destination stock. The leg carries **`origin_chain`** (present only
  on this kind of leg; `chain` stays the destination), `cross_chain_cost_bps` /
  `cross_chain_cost_usdc`, and `base_asset` / `amount_base` in the **origin's** currency.
- **⚠ Action (strict response parsers only):** `cost_breakdown_bps.dex_swap_fee_bps` is now
  nullable. It is `null` only on a cross-chain leg (a `"speed"` buy, or a `payout_chain` sell from
  `1.16.0`), so a caller that sends neither never sees it.
- `chain: ["eth"]` with `"speed"` on `/quote/buy` and `/quote/preview` is no longer an
  unconditional `400`: it returns a cross-chain leg or `422 no_routes`. On `/quote/sell` it is still
  `400 invalid_request`, unless `payout_chain` names another chain.
- Signing: an EVM-origin leg may carry a `role: "deposit"` payload whose paired `approve` is
  **exact-amount**; a sol-origin leg names your own sol wallet as fee payer; an `evm_calls` payload
  may carry `chain_id: 1`.
- New `warnings[]`: `cross_chain_route_unavailable` and `cross_chain_price_deviation` (advisory,
  never a refusal). New per-leg `error_code`s: `refunded`, `deposit_reverted`,
  `delivered_other_currency`, `cross_chain_unsettled` (**do not re-quote**, it may still settle),
  and a per-leg `quote_stale`. `422 no_routes` may carry `reason: "cross_chain_wallet_missing"`:
  the buy could only be served cross-chain from a chain whose wallet you did not send.
- Enums widen for an upcoming venue: `Chain` gains `arbitrum` and `Protocol` gains `reality`, and
  `/stocks` / `/stocks/tickers` gain a `reality` listings block that is `null` until the venue
  opens.
- `/stocks/{ticker}` `onchain.*`: a venue whose on-chain price sits more than 1.5x from the tradfi
  price reads `share_price_usd: null` (volume is still reported).

---

## 2026-09-19 — b2b `1.14.0` (content update, version unchanged)

A documentation-only refresh of the `1.14.0` skill: the API contract below was already live, the
skill text now says so. No `metadata.version` bump, so no gate signal fires — read this entry.

### treasures-b2b-api — `portfolio_busy` and the rewritten rate limits

- **`GET /portfolio` can now answer `503 portfolio_busy`** with a `Retry-After` header
  (delta-seconds). It means the server is already computing as many fresh snapshots as it safely
  can and yours was not cached. Sleep `Retry-After` and retry the same request; a cached snapshot
  is never refused. The `apiFetch` helper's transient-error branch is the right place to handle it.
- **Rate limits, restated.** Anonymous `/portfolio` calls have their own per-IP row (600/min). With
  a general `tik_` key, `/portfolio` is not bucketed by IP at all: your organisation bucket is the
  bound, sized for one request every 5 s per active end-user. The blanket per-IP ceiling across all
  endpoints is now set per environment (always at or above the per-endpoint rows) and no longer
  covers `/portfolio`. `/trades` keeps 60/min.
- **`/portfolio` snapshots refresh on a 30 s cadence** (`as_of`, `is_cached`). Polling faster
  returns the same snapshot.
- **Not included:** the backend spec already carries integrator-fee fields (`integrator_fee_bps`
  and its error codes). They were left out of this publication and ship in `1.16.0`, above.

## 2026-09-11 — b2b `1.14.0`

Folds in b2b `1.11.0`, `1.12.0` and `1.13.0`, none of which shipped standalone. Every item is
additive on the request unless marked **⚠ Action**; the only actions are inside `priority:"speed"`,
a shape that first appeared in `1.11.0`.

### treasures-b2b-api `1.14.0` — `/portfolio` reads any wallet with your key

- **⚠ Action — send your `tik_` key on `GET /portfolio`.** Positions on the two single-cell venues
  (Robinhood Chain 4663, Base 8453) are read for **any** `eth_wallet` when a verified general key is
  presented. Without it, only a wallet Treasures already has a row for is read, and an end-user
  wallet that merely *holds* a token (an airdrop, a transfer in) comes back with an empty
  `positions` list and is not flagged `partial`. Anonymous calls are byte-identical to before; cash
  (`usdc.base`, `usdg.robinhood`) was never gated either way.

### treasures-b2b-api `1.13.0` — sponsored user-operation lane

- Optional `execution` request field on `/quote/buy`, `/quote/sell` and `/quote/preview`:
  `"transaction"` (the default, byte-identical to `1.12.0`) or `"user_operation"`. Only meaningful
  with `priority:"speed"`; sending it alone is `400 invalid_request`.
- Under `"user_operation"` a `base` speed leg returns **one `evm_calls` signable payload** instead
  of a signed-transaction pair. Pack it into a sponsored ERC-4337 UserOperation from an
  EIP-7702-delegated (Modular Account v2) wallet, sign, and submit as `evm_user_operation`. The leg
  is then `gasless: true` with `estimated_gas_usd: "0"`; there is no nonce to collide on.
- New response keys: `user_op_hash` on submit results and `/status` legs (`null` elsewhere). New
  refusals on that lane only: `sponsorship_rejected`, `wallet_not_delegated`. `robinhood` is not
  served on this lane yet.

### treasures-b2b-api `1.12.0` — speed route hardened (actions only if you send `priority:"speed"`)

- **⚠ Action — `role` is now required on every `SignableEvmTx`** (`"approve"` or `"swap"`), and a
  speed leg on an unapproved wallet returns **two payloads** with consecutive nonces: sign and
  return both, in the order issued. Signing only `[0]` is `incomplete_submit`; reordering on a
  replay is `leg_already_submitted`. The bundled approve grants the router an unlimited
  (`max-uint256`) allowance on the token being spent, so surface that consent to the user.
- **⚠ Action — `approval_spender` is nullable** on the speed route (the approve rides inside the
  quote). `allowance_required` left both the `no_routes` reason enum and the
  `speed_route_unavailable` reason list; a client that branched on it can drop that branch.
- **`priority:"speed"` is a filter, not a preference.** `eth` is never quoted under it; a one-element
  `chain:"eth"` (or `["eth"]`) with speed is `400 invalid_request`; a `robinhood`/`base` leg the
  speed route cannot serve is **absent** from `quotes[]` with a `speed_route_unavailable` warning,
  never handed back on its relayed route.
- `approve_failed` on a pair means the approve reverted or did not land in 10 min and the swap was
  never sent (`tx_hash` is `null`). Re-quote.

### treasures-b2b-api `1.11.0` — routing controls, preview, ticker details

- `protocol` on the quote routes accepts an array (1–4, unique: restrict the auto-route to a
  protocol subset, exactly as `chain` already did). New `preferred_chain`: a preference, not a
  pin. The leg on that chain becomes `quote_index: 0` when the plan has one; nothing is added or
  dropped, and naming a chain outside a sent `chain` set is `400 invalid_request`.
- **`priority:"speed"`** asks for the speed route on `robinhood` and `base`: a single on-chain swap
  that settles in one block, returned as a complete unsigned transaction you sign and Treasures
  broadcasts against **your** native gas on that chain. Absent or `null` is unchanged routing.
- New response keys on every leg: `gasless` (who pays this leg's gas) and `estimated_gas_usd`
  (speed-route legs only; `null` with a `speed_route_unpriced_gas` warning when no USD gas price was
  available). New drop reason `speed_route_unavailable`; new submit refusals `insufficient_native_gas`,
  `nonce_conflict`.
- **Two new routes, both `tik_` key required:** `POST /quote/preview` (wallet-less price preview:
  the buy-quote legs minus `quote_id`, `expires_at`, `signable_payloads` and `warnings_ack_token`;
  nothing persisted; capped at 1,000,000 USDC; a route-wide shared rate ceiling, so honour
  `Retry-After`) and `GET /stocks/{ticker}` (profile, listings, tradfi snapshot, extended hours,
  market session, analyst consensus and grades, next earnings, latest news; every block best-effort
  and `null` on its own; cached 60 s; no `onchain` block).

## 2026-09-02 — b2b `1.10.0`, wallet `1.3.0`

Published together. Folds in b2b `1.7.0`, `1.8.0` and `1.9.0`, and wallet `1.2.0`, none of which
shipped standalone.

### treasures-wallet `1.3.0` — withdrawals are no longer size-capped

- `POST /wallets/:id/withdrawals` **no longer enforces a per-transaction or rolling-24h size cap.**
  `422 withdraw_cap_exceeded` (with `cap: per_tx|daily`) and `503 withdraw_cap_unavailable` are gone
  — if you branch on either, that code is now unreachable and can be removed.
- `422 unsupported_withdrawal` remains, for an asset/chain pairing the wallet cannot withdraw.
- Nothing else about the endpoint changes: it still returns an **unsigned** transaction for the owner
  to sign client-side.

### treasures-wallet `1.2.0` — the sell contract, corrected

- **⚠ Action — a sell returns N legs, not one job. If you built a sell flow from `1.0.1` or `1.1.0`,
  it is mis-parsing every sell.** Those releases said the server "does NOT split" a sell, told you to
  reproduce a greedy split client-side, and documented `job_id` at the top level of the `202`. That
  stopped being true on 2026-07-18. The server has since planned and executed the split itself:

  ```json
  { "order_status": "pending",
    "legs": [ { "job_id": "job_s0", "chain": "sol", "protocol": "ondo",     ... },
              { "job_id": "job_s1", "chain": "eth", "protocol": "xstocks", ... } ] }
  ```

  Read `job_id` from each `legs[]` entry and poll them all — legs settle at different speeds, so the
  order is not done when the first one lands. A **buy** keeps the flat top-level shape; branch on the
  `side` you sent, not on inspecting the body. `order_status` is `pending` while any leg is
  non-terminal, else `confirmed` / `partially_filled` / `failed`, and it is a submit-time snapshot —
  re-derive it from the polled legs. `partially_filled` is a real outcome, not an error.
- **Delete the client-side greedy-split helper.** One `POST /trades` with one `Idempotency-Key`
  replaces it. Pinning `chain`+`protocol` still narrows to a single venue and still `422`s
  `quote_unavailable` if that cell holds less than the requested size.
- **The skill now sends `X-Treasures-Skill` / `X-Treasures-Skill-Version`,** so wallet agents enrol
  in the version gate for the first time — deprecation warnings while the skill ages, and a clean
  `426` with upgrade instructions instead of a silent break. Nothing is gated today.

### treasures-b2b-api `1.10.0` — update notifications

- Reads the new **`X-Treasures-Skill-Latest`** response header and tells you once when a newer skill
  exists, pointing here. Informational only: it never blocks, retries, or changes a trade decision.
  Both skills read it, and the wallet skill does so without needing to enrol in the gate.
- `Sunset` may now be **absent** on a deprecation. The deprecation still stands; it simply has no
  published deadline yet. `Link` always carries the upgrade pointer, so follow that regardless.

### treasures-b2b-api `1.9.0` — foreign listings, exchange axis, two-tier quote guardrail

- **Non-US listings are first-class.** Read surfaces carry `currency` and `price_native` beside the
  USD figure. Hong Kong (HKEX) listings resolve today.
- **`exchange` browse axis** on the listing endpoints — filter the catalog by listing venue.
- **Exchange-keyed market sessions.** Sessions follow the listing's own venue: the HKEX lunch break,
  venue-local weekends, and forward-looking holiday calendars per exchange.
- **A warned quote now returns instead of refusing.** Quotes that previously failed closed on a
  missing or stale reference price now come back carrying `warnings[]` and a `warnings_ack_token`.
  Echo the token back on submit to trade on a warned quote. Absent token → the quote proceeds with
  no consent recorded; a present-but-mismatched token is rejected `422`.
- **New warn reason `no_settlements`** — the venue quotes normally but no on-chain settlement has
  been observed for it. Advisory: it never hides or blocks a listing.
- **⚠ Action — two fields that were published as non-nullable can now be `null`.** Both are on the
  B2B surface and both were declared required and non-nullable in the `1.6.0` OpenAPI, so a client
  that modelled them from the published spec was following our documentation. Guard both before
  parsing:
  - `tradfi_reference` on a quote response. An asset with no reference price to check against used
    to be refused outright with `422 reference_unavailable`; it is now priced, returned on a `200`
    with `tradfi_reference: null`, and disclosed as a `no_reference` entry in `warnings[]`. The
    field is still published, so a client that models it does not break — but do not build a flow
    around always receiving a value. `premium_vs_anchor_pct` is omitted when it is null.
  - `market_cap_usd` on the tradfi block — a foreign line or a fund can quote without one rather
    than dropping the whole block.

  Neither field gates or prices a trade. Nothing else in this release loosens a published type.
- **⚠ Action — ticker charset widened** to `^[A-Z0-9][A-Z0-9.]{0,9}$` so HKEX board codes (`700` =
  Tencent) parse. If you validate tickers client-side with a letter-leading pattern, widen it or you
  will reject valid symbols.

### treasures-b2b-api `1.8.0` — `chain` accepts a set

- `POST /quote/buy` and `POST /quote/sell` take `chain` as a single chain (a pin, as before) **or**
  an array of 1–4 unique chains, auto-routing within that subset only — at most one leg per chain.
- `["sol"]` is exactly equivalent to `"sol"`. A member the ticker has no listing on is dropped. An
  empty intersection returns `422 no_routes`.
- `null` or omitted is unchanged: every chain the ticker lists.

### treasures-b2b-api `1.7.0` — tradability advisories

- `tradability`, `warn_reason` and `thin_since` on quote legs, list surfaces and portfolio positions.
- A listing can price normally and still be warned — check the fields before submitting rather than
  inferring health from a successful quote.
- `warn_reason` is an **open vocabulary**. Treat a value you don't recognize as "warned, reason
  unknown", never as an error.
- Ticker-grain `tradability` warns when *every* measured venue for that ticker warns.

### treasures-wallet `1.1.0` — sell preview shape, advisories

- **⚠ Action — the sell preview's real shape is documented for the first time.**
  `GET /wallets/:id/quotes` on a **sell** returns
  `{side, asset, route_type, legs:[{chain, protocol, shares_consumed, max_amount_in, min_amount_out}]}`
  — one entry per listing the sell draws from. A **buy** keeps the flat top-level shape. If you were
  reading `max_amount_in` off the top level of a sell preview, read it off `legs[]` instead.
- `tradability` / `warn_reason` / `thin_since` on buy previews (top level, for the resolved listing),
  on each sell `legs[]` entry (side-neutral reasons only), and on `/portfolio` positions.
- **New behavior #7** — read the preview's tradability before every trade, with a per-reason action
  table in [`references/endpoints.md`](skills/treasures-wallet/references/endpoints.md#tradability).
  Advisories never make a holding unsellable.

---

## 2026-08-25 — b2b `1.6.0`, wallet `1.0.1`

### treasures-b2b-api `1.6.0` — Base holdings on `/portfolio` (corrective)

- **⚠ Action — this retracts guidance from `1.5.0`.** `1.5.0` said Base holdings are *not* on
  `/portfolio` and told you to read them from your own 8453 RPC. They are now reported there,
  including `usdc.base`. **If you followed the `1.5.0` instruction, stop** — reading both and summing
  them double-counts your Base position.

### treasures-b2b-api `1.5.0` — Coinbase B20 venue on Base

- New venue: `chain: "base"` with `protocol: "coinbase"`. Opt-in and chain-pinned — a caller that
  never asks for Base sees an identical API.

### treasures-b2b-api `1.4.0` — `GET /settlements` filters

- Optional query filters on the settlements ledger. Additive on the request: send none and you get
  the unfiltered ledger you already expected.
- **⚠ Action — the settlements cursor changed shape.** It gained a filter fingerprint and is parsed
  by exact arity. A cursor issued by `1.3.0` that you are mid-loop on returns `400 cursor: malformed`;
  restart that pagination from page 1. Cursors live seconds, so this only bites an in-flight loop.

### treasures-b2b-api `1.3.0` — `GET /settlements`

- New endpoint: the settlement ledger. Purely additive.

### treasures-wallet `1.0.1`

- Resync alongside b2b `1.6.0` — Base venue reflected on `/portfolio`.

---

## 2026-07-11 — b2b `1.2.0`

- **Sign-only Robinhood venue.** That opt-in venue's execution contract changed; `sol` and `eth`
  callers were unaffected.

## 2026-07-07 — b2b `1.1.0`

- **Robinhood stock reads** — `chain: "robinhood"`, `protocol: "robinhood"`.
- **Opt-in API version gate.** The b2b skill's fetch helper now sends `X-Treasures-Skill` and
  `X-Treasures-Skill-Version`, and handles the response side: `426 skill_version_unsupported` is a
  hard stop, `Deprecation`/`Sunset`/`Warning` are non-blocking notices. Sending the version header is
  the opt-in — omit it and you are treated as a generic client, never gated, but you also lose the
  early warning. Spec: [`docs/skill-version-compatibility.md`](docs/skill-version-compatibility.md).

## 2026-06-25 — wallet `1.0.0`

- First release of the delegated-wallet skill: onboarding, quotes, async buys/sells, balances,
  API-key management.

## 2026-06-09 — b2b `1.0.0`

- First release of the B2B API skill.

---

## Known limitations

- **The `treasures-wallet` skill does not participate in the version gate.** It sends no
  `X-Treasures-Skill-Version` and does not act on `Deprecation`/`Sunset`/`426`. It is therefore
  treated as a generic client — never gated, but never warned either. Until that lands, this
  changelog is the only place a wallet-skill update is announced. Tracked upstream.
- **No `X-Min-Skill-Version` floor has ever been raised.** It sits at `1.0.0` and every release to
  date has been additive, so no version of either skill has been blocked or deprecated.
