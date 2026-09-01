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

## Unreleased — b2b `1.10.0`, wallet `1.3.0`

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
