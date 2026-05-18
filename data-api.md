# data API reference

`GET https://data.avantisfi.com/v2/trading` — consolidated snapshot of Avantis pair config, group config, fees, leverage envelopes, open interest, and Pyth feed metadata. Public; no auth.

All `overrides` from the upstream contract state are **already applied server-side**. The response shape never includes an `overrides` key.

## Top-level shape

```json
{
  "dataVersion": 1.5,
  "pairInfos":   { "0": { ... }, "1": { ... }, ... },
  "groupInfo":   { "0": { ... }, "1": { ... }, ... },
  "pairCount":   102,
  "maxTradesPerPair": 40,
  "totalOi":     38934218.65,
  "maxOpenInterest": 90871359.02
}
```

| Field | Type | Meaning |
| --- | --- | --- |
| `dataVersion` | number | Payload schema version |
| `pairInfos` | `Record<string, Pair>` | Map of `pairIndex` → pair record |
| `groupInfo` | `Record<string, Group>` | Map of `groupIndex` → group record |
| `pairCount` | number | Total pairs in snapshot |
| `maxTradesPerPair` | number | Max simultaneous trades per pair (system-wide) |
| `totalOi` | number | Total open interest across all pairs, in USDC (human decimal) |
| `maxOpenInterest` | number | Global OI cap, in USDC |

## `Pair` fields (the ones agents care about)

```json
{
  "index": 1,
  "from": "BTC",
  "to": "USD",
  "groupIndex": 0,
  "isPairListed": true,
  "leverages": {
    "minLeverage": 1,
    "maxLeverage": 75,
    "pnlMinLeverage": 75,
    "pnlMaxLeverage": 500
  },
  "values": {
    "maxGainP": 2500,
    "maxSlP": 80,
    "maxLongOiP": 50,
    "maxShortOiP": 50,
    "groupOpenInterestPercentageP": 100,
    "maxWalletOIP": 50,
    "isUSDCAligned": false
  },
  "pairMinLevPosUSDC": 100,
  "pairOI": 6701121.9,
  "pairMaxOI": 28271089.47,
  "maxWalletOI": 45435679.51,
  "openFeeP": 0.045,
  "closeFeeP": 0.045,
  "limitOrderFeeP": 0.01,
  "openInterest": { "long": 3479113.63, "short": 3222008.27 },
  "marginFee": { "long": 0.00171, "short": 0.00171 },
  "feed": {
    "feedId": "0x...",
    "maxOpenDeviationP": 1,
    "maxCloseDeviationP": 5,
    "attributes": {
      "symbol": "Crypto.BTC/USD",
      "is_open": true,
      "next_open": 0,
      "next_close": 0
    }
  },
  "lazerFeed": { "feedId": 1, "state": "stable", "exponent": -8 },
  "pnlFees": { "numTiers": 10, "tierP": [1, 5, ...], "feesP": [80, 50, ...] },
  "spreadP": 0.01,
  "pnlSpreadP": 0.01
}
```

### Identity

| Field | Meaning |
| --- | --- |
| `index` | Use as `pairIndex` in tx-builder calls and on-chain interactions |
| `from` / `to` | Base / quote symbols (e.g. `BTC` / `USD`) |
| `groupIndex` | Group this pair belongs to (drives group-level OI checks) |
| `isPairListed` | If `false`, the pair is delisted — agents must not open new trades on it |

### Leverage

| Field | Used by |
| --- | --- |
| `leverages.minLeverage` / `leverages.maxLeverage` | Fixed-fee order types (`market`, `limit`, `stop_limit`) |
| `leverages.pnlMinLeverage` / `leverages.pnlMaxLeverage` | Zero-fee perp / ZFP order type (`market_zero_fee`) |

The relevant envelope is enforced by tx-builder on `/trade/open`.

### Trade-size limits

| Field | Meaning |
| --- | --- |
| `pairMinLevPosUSDC` | Minimum `collateralUsdc × leverage` for an open trade |
| `pairOI` / `pairMaxOI` | Current vs allowed OI on this pair, in USDC |
| `maxWalletOI` | Per-wallet OI cap for this pair, in USDC |
| `values.maxGainP` / `values.maxSlP` | Max TP / SL distance as percentage (e.g. `2500` = 25.00, `80` = 80%) |
| `values.maxLongOiP` / `values.maxShortOiP` | Per-side OI caps as percent |

### Fees (informational, on-chain enforces)

| Field | Meaning |
| --- | --- |
| `openFeeP` / `closeFeeP` | Open / close fee as percent of position size |
| `limitOrderFeeP` | Extra fee for limit-order execution |
| `marginFee.long` / `.short` | Hourly margin fee percent, by side |
| `pnlFees.tierP[]` / `feesP[]` | Tiered performance fee schedule for ZFP trades |
| `spreadP` / `pnlSpreadP` | Constant spread (percent), fixed-fee vs ZFP |

### Feed

| Field | Meaning |
| --- | --- |
| `feed.feedId` | Pyth Hermes feed ID (`bytes32`) |
| `feed.attributes.is_open` | Live market-open flag |
| `feed.attributes.next_open` / `next_close` | Unix-seconds market schedule markers |
| `lazerFeed.feedId` | Pyth Lazer feed ID (uint) if set |
| `lazerFeed.state` | `stable` → use Lazer / PYTH_LAZER for `priceSourcing=1`; otherwise fall back to Hermes / PYTH_CORE (`0`) |

**Market-open logic** (for market orders):

- Open if `is_open === true`, **OR** (`now > next_open` **AND** `now < next_close`).
- Closed if `is_open === false` **AND** (`next_open > 0` **AND** `now < next_open`).

## `Group` fields

```json
{
  "name": "CRYPTO1",
  "groupMaxOI": 28271089.47,
  "groupOI": 14424987.62,
  "maxOpenInterestP": 28,
  "isSpreadDynamic": true
}
```

| Field | Meaning |
| --- | --- |
| `name` | Group label (`CRYPTO1`, `FOREX1`, etc.) |
| `groupMaxOI` | Max OI cap for the whole group, in USDC |
| `groupOI` | Current group OI, in USDC |
| `maxOpenInterestP` | Group OI ceiling as percent (policy metric) |
| `isSpreadDynamic` | Whether spread is dynamically adjusted within this group |

## Liquidity formula

The single check the UI (and tx-builder) uses to know whether a new position will fit:

```
pairAvail   = pairMaxOI - pairOI
groupAvail  = groupInfo[pair.groupIndex].groupMaxOI - groupInfo[...].groupOI
available   = min(pairAvail, groupAvail)
```

A new trade's position size (`collateralUsdc × leverage`) must be `<= available`.

## Skew

For per-side OI tracking:

```
longSkew  = openInterest.long  / (openInterest.long + openInterest.short)
shortSkew = openInterest.short / (openInterest.long + openInterest.short)
```

Skew tier configs (`longSkewConfig`, `shortSkewConfig`) map percentage thresholds to tier numbers (e.g. `{0:0, 1:50, 2:70, 3:80}` → ≥80% skew is tier 3).

## Worked example

`pair = BTC/USD` (index 1) on Base mainnet (live values change; field semantics do not):

```js
const r = await fetch('https://data.avantisfi.com/v2/trading').then(x => x.json());
const p = r.pairInfos['1'];
const g = r.groupInfo[String(p.groupIndex)];

if (!p.isPairListed)                  throw new Error('delisted');
if (p.lazerFeed?.state === 'stable')  console.log('use Pyth Lazer');
const envelope = isZfp
  ? [p.leverages.pnlMinLeverage, p.leverages.pnlMaxLeverage]
  : [p.leverages.minLeverage, p.leverages.maxLeverage];
const available = Math.min(p.pairMaxOI - p.pairOI, g.groupMaxOI - g.groupOI);
const minPos    = p.pairMinLevPosUSDC;
```

## Scaling / units

All numeric fields in this endpoint are **already human-decimal** (no `1e6` / `1e10` scaling). USDC amounts are plain dollars, percentages are plain percent, leverage is the multiplier.

This differs from on-chain values (and from the `core-backend` and `history-api` shapes) — see [`contracts.md`](contracts.md) for the scaling rules elsewhere.

## Caching

The tx-builder caches this snapshot in-process for 5 minutes (`PAIRS_CACHE_TTL`). If you're calling `data.avantisfi.com` directly, you should also cache to avoid hammering the origin.

## CORS

`Access-Control-Allow-Origin: *`. Safe to call from the browser.
