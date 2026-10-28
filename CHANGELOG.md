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

## 2026-10-28: b2b `1.21.0`

A wallet with no code on Robinhood Chain or Base can use `execution: "user_operation"` from its
first trade there: the quote is served, and that first UserOperation carries the wallet's EIP-7702
delegation. Additive: a `1.20.0` caller whose wallet is already delegated sees an unchanged API.

### treasures-b2b-api `1.21.0`: first-op delegation on the user-operation route

- **A wallet with no code is quoted.** Under `priority: "speed"` + `execution: "user_operation"`,
  `/quote/buy` and `/quote/sell` now serve a wallet that has never delegated on that chain.
  `400 wallet_not_delegated` (and the `speed_route_unavailable` reason of the same name) now means
  only a wallet delegated to code other than Modular Account v2.
- **`eip7702Auth` on that first op.** Send the wallet's signed EIP-7702 authorization beside the op
  as `user_operation.eip7702Auth` `{ address, chainId, nonce, r, s, yParity }`, spelled as your SDK's
  bundler call spells it (SDKs with a 7702 mode attach it on their own). It must name a Modular
  Account v2 delegate, carry the leg's `chain_id` (never `0`), be a low-s signature and be signed by
  `sender`; anything else is `invalid_signature`. It does not change the op hash. Omit it once the
  wallet is delegated. An authorization the chain skips (a stale `nonce`) leaves the wallet without
  code, and the op is refused as `swap_reverted`: re-quote and re-sign.
- **Robinhood Chain on this route.** The skill's pre-flight list no longer says Robinhood Chain is not
  served under `execution: "user_operation"`; it is, as the trading reference already said.

## 2026-10-26: b2b `1.20.0`

Catches the published b2b skill up with the live API: BNB Smart Chain (`1.18.0`), bridging between
any two chains (`1.19.0`), and SOL fees paid by a wallet you fund (`1.20.0`). `1.18.0` and `1.19.0`
never shipped on their own. The `openapi.yaml` published with the skill now also carries the
Signals API and `POST /trade/submit/batch`.
Additive on the request: a `1.17.0` caller that sends none of the new fields sees an unchanged API.
Three **⚠ Action** items:

- **Robinhood Chain approvals:** a gasless order there needs a one-time allowance to
  `0x5A705DE8982235a7fa45bB83dCaCf03a211389C7` (Robinhood Chain's own settlement contract, not the
  `0x1111…0a65` used elsewhere), sent with a little ETH on 4663. Earlier versions said none was needed.

- **Strict response parsers:** `chain: "bnb"` and `protocol: "bstocks"` can appear on held or routed
  rows, `/portfolio` `usdc` adds `arbitrum` and `bnb` keys, and every executable quote leg carries
  its own `expires_at`. Parse `chain` and `protocol` as open vocabularies (as `1.17.0` already said).
- **Echo `quotes[].quote_index`, never the array position.** A buy alternative that has already
  expired when the quote is built is dropped, so `quote_index` can skip a value. This was always the
  contract; a caller that indexed by position now gets the wrong leg.

### treasures-b2b-api `1.18.0`: BNB Smart Chain (56)

- **`chain: "bnb"`** on `/quote/buy`, `/quote/sell`, `/quote/preview` and the `/settlements`
  filters. 56 carries up to three issuers: `ondo` and `xstocks` (their third
  deployment) and **`protocol: "bstocks"`**, which lists on 56 alone. Read each leg's or position's
  `protocol`: on 56, `chain` alone does not name a cell. Quote `chain` and `preferred_chain` arrays
  now take up to six values, `protocol` arrays up to seven.
- **Not open to every caller yet.** While the venue is off for you, nothing about it reaches the
  wire: no `bnb_address` key, no `bstocks` block, no `bnb` in `available_chains`, and a `bnb` buy is
  `422 no_routes`. Selling is never gated: a 56 position you hold stays on `/portfolio` and quotes on
  `/quote/sell`.
- **Two signing models.** `ondo` and `xstocks` legs on 56 use the same sign-only, gasless model as
  `eth`, with one-time approvals paid in **BNB**, not ETH. A `bstocks` leg is a complete transaction
  at every priority (an `approve` first when one is due), broadcast by Treasures and paid in BNB; it
  is the one 56 cell served under `priority: "speed"`. `origin_chain` / `payout_chain` do not take 56.
- **The base currency is 18-decimal Binance-Peg USDC**, reported as `base_asset: "usdc"`. It is not
  Circle USDC and is not fungible with USDC on any other chain: never pool `usdc.bnb` with the rest.
- **Short signing window.** A `bnb` `ondo` / `xstocks` leg expires about 14 seconds after the quote (a
  `bstocks` leg about 40). Every executable
  leg now carries its own `quotes[].expires_at`; the top-level `expires_at` is still the earliest
  leg's, and `/trade/submit` enforces the deadline of the leg you signed (a sell signs every leg, so
  it is bound by the earliest).
- **Discovery.** `/stocks/tickers` and `/stocks` add `bnb_address` and `bnb_min_trade_size_usd` on the
  `ondo` and `xstocks` blocks and a `bstocks` block (`address`, `share_multiplier`, `token_ticker`
  `<TICKER>B`); `/stocks/prices` adds `onchain.bstocks`. Listing is a weak signal on 56: many bStocks
  tokens have no price at the execution venue and refuse with `422 no_routes` /
  `reason: "no_price_feed"`, so read `tradability` and `min_trade_size_usd` first.
- **`/portfolio`** reports 56 positions and `usdc.bnb` (and documents `usdc.arbitrum`), even while the
  trading venue is off for you.

### treasures-b2b-api `1.19.0`: bridge between any two chains

- **`/bridge/quote` takes any two distinct chains** among `sol`, `eth`, `robinhood`, `base`,
  `arbitrum` and `bnb`. Each side settles in that chain's base currency (USDG on `robinhood`,
  Binance-Peg USDC on `bnb`). `bnb` answers `400 unsupported_route` where BNB Smart Chain is not
  supported.
- **`sol_wallet` and `sol_signature` are optional** when neither side is `sol`: an eth-only
  ownership proof covers an EVM-to-EVM bridge.
- **`delivery: "submit"`** (optional; default `"self_broadcast"`, the response you already parse)
  quotes for the new **`POST /bridge/{bridge_quote_id}/submit`**. From `eth`, `base` or `arbitrum` you
  sign one `evm_permit_typed_data` payload (each of its one or two `signatures[]` items, in order): no
  approval, no ETH and no nonce. From `sol` you sign the
  transaction and Treasures broadcasts it. Other origins answer `400 unsupported_route` for now.
  Resubmitting the same body is safe for an hour.
- New refusals on `/submit`: `400 quote_not_submittable`, `400 signed_tx_mismatch`,
  `409 bridge_already_submitted`, `410 bridge_quote_expired`, `422 broadcast_rejected` and
  `502 broadcast_unavailable` (resubmit the same body; never sign a second deposit).
- **Funding a venue chain.** The Robinhood Chain, Base and BNB Smart Chain sections no longer call
  getting USDC there out-of-band: `/bridge/quote` reaches all three.

### treasures-b2b-api `1.20.0`: SOL fees paid by a wallet you fund (`sol_fee_payer`)

- **`sol_fee_payer`** on `/quote/buy` and `/quote/sell`: one fixed sol wallet you fund. A wallet
  holding almost no SOL (under about 0.003) is refused on `sol` with `422 no_routes`; with
  `sol_fee_payer`, each same-chain `sol` leg comes back either carrying `sol_fee_payer` (your wallet
  pays the fees and the rent of any token account the trade opens; the leg reads `gasless: false`
  with `estimated_gas_usd`) or as a market-maker fill without it (the maker pays the fee). Every
  `ondo` leg is a market-maker fill, so the field never pays for one: a first buy of an `ondo` token
  still needs about 0.01 SOL in the user's wallet for the new token account.
- **Sign a leg carrying `sol_fee_payer` as issued, with both keys**: the `sol_fee_payer` and the
  user's `sol_wallet`. Never recompile it.
- **Send it on sells as well as buys.** It covers only the quote it rides on: a wallet still under
  about 0.003 SOL is refused on the sell unless the sell carries it too. Once sent, a pool-filled leg
  is only ever the sponsored one, so keep the sponsor funded.
- Requires `sol_wallet` and must differ from it; refused (`400 invalid_request`) beside
  `quote_only: true` and on `/quote/preview`; screened like your wallets. Cross-chain `sol` legs are
  unchanged.

### Corrections and additions in `1.20.0`

- **⚠ Action, Robinhood Chain approvals.** A gasless order on Robinhood Chain needs a one-time
  ERC-20 allowance like every other EVM chain (USDG to buy, the stock token to sell), sent with a
  little ETH on 4663. Its spender is `0x5A705DE8982235a7fa45bB83dCaCf03a211389C7`, Robinhood
  Chain's own settlement contract, not the `0x1111…0a65` used elsewhere. Earlier versions said the
  venue needed no native float at all.
- **Arbitrum (Reality) section.** The live `chain: "arbitrum"` venue is now documented: Arbitrum-native
  USDC, approvals to `0x1111…0a65` on 42161, buys gated on the market week, no speed route, cross-chain leg,
  `origin_chain` or `payout_chain`.
- **Robinhood Chain liquidity.** No longer named as four fillable tickers: read `tradability` and
  `min_trade_size_usd` on the `robinhood` listing block.
- **bStocks on BNB Smart Chain** (not open yet). Earlier text called every 56 leg gasless and said 56
  had no speed route. A `bstocks` leg is a complete transaction paid in BNB at every priority, with a
  signing window of about 40 seconds.

### Spec: Signals API and batch submit

- `openapi.yaml` now carries the `Signals` tag (`/signals/strategies*` and `/signals-orders/*`,
  general integrator key required) and `POST /trade/submit/batch` (up to 25 submits in one request,
  integrator key required). Both were already live; neither changes the skill's own flows.

---

## 2026-10-26: b2b `1.17.0` (content update, version unchanged)

A documentation-only refresh of the `1.17.0` skill: the parameters below are already live on the
API, and the skill text now says so. No `metadata.version` bump, so no gate signal fires: read this
entry.

### treasures-b2b-api: search, sort and paging on `GET /stocks`

- Four optional query parameters: `q` (case-insensitive substring of the ticker, the name or any
  venue's `token_ticker`, 1 to 100 characters), `sort` (`market_cap`, `volume_1d`, `change_24h`,
  `price`, `pe` or `ticker`; a stock missing the metric sorts last), `order` (`asc` or `desc`) and
  `page` (1-based, 25 rows a page). New `StockSortKey` schema.
- Only `page` changes the response shape: it adds `page`, `next_page` (`null` on the last page) and
  `total`. Without any of the four, the response is the whole catalog, as before.
- These four are the only query parameters `/stocks` takes; any other is still
  `400 invalid_request`.

---

## 2026-10-06: b2b `1.17.0`, wallet `1.4.0`

Adds Backpack Securities tokenized stocks as a third stock protocol on Solana, in both skills, and
an optional `origin_chain` on b2b buy quotes and previews. Additive on the request. Two
**⚠ Action** items:

- **Strict response parsers:** `backpack` can appear as a `protocol` or `issuer` on response rows for
  any wallet that holds the tokens, whatever skill version you run. Both skills now state that
  `chain`, `protocol` and `issuer` are open vocabularies: never fail a parse on an unlisted value.
- **Unpinned Solana buys widen:** while the venue is offered, a request that names no `protocol` can
  now route to a listed Backpack cell. To keep the old routing, send `protocol` as an array of the
  protocols you want.

### treasures-b2b-api `1.17.0`: Backpack, a third stock protocol on Solana

- **`protocol: "backpack"`** on `/quote/buy`, `/quote/sell` and `/quote/preview`, alone or in a
  `protocol` array (arrays now take one to six values). Solana only: it pairs with `chain: "sol"`,
  and there is no Ethereum cell.
- **Co-ranked, not opt-in.** An unpinned Solana buy ranks a listed `backpack` cell against `ondo`
  and `xstocks`, and an unpinned sell fills from a held `backpack` position like any other.
- **Discovery.** `/stocks` and `/stocks/tickers` gain an optional `backpack` listing block
  (`BackpackListing`: the `ondo`/`xstocks` shape, `eth_address` always `null`). The key is absent,
  not `null`, while the venue is not offered, and a cell is listed only once its token has supply.
  `/stocks/{ticker}` lists a `backpack` entry in `listings[]` on the same terms. `/stocks/prices`
  carries no `onchain.backpack`: compare a Backpack leg against `tradfi`.
- **Bare `token_ticker`.** Backpack positions report the plain stock symbol (`"MU"`, no suffix) on
  `/portfolio`, `/trades` and `/settlements`, with `protocol: "backpack"` and `chain: "sol"`. A bare
  symbol no longer implies a single-cell venue: branch on `protocol` and `chain`.
- **Cross-chain, like `sol/xstocks`.** Under `priority: "speed"`, a `sol/backpack` cell you are not
  funded for on Solana can be bought from your USDC on another chain as one leg (marked by
  `origin_chain`), and a held Backpack position sells cross-chain with `payout_chain`.
- New refusal: `422 no_routes` on a buy pinned to a `backpack` cell that is not listed.
- `/settlements` filters list every value: `chain` adds `arbitrum`, `protocol` adds `reality` and
  `backpack`.
- `SettlementFeeCost.amount_usd`: the note on which recorded notionals are already net of the fee
  now includes a Solana buy.

### treasures-b2b-api `1.17.0`: rank by where your money is (`origin_chain`)

- **`origin_chain`** (`sol`, `eth`, `robinhood` or `base`) on `/quote/buy` and `/quote/preview`
  names the chain the stable you will pay with sits on. Optional: omit it for today's behaviour
  exactly. `/quote/sell` rejects it as an unknown field.
- **Default route: ranking only.** Every leg on another chain ranks as if it also paid the measured
  cost of moving your money there, so `quotes[0]` is the cheapest leg for money held on that chain.
  No leg is added, dropped or repriced, and a leg on another chain is still a same-chain leg: fund
  that chain (for example through `/bridge/quote`) before you submit it. The transfer cost excludes
  the gas you pay to send the transfer, which matters most on `eth`.
- **Under `priority: "speed"` it also picks the origin.** A cross-chain leg spends only from
  `origin_chain`, so its `origin_chain` field always equals the value you sent. A cell already
  funded on its own chain still trades there; if `origin_chain` does not cover the amount, no
  cross-chain leg is offered.
- **`/quote/preview`** reads no balance, so under `"speed"` it prices as if the whole amount sits on
  `origin_chain` (`sol` when you omit it, as before).
- New refusal: `400 invalid_request` (`origin_chain "eth" requires eth_wallet`) when `/quote/buy`
  names an `origin_chain` whose wallet the request does not carry.

### treasures-wallet `1.4.0`: Backpack on the wallet API

- **`protocol: "backpack"`** on `GET /wallets/:id/quotes` and `POST /wallets/:id/trades`. Solana
  only: pair it with `chain: "solana"` or omit `chain`; with `ethereum` it is `400 invalid_request`.
- An unpinned buy resolves across `{solana, ethereum} × {ondo, xstocks}` plus `solana × backpack`
  when that venue lists the asset. A sell fans out across every venue the wallet holds, Backpack
  included.
- **`/balances`:** `issuer` adds `backpack` and `reality`, `chain` and `native` add `arbitrum`, and
  `needs_funding` is true only when every `native` entry is zero. Only `sol`/`eth` positions with
  issuer `ondo`, `xstocks` or `backpack` trade on this plane; the rest are reported for
  completeness. A Backpack position carries the bare ticker as its token symbol on the reads routes.

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
