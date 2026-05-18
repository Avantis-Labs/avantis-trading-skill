# contracts reference

Compact contract-level facts an agent needs when working with the raw Avantis on-chain surface. tx-builder handles encoding for you — these are useful when reading positions / events directly, debugging, or implementing your own signer.

Chain: **Base mainnet**, chainId `8453`.

## Mainnet addresses

| Contract | Address |
| --- | --- |
| `Trading` | `0x44914408af82bC9983bbb330e3578E1105e11d4e` |
| `TradingStorage` | `0x8a311D7048c35985aa31C131B9A13e03a5f7422d` |
| `PairStorage` | `0x5db3772136e5557EFE028Db05EE95C84D76faEC4` |
| `PairInfos` | `0x81F22d0Cc22977c91bEfE648C9fddf1f2bd977e5` |
| `PriceAggregator` | `0x64e2625621970F8cfA17B294670d61CB883dA511` |
| `USDC` | `0x833589fCD6eDb6E08f4c7C32D4f71b54bdA02913` |
| `Multicall` | `0xA7cFc43872F4D7B0E6141ee8c36f1F7FEe5d099e` |
| `Referral` | `0x1A110bBA13A1f16cCa4b79758BD39290f29De82D` |

Available live from `GET https://tx-builder.avantisfi.com/addresses`.

## Scaling

| Domain | Scale | Example |
| --- | --- | --- |
| Prices, leverage, percentages, slippage | `× 10^10` | `10x` leverage → `100_000_000_000` |
| USDC amounts (collateral, position size, fees) | `× 10^6` | `100 USDC` → `100_000_000` |
| ETH amounts (`value`, execution fee) | `× 10^18` | `0.00035 ETH` → `350_000_000_000_000` |

Surfaces where these scales are **already divided**:

- `data.avantisfi.com/v2/trading` returns human decimals throughout.
- `api.avantisfi.com /v2/history/referral/stats/...` pre-divides USDC by 1e6.

Surfaces that return **raw** values (must divide):

- `core.avantisfi.com /user-data` (all USDC and price fields are stringified raw ints).
- `api.avantisfi.com /v2/history/portfolio/*` (mixed; see [`history-api.md`](history-api.md)).

## `Trade` tuple (openTrade input)

Order matches the on-chain `ITradingStorage.Trade` struct:

| # | Field | Type | Scale |
| --- | --- | --- | --- |
| 0 | `trader` | address | — |
| 1 | `pairIndex` | uint256 | — |
| 2 | `index` | uint256 | — (pass `0` on open; assigned at fill) |
| 3 | `initialPosToken` | uint256 | `× 10^6` (USDC collateral) |
| 4 | `positionSizeUSDC` | uint256 | `× 10^6` (often passed as `0` on open; after open it's repurposed as a timestamp on chain) |
| 5 | `openPrice` | uint256 | `× 10^10` |
| 6 | `buy` | bool | — (`true` = long) |
| 7 | `leverage` | uint256 | `× 10^10` |
| 8 | `tp` | uint256 | `× 10^10` (0 = none) |
| 9 | `sl` | uint256 | `× 10^10` (0 = none) |
| 10 | `timestamp` | uint256 | usually 0; contract overrides |

`Trading.openTrade(t, type, slippageP)` — `type` is the `OpenLimitOrderType` enum below, `slippageP` is `× 10^10` percent.

## Order-type enum

`tx-builder` query → on-chain `OpenLimitOrderType`:

| Skill / query value | On-chain enum | Numeric | Notes |
| --- | --- | --- | --- |
| `market` | `MARKET` | `0` | Fixed-fee path |
| `stop_limit` | `REVERSAL` | `1` | Requires `openPrice` |
| `limit` | `MOMENTUM` | `2` | Requires `openPrice` |
| `market_zero_fee` | `MARKET_PNL` | `3` | Zero-fee perp (ZFP). Uses `pnlMin/MaxLeverage` envelope. Internal contract / API naming sometimes uses `MARKET_ZERO_FEE` interchangeably with `MARKET_PNL` — the wire value is the same. |

`tx-builder` always speaks the skill names; agents do not need to know the contract enum names unless decoding logs.

## `PriceSourcing` enum

Used by `Trading.updateMargin(...)` and `Trading.updateTpAndSl(...)`:

| Value | Name | Source |
| --- | --- | --- |
| `0` | `PYTH_CORE` (a.k.a. Hermes) | Pyth Hermes feed; works for all pairs |
| `1` | `PYTH_LAZER` (a.k.a. Pro) | Pyth Lazer feed; use when `pair.lazerFeed.state === 'stable'` |

`tx-builder` auto-detects this from the pair when omitted.

## Key thresholds

| Constant | Value | Meaning |
| --- | --- | --- |
| Liquidation threshold | `85` (%) | Position liquidates once loss exceeds ~85% of collateral |
| Max stop-loss distance | `80` (%) | `_MAX_SL_P` |
| Max slippage | `80` (%) | `_MAX_SLIPPAGE` |
| Max execution / keeper reward | `10 USDC` | `_MAX_EXEC_REWARD` |
| Default execution fee | `0.00035 ETH` | Used by tx-builder when `executionFeeEth` omitted |
| Margin withdraw threshold | `80` (%) | Margin withdrawals must keep position above ~20% effective collateral |

## Delegation

The Avantis `Trading` contract supports a one-delegate-per-trader model:

| Function | Notes |
| --- | --- |
| `setDelegate(address delegate)` | Replaces any prior delegate. Called by the trader. |
| `removeDelegate()` | Clears the delegate. Called by the trader. |
| `delegations(address trader) view returns (address)` | Read who is the current delegate. |
| `delegatedAction(address trader, bytes calldata)` | Delegate executes any trader-scoped call by wrapping the inner calldata. |

`tx-builder` handles the wrap automatically: pass `&delegate=0x...` to any trade-side endpoint and the inner call is wrapped in `delegatedAction(trader, innerData)` with the delegate as the outer caller (`from`).

For the trader to enable a new delegate:

1. `GET https://tx-builder.avantisfi.com/delegate/set?trader=0xTRADER&delegate=0xDELEGATE` — trader signs.
2. From then on, the delegate can sign any `&delegate=0xDELEGATE` call on the trader's behalf.

## Pyth feeds

Avantis prices flow through Pyth, never Chainlink directly for entry / settlement. Two paths:

- **Pyth Core / Hermes** — every pair has a `feed.feedId` (bytes32). Hermes update bytes come from `feed-v3.avantisfi.com /v2/pairs/:pairIndex/price-update-data` (the `core` leg of the response).
- **Pyth Lazer / Pro** — pairs with `lazerFeed.state === 'stable'` get faster updates from the Lazer service. tx-builder calls the same `feed-v3` endpoint and uses the `pro` leg when available.

Agents don't normally call Pyth directly — tx-builder fetches the update bytes for `/margin/update` and `/tpsl/update`. Override via `priceUpdateData` + `priceSourcing` only when you have a fresh blob cached.

## Events worth indexing

If your agent indexes Avantis logs (rather than polling APIs), the relevant Trading events are:

| Event | When |
| --- | --- |
| `MarketOrderInitiated(orderId, trader, pairIndex, open)` | A market open/close has been queued |
| `MarketExecuted(orderId, t, open, price, positionSizeUSDC, percentProfit, usdcSentToTrader, isPnl, lossProtectionTier)` | Market order has settled |
| `LimitExecuted(orderId, t, ...)` | Limit / stop-limit triggered |
| `OpenLimitPlaced` / `OpenLimitUpdated` / `OpenLimitCanceled` | Resting limit lifecycle |
| `MarginUpdated` | Collateral deposit / withdrawal |
| `TpUpdated` / `SlUpdated` | TP / SL change |

`orderId` from `MarketOrderInitiated` (or just the `txHash`) is what to feed into `/v2/market-order-initiated/status/:txHash`.

## Deep contract docs

Internal contract documentation (full module-by-module walkthrough, accounting ledger, state transitions, fee schedule) lives in the private `avantis-contracts/docs` repo. The essentials above are sufficient for agent-level integration; consult the internal docs if you need to model liquidations, rollover math, or vault flows directly.
