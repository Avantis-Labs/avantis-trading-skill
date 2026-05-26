# history API reference

`https://api.avantisfi.com` — public read service for closed-trade history, portfolio aggregates, referral stats, and market-order settlement status.

All endpoints return HTTP 200 (even for logical failures) and use the **legacy** response envelope:

```json
{ "success": true,  /* fields */ }
{ "success": false, "errorMessage": "..." }
```

Agents must always check `success` before reading the data fields.

## Endpoint index

| Endpoint | Purpose |
| --- | --- |
| `GET /v2/history/portfolio/history/:userAddress/:page/:limit?` | Closed trades, paginated |
| `GET /v2/history/portfolio/all/:userAddress/:page/:limit?` | All trades (open + closed), paginated |
| `GET /v2/history/portfolio/top/:userAddress` | Top 3 by net PnL |
| `GET /v2/history/portfolio/top/:userAddress/:limit?/:timeStamp?` | Top N by net PnL, optionally from a timestamp |
| `GET /v2/history/portfolio/profit-loss/:userAddress/:grouped?/:startDate?` | Aggregated PnL (optionally per-pair, optionally from a date) |
| `GET /v2/history/referral/stats/:userAddress` | Referral activity for a wallet |
| `GET /v2/market-order-initiated/status/:txHash` | Market-order settlement state |

`userAddress` must be `0x`-prefixed (the service short-circuits otherwise). `txHash` is lowercased server-side.

## `GET /v2/history/portfolio/history/:userAddress/:page/:limit?`

Closed trades only. Server caps `limit` at **20**. The `limit` path segment is optional, but agents should always supply it — when omitted, server behavior is undefined.

`page` is **1-indexed** (the first page is `1`, not `0`). The same convention applies to `/v2/history/portfolio/all/...` below.

Filter (server-side): events `MarketExecuted` / `LimitExecuted` with `event.args.open === false` **or** `event.args.orderType < 3`, and the trader matches.

```json
{
  "success": true,
  "portfolio": [
    {
      "event": {
        "args": {
          "t":                  { "trader": "0x...", "pairIndex": 1, "index": 0, ... },
          "positionSizeUSDC":   0.98542,
          "price":              112002.68,
          "usdcSentToTrader":   0,
          "_feeInfo":           { "closingFee": 0.147813, "r": 0.222313, "liquidationFee": 0, "keeperFee": 0, ... }
        }
      },
      "_grossPnl": -0.615294,
      "timeStamp": "2025-09-23T02:54:35.000Z"
    }
  ],
  "count": 70892,
  "pageCount": 7090
}
```

| Field | Meaning |
| --- | --- |
| `portfolio[i].event.args.t` | Trade struct at open time (same field order as on-chain `Trade`). See [`contracts.md`](contracts.md) for tuple layout. |
| `portfolio[i].event.args.positionSizeUSDC` | **Collateral closed in this event** (not the position notional). |
| `portfolio[i].event.args.price` | Closing price. |
| `portfolio[i].event.args.usdcSentToTrader` | USDC returned to trader on close. |
| `portfolio[i].event.args._feeInfo` | Fee breakdown for the close. |
| `portfolio[i]._grossPnl` | Gross PnL for this close (USDC, negative if loss). |
| `portfolio[i].timeStamp` | ISO 8601 close timestamp. |
| `count` | Total matching records across all pages. |
| `pageCount` | `ceil(count / limit)`. |

## `GET /v2/history/portfolio/all/:userAddress/:page/:limit?`

Like `/history` but includes still-open trades. Filter: any `MarketExecuted` / `LimitExecuted` event matching the trader. `limit` capped at 20.

```json
{
  "success": true,
  "portfolio": [
    {
      "trade":              { "trader": "0x...", "pairIndex": 1, "index": 0, ... },
      "collateral":         100,
      "positionSize":       1000,
      "executionPrice":     112002.68,
      "usdcSentToTrader":   0,
      "isPnl":              false,
      "grossPnl":           -0.615294,
      "netPnl":             -0.815294,
      "openingFee":         0.045,
      "feeInfo":            { ... },
      "referralData":       { ... },
      "timestamp":          1758710931,
      "longTimestamp":      "2025-09-23T02:54:35.000Z",
      "orderId":            123,
      "orderIdOfOpenTrade": 122,
      "open":               true
    }
  ],
  "count": 0,
  "pageCount": 0
}
```

| Field | Meaning |
| --- | --- |
| `trade` | Same as `event.args.t` from `/history`. |
| `collateral` | Collateral in trade (USDC, human decimal). |
| `positionSize` | `floor(collateral × leverage × 1e6) / 1e6`. |
| `executionPrice` | Price at fill. |
| `isPnl` | `true` for ZFP trades. |
| `grossPnl`, `netPnl` | Pre / post-fee PnL. |
| `openingFee` | Open fee in USDC. |
| `referralData` | Referrer info if any. |
| `open` | `true` if the trade is still open, `false` after close. |

## `GET /v2/history/portfolio/top/:userAddress`

Top **3** closed trades by net PnL, descending. Server-fixed limit.

```json
{
  "success": true,
  "portfolio": [
    {
      "event":          { "args": { "t": { ... }, "positionSizeUSDC": 0.98, "price": 112002.68, ... } },
      "_grossPnl":      8543.21,
      "_mapped_netPnl": 8499.7,
      "timeStamp":      "2025-09-23T02:54:35.000Z"
    }
  ]
}
```

Same filter as `/history` (closed only).

## `GET /v2/history/portfolio/top/:userAddress/:limit?/:timeStamp?`

Top N by net PnL, optionally from a Unix-seconds floor.

- `limit`: parsed as `Number(req.params.limit)`. Default `3` if missing.
- `timeStamp`: Unix **seconds** floor (e.g. `1696118400` = 2023-10-01). Omit for no time filter.

**Filter differs from `/top/:userAddress`**: matches `MarketExecuted` **or** `LimitExecuted` with `orderType < 3` — does **not** require `event.args.open === false` on `MarketExecuted`. Can therefore include still-open market entries.

Same response shape as `/top/:userAddress`.

## `GET /v2/history/portfolio/profit-loss/:userAddress/:grouped?/:startDate?`

Aggregated PnL.

- `grouped`: the literal string `grouped` to bucket per `pairIndex`; anything else (including `ungrouped`, omitted) returns a single combined bucket.
- `startDate`: any value parseable by `new Date(...)`. If the literal string `false` or omitted, no time filter. The filter is `timeStamp >= new Date(startDate).getTime() / 1000` — that is, **Unix seconds**.

```json
{
  "success": true,
  "data": [
    { "total": 1234.56, "totalCollateral": 5000.00, "pairIndex": 1 },
    { "total": -89.20,  "totalCollateral": 2500.00, "pairIndex": 5 }
  ]
}
```

In the **ungrouped** case `pairIndex` is `null`. Both `total` and `totalCollateral` are rounded to 2 decimals.

PnL per row uses:

- `$_mapped_netPnl` when `event.args.isPnl === true` (ZFP)
- `$_mapped_grossPnl` otherwise

Filter: same closed-trade filter as `/history`.

## `GET /v2/history/referral/stats/:userAddress`

Referral activity, both as a referrer and as a referred trader.

```json
{
  "success": true,
  "data": {
    "asReferrer": {
      "totalFees":    12345.67,
      "totalRebates": 1234.56,
      "totalTraders": 42
    },
    "asTrader": {
      "totalVolume":     98765.43,
      "totalFeeDiscount": 234.56
    }
  }
}
```

| Field | Scaling | Meaning |
| --- | --- | --- |
| `asReferrer.totalFees` | **pre-divided by 1e6** | Total fee USDC generated by users this wallet referred |
| `asReferrer.totalRebates` | **pre-divided by 1e6** | Total rebate USDC the referrer earned |
| `asReferrer.totalTraders` | integer count | Distinct referred trader count |
| `asTrader.totalVolume` | **pre-divided by 1e6** | Total volume traded under a referrer |
| `asTrader.totalFeeDiscount` | **pre-divided by 1e6** | Total USDC fee discount received |

All USDC amounts on **this** endpoint come back as plain decimals — the service already divides by `1e6`. Multiply when storing in raw units.

## `GET /v2/market-order-initiated/status/:txHash`

Status of a market-order intent (the on-chain init tx). Use this to poll a freshly broadcast market-open or close until it settles.

```json
{
  "success": true,
  "data": {
    "status":  "executed",
    "orderId": "12345",
    "initiated": {
      "trader":         "0x...",
      "pairIndex":      1,
      "open":           true,
      "timestamp":      1758710900,
      "blockNumber":    35961058,
      "blockTimestamp": 1758710900,
      "txHash":         "0x..."
    },
    "executed": {
      "blockNumber":      35961100,
      "blockTimestamp":   1758710930,
      "logIndex":         12,
      "txHash":           "0x...",
      "orderId":          "12345",
      "t_trader":         "0x...",
      "t_pairIndex":      1,
      "t_index":          0,
      "t_initialPosToken": 0,
      "t_positionSizeUSDC": 100,
      "t_openPrice":      112002.68,
      "t_buy":            true,
      "t_leverage":       10,
      "t_tp":             0,
      "t_sl":             0,
      "t_timestamp":      1758710930,
      "open":             true,
      "price":            112002.68,
      "positionSizeUSDC": 100,
      "percentProfit":    0,
      "usdcSentToTrader": 0,
      "isPnl":            false,
      "lossProtectionTier": 0
    }
  }
}
```

`status` is one of `executed`, `canceled`, `pending`.

| If `status === ...` | Blocks present |
| --- | --- |
| `executed` | `initiated` + `executed` |
| `canceled` | `initiated` + `canceled` |
| `pending` | `initiated` only |

`canceled` block fields: `blockNumber`, `blockTimestamp`, `logIndex`, `txHash`, `orderId`, `txFrom`, `trader`, `pairIndex`.

If the tx hash is unknown the service returns `{ success: false, errorMessage: "Market order not found..." }` (still HTTP 200).

### Polling pattern

```ts
async function waitForSettlement(txHash: string, maxMs = 60_000) {
  const t0 = Date.now();
  let delay = 500;
  while (Date.now() - t0 < maxMs) {
    const r = await fetch(`https://api.avantisfi.com/v2/market-order-initiated/status/${txHash}`).then(x => x.json());
    if (r.success && r.data.status !== 'pending') return r.data;
    await new Promise(res => setTimeout(res, delay));
    delay = Math.min(delay * 2, 4000);
  }
  throw new Error('settlement timeout');
}
```

## CORS

Portfolio endpoints (`/v2/history/portfolio/*`) use `Access-Control-Allow-Origin: *`.

Referral (`/v2/history/referral/*`) and market-order-initiated (`/v2/market-order-initiated/*`) endpoints are **not** in the open-CORS prefix list — call them server-side or proxy when running in a browser.

## Rate limit

Global limit: ~10 requests per second per IP. Agents that need higher throughput should batch and cache.
