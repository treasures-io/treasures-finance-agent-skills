# Validated examples

Real calls against the current `/api` + `/public/v1` paths (mainnet, this session). Replace `$KEY`
(`twk_…`, keep secret) and `$WID` (`wlt_…`). `HOST` is `https://api.treasures.io`;
`API=$HOST/api/v1`, `READS=$HOST/public/v1`.

```bash
HOST=${HOST:-https://api.treasures.io}
KEY=twk_…; WID=wlt_XD825…; API=$HOST/api/v1; READS=$HOST/public/v1
SOL=3HW73Fk…; ETH=0xaF51…   # from GET $API/wallets/$WID → addresses
```

## Resolve addresses + balances (no key)
```bash
curl -s "$API/wallets/$WID"           # → { addresses:{evm,solana}, signers, policies, ... }
curl -s "$API/wallets/$WID/balances"  # → { native, stablecoins, positions, needs_funding, as_of }
curl -s "$API/wallets/$WID/delegation" # → { app_enabled, signers }
```

## Quote (X-API-Key, scope quote)
```bash
# auto-route $10 NVDA buy → resolves to the single best cell (here solana/ondo)
curl -s "$API/wallets/$WID/quotes?side=buy&asset=NVDA&notional_usdc=10&slippage_bps=100" -H "x-api-key: $KEY"
# → {"chain":"solana","protocol":"ondo","side":"buy","asset":"NVDA","max_amount_in":"10000000","min_amount_out":"47382065","route_type":"dex_aggregator"}
#   a warned cell also carries "tradability":"thin"|"untradable", "warn_reason":"…", "thin_since":"…" —
#   advisory, absent when unwarned; act per endpoints.md#tradability before trading

# pinned cell, sell quote
curl -s "$API/wallets/$WID/quotes?chain=ethereum&protocol=xstocks&side=sell&asset=NVDA&shares=0.047&slippage_bps=100" -H "x-api-key: $KEY"
# → {"chain":"ethereum","protocol":"xstocks",...,"min_amount_out":"9302939",...}   (USDC atomic, 6dp)
```

## Buy + poll (X-API-Key, scope trade)
```bash
IDK=$(uuidgen)
curl -s -X POST "$API/wallets/$WID/trades" -H "x-api-key: $KEY" \
  -H "Idempotency-Key: $IDK" -H 'content-type: application/json' \
  -d '{"side":"buy","asset":"NVDA","size":{"notional_usdc":"10"},"slippage_bps":100}'
# → 202 {"job_id":"job_…","state":"routing","chain":"sol","protocol":"ondo",...}
curl -s "$API/wallets/$WID/trades/job_…" -H "x-api-key: $KEY"  # same trade key — poll until confirmed|failed|rejected
# confirmed → {"state":"confirmed","result":{"amount_in":"…","amount_out":"…","tx_sig":"…"}}
```

## Sell across cells → one call, N legs
```bash
# The server splits a sell across every venue the wallet holds. One POST, one Idempotency-Key.
curl -s -X POST "$API/wallets/$WID/trades" -H "x-api-key: $KEY" \
  -H "Idempotency-Key: $(uuidgen)" -H 'content-type: application/json' \
  -d '{"side":"sell","asset":"NVDA","size":{"shares":"0.14"},"slippage_bps":100}'
# → 202 {"order_status":"pending",
#        "legs":[{"job_id":"job_…","chain":"sol","protocol":"ondo","state":"broadcast",...},
#                {"job_id":"job_…","chain":"eth","protocol":"xstocks","state":"routing",...}]}

# Poll EVERY leg — job_id is per leg, and legs settle at different speeds (sol ~2s, eth ~20-40s).
curl -s "$API/wallets/$WID/trades/job_…" -H "x-api-key: $KEY"
# Sum result.amount_out over the confirmed legs for what you received. Some legs confirmed and
# some failed is `partially_filled` — a real outcome, not an error.

# Pinning narrows to ONE venue; same {order_status, legs} shape, one entry. Larger than that cell
# holds → 422 quote_unavailable.
curl -s -X POST "$API/wallets/$WID/trades" -H "x-api-key: $KEY" \
  -H "Idempotency-Key: $(uuidgen)" -H 'content-type: application/json' \
  -d '{"chain":"solana","protocol":"ondo","side":"sell","asset":"NVDA","size":{"shares":"0.09"},"slippage_bps":100}'
```

## Reads (no key)
```bash
curl -s "$READS/trades?sol_wallet=$SOL&eth_wallet=$ETH&source=internal&limit=4"  # executed trades + P&L
curl -s "$READS/portfolio?sol_wallet=$SOL&eth_wallet=$ETH&source=internal"        # positions + unrealized P&L
```

## Onboarding (agent side)
```bash
# mint (no auth) — optionally request scopes/caps for the key
curl -s -X POST "$API/onboarding-sessions" -H 'content-type: application/json' \
  -d '{"api_key":{"scopes":["trade","quote"],"caps":{"max_trade_notional_usd":"100"}}}'
# → 201 {"request_id":"…","device_secret":"…","url":"https://…/onboard?requestId=…","interval":3}
# send `url` to the human; then poll with the device_secret until approved:
curl -s -X POST "$API/onboarding-sessions/poll" -H "Authorization: Bearer <device_secret>" \
  -H 'content-type: application/json' -d '{"request_id":"…"}'
# → {"status":"pending"} … then once {"status":"approved","wallet":{…},"owner_email":…,"api_key":"twk_…"}
```

## Notes confirmed live this session
- New paths: wallet/onboarding under **`/api`**; reads under **`/public/v1`**. Old
  `/public/v1/wallets/...` now **404s**.
- Auto-route is **single-cell** (picks one best cell; sol·ondo won a $10 NVDA buy and a 0.04/0.09 sell).
- A job can **wedge in `broadcast`** (an under-funded $10 buy did) — bound the poll, treat as unknown.
- EVM `confirmed.result.tx_sig` is the **settlement hash**, distinct from the `broadcast` orderHash.
