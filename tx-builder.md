# tx-builder reference

`https://tx-builder.avantisfi.com` — GET-only HTTP service that ABI-encodes Avantis `Trading` and `USDC` calls into ready-to-sign `{ to, data, value }` payloads. No signing, no broadcasting, no RPC.

- **Live spec**: <https://tx-builder.avantisfi.com/openapi.json>
- **Swagger UI**: <https://tx-builder.avantisfi.com/docs>

## Response envelope

Every call returns:

```json
{
  "ok": true,
  "data": {
    "to":          "0x44914408af82bC9983bbb330e3578E1105e11d4e",
    "from":        "0x1111...1111",
    "data":        "0x19cde9a1...",
    "value":       "0x13e52b9abe000",
    "chainId":     8453,
    "description": "Open long BTC/USD 10x with 100 USDC (market)",
    "meta":        { "...": "endpoint-specific" }
  }
}
```

- `to`, `data`, `value` — the fields needed to sign. `value` is `0x`-prefixed wei; convert with `BigInt(value)`.
- `from` — who must sign (the delegate when `delegate=` is set, otherwise the trader).
- `chainId` — always `8453` (Base mainnet).
- `nonce` / `gas` — never returned. Caller's wallet manages them.
- `meta` — endpoint-specific context. `/trade/open` includes a `validation` block (see below).

Errors:

```json
{ "ok": false, "error": { "code": "BAD_REQUEST", "message": "...", "details": { ... } } }
```

Codes: `BAD_REQUEST`, `VALIDATION_ERROR` (with `details.fieldErrors`), `UPSTREAM_ERROR`, `NOT_FOUND`, `INTERNAL_ERROR`.

## Endpoint index

| Endpoint | Encodes |
| --- | --- |
| `GET /trade/open` | `Trading.openTrade(t, type, slippageP)` |
| `GET /trade/close` | `Trading.closeTradeMarket(pairIndex, index, amount)` |
| `GET /trade/cancel` | `Trading.cancelOpenLimitOrder(pairIndex, index)` |
| `GET /margin/update` | `Trading.updateMargin(...)` |
| `GET /tpsl/update` | `Trading.updateTpAndSl(...)` |
| `GET /delegate/set` | `Trading.setDelegate(delegate)` |
| `GET /delegate/remove` | `Trading.removeDelegate()` |
| `GET /token/approve` | `USDC.approve(spender, amount)` |
| `GET /pairs` | All pairs (index + symbol + Lazer flag) |
| `GET /pairs/:index` | Single pair detail |
| `GET /addresses` | Avantis contract addresses on Base |
| `GET /health` | Liveness |
| `GET /docs` | Swagger UI |
| `GET /openapi.json` | OpenAPI 3.1 spec |

Any of `/trade/*`, `/margin/update`, `/tpsl/update` accepts `&delegate=0x...` to wrap the inner call in `Trading.delegatedAction(trader, calldata)`. The response's `from` becomes the delegate.

## `GET /trade/open`

```
GET /trade/open
  ?trader=0x...                  # required
  &pair=BTC/USD                  # OR &pairIndex=1 (one required)
  &side=long                     # or short
  &orderType=market              # market | limit | stop_limit | market_zero_fee (default market)
  &collateralUsdc=100            # required, > 0
  &leverage=10                   # required, 0 < x <= 1000 sanity cap
  &slippagePercent=1             # default 1, 0 < x <= 100
  &openPrice=80000               # required for limit/stop_limit; optional market override
  &takeProfit=85000              # optional
  &stopLoss=70000                # optional
  &delegate=0xDELEGATE           # optional, wraps in delegatedAction
  &executionFeeEth=0.0005        # optional override (default 0.00035 ETH, max 1 ETH)
  &skipValidation=true           # optional, default false
```

### Order types

| `orderType` | On-chain enum | Notes |
| --- | --- | --- |
| `market` | `0` | Fixed-fee path; `openPrice` auto-resolved if omitted |
| `stop_limit` | `1` | `openPrice` required |
| `limit` | `2` | `openPrice` required |
| `market_zero_fee` | `3` | ZFP path — uses `pnlMin/MaxLeverage` envelope |

### Validation (default on)

The service consults `data.avantisfi.com/v2/trading` and rejects with `400 BAD_REQUEST` when:

1. **Pair not listed** — `isPairListed === false`.
2. **Position too small** — `collateralUsdc × leverage < pairMinLevPosUSDC`.
3. **Leverage out of envelope** — for `market_zero_fee` orders the ZFP envelope `pnlMin/MaxLeverage` is enforced; for everything else the fixed-fee `min/maxLeverage` is.
4. **Insufficient liquidity** — `positionSize > min(pairMaxOI − pairOI, groupMaxOI − groupOI)`.

`meta.validation` carries the computed envelope:

```json
"validation": {
  "positionSizeUsdc": 1000,
  "pairAvailableUsdc": 33683421.1,
  "groupAvailableUsdc": 31539778.34,
  "availableUsdc": 31539778.34,
  "minLeverage": 1,
  "maxLeverage": 75,
  "minPositionUsdc": 100,
  "isZfp": false
}
```

`skipValidation=true` disables all four checks for advanced callers.

## `GET /trade/close`

```
GET /trade/close
  ?trader=0x...                  # required
  &pairIndex=1                   # required
  &tradeIndex=0                  # required (from /user-data positions[].index)
  &collateralUsdc=100            # required; pass full collateral for full close
  &delegate=0x...                # optional
  &executionFeeEth=0.0003        # optional
```

`value` of the returned tx is the execution fee in wei.

## `GET /trade/cancel`

Cancels an unfilled limit / stop-limit order.

```
GET /trade/cancel
  ?trader=0x...
  &pairIndex=1
  &tradeIndex=0
  &delegate=0x...                # optional
```

`value` is `0x0`.

## `GET /margin/update`

Deposit or withdraw collateral on an open trade.

```
GET /margin/update
  ?trader=0x...
  &pairIndex=1
  &tradeIndex=0
  &action=deposit                # deposit | withdraw
  &collateralUsdc=50
  &priceUpdateData=0x...         # optional; Pyth update bytes
  &priceSourcing=0               # optional; 0 = PYTH_CORE/HERMES, 1 = PYTH_LAZER/PRO
  &delegate=0x...                # optional
```

If `priceUpdateData` is omitted the service fetches it from `feed-v3.avantisfi.com`. `priceSourcing` is auto-detected from the pair's Lazer status when not provided. `value` is `0x1`.

To run fully offline (no upstream HTTP), supply both `priceUpdateData` **and** `priceSourcing`.

## `GET /tpsl/update`

Set or change take-profit / stop-loss on an open trade.

```
GET /tpsl/update
  ?trader=0x...
  &pairIndex=1
  &tradeIndex=0
  &takeProfit=100000             # required, > 0
  &stopLoss=70000                # required, 0 clears the stop
  &priceUpdateData=0x...         # optional
  &priceSourcing=1               # optional
  &delegate=0x...                # optional
```

Same Pyth-bytes auto-fetch behavior as `/margin/update`. `value` is `0x1`.

## `GET /delegate/set` and `GET /delegate/remove`

```
GET /delegate/set?trader=0x...&delegate=0x...
GET /delegate/remove?trader=0x...
```

`from` is the trader (only the trader can grant or revoke). `value` is `0x0`.

## `GET /token/approve`

```
GET /token/approve
  ?trader=0x...                  # required
  &amountUsdc=100                # optional; omit for unlimited (uint256.max)
  &spender=0x...                 # optional; defaults to TradingStorage
```

`to` is the USDC contract; `value` is `0x0`.

## Catalog reads

- `GET /pairs` → `data: [{ index, symbol, from, to, lazerStable }, ...]`
- `GET /pairs/:index` → full pair detail (same record as `data.avantisfi.com` for that index)
- `GET /addresses` → `{ chainId, Trading, TradingStorage, USDC, PairStorage, PairInfos, PriceAggregator, Multicall, Referral }`
- `GET /health` → `{ status: "ok", chainId: 8453 }`

## Validation errors

| `error.code` | When |
| --- | --- |
| `VALIDATION_ERROR` | Query-string shape problems (bad address, missing field, out-of-range numeric). `details.fieldErrors` is populated. |
| `BAD_REQUEST` | Pre-trade check failed (delisted, min position, leverage envelope, liquidity), or domain rule violated (e.g. `takeProfit=0`). |
| `UPSTREAM_ERROR` | `data.avantisfi.com` or `feed-v3.avantisfi.com` returned non-2xx. |
| `NOT_FOUND` | Path not handled, or `/pairs/:index` lookup missed. |
| `INTERNAL_ERROR` | Anything else. Report. |

## Sanity bounds

The service caps a few inputs server-side to catch typos. These are looser than the per-pair limits — the per-pair envelope still applies.

| Field | Server cap | Notes |
| --- | --- | --- |
| `leverage` | `<= 1000` | Per-pair max is stricter; see `meta.validation.maxLeverage` |
| `slippagePercent` | `<= 100` | |
| `executionFeeEth` | `<= 1` ETH | Default `0.00035` |
| `priceUpdateData` | `<= 16 KB` | URL length limits the practical max |

## Pair separators

`pair` accepts any of `/`, `-`, or `_`. `BTC/USD`, `btc-usd`, `eth_usd` all resolve.

## Addresses

All address inputs (`trader`, `delegate`, `spender`) are validated EIP-55. Both checksummed and all-lowercase are accepted; the service normalises to checksum in the response.

## Constraints

- All endpoints are **GET-only**. `value`, `data`, etc. are encoded as `0x` hex strings to survive JSON without precision loss.
- The pairs catalog is cached for 5 minutes by default. Expect a ~200 ms cold-load on first hit.
- The service is read-only with no side effects. CORS is `*`.
