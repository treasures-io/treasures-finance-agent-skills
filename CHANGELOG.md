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

## Unreleased — b2b `1.9.0`, wallet `1.1.0`

Published together. Folds in b2b `1.7.0` and `1.8.0`, which never shipped standalone.

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
