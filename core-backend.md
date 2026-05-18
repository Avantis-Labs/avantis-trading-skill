# core-backend reference

`https://core.avantisfi.com` — public, unauthed read service exposing per-trader open positions, open limit orders, and per-pair open interest.

- **Swagger UI**: <https://core.avantisfi.com/api>
- **OpenAPI JSON**: <https://core.avantisfi.com/api-json>

All endpoints return JSON unless noted. Rate limit applies (`HTTP 429` under load).

## `GET /user-data`

Per-trader: open positions and open limit orders. Most useful endpoint for agents that need to know a trader's current on-chain state before acting.

```
GET /user-data?trader=0x1111111111111111111111111111111111111111
```

Response:

```json
{
  "positions": [
    {
      "isOneCT":          false,
      "trader":           "0x...",
      "pairIndex":        62,
      "index":            0,
      "buy":              false,
      "collateral":       "2000000000",
      "leverage":         "100000000000",
      "openPrice":        "443574692469",
      "tp":               "222108938128",
      "sl":               "479060667866",
      "liquidationPrice": "481278299403",
      "rolloverFee":      "10908",
      "lossProtection":   "1",
      "openedAt":         1758710931,
      "isPnl":            false
    }
  ],
  "limitOrders": [
    {
      "isOneCT":          false,
      "trader":           "0x...",
      "pairIndex":        21,
      "index":            0,
      "buy":              false,
      "block":            35961058,
      "collateral":       "30000000",
      "positionSize":     "3000000000",
      "price":            "37600000000000",
      "leverage":         "1000000000000",
      "tp":               "37528560000000",
      "sl":               "37670000000000",
      "slippageP":        "30000000000",
      "executionFee":     "0",
      "liquidationPrice": "37919600000000",
      "limitOrderType":   0
    }
  ]
}
```

### Field semantics

#### `positions[]`

| Field | Type | Scaling | Meaning |
| --- | --- | --- | --- |
| `trader` | string | — | Trader EVM address |
| `pairIndex` | number | — | Pair index (see `data-api.md`) |
| `index` | number | — | Per-pair trade index (use as `tradeIndex` on `/trade/close`, `/margin/update`, etc.) |
| `buy` | bool | — | `true` = long, `false` = short |
| `collateral` | string | **/ 1e6** | Collateral in trade, USDC |
| `leverage` | string | **/ 1e10** | Leverage multiplier |
| `openPrice` | string | **/ 1e10** | Entry price |
| `tp` | string | **/ 1e10** | Take-profit price (`0` if not set) |
| `sl` | string | **/ 1e10** | Stop-loss price (`0` if not set) |
| `liquidationPrice` | string | **/ 1e10** | Liquidation price |
| `rolloverFee` | string | **/ 1e6** | Accrued margin fee, USDC |
| `lossProtection` | string | — | Loss protection tier (`0` = none) |
| `openedAt` | number | — | Unix seconds when opened |
| `isPnl` | bool | — | `true` = zero-fee perp (ZFP), `false` = fixed-fee |
| `isOneCT` | bool | — | Internal flag for one-click trading wallet linkage; ignore unless needed |

#### `limitOrders[]`

Same shape as positions plus / minus:

| Field | Type | Scaling | Meaning |
| --- | --- | --- | --- |
| `price` | string | **/ 1e10** | Trigger price for the limit |
| `slippageP` | string | **/ 1e10** | Slippage tolerance percent at execution |
| `block` | number | — | Block number when the limit was registered |
| `executionFee` | string | **/ 1e6** | Currently `0` |
| `positionSize` | string | **/ 1e6** | Notional position size = `collateral × leverage` |
| `limitOrderType` | number | — | `0` for `LIMIT`, others reserved |
| `collateral` | string | **/ 1e6** | Collateral committed |

### Reading example

```js
const r = await fetch(`https://core.avantisfi.com/user-data?trader=${trader}`).then(x => x.json());
for (const p of r.positions) {
  console.log({
    pair:    p.pairIndex,
    side:    p.buy ? 'LONG' : 'SHORT',
    coll:    Number(p.collateral) / 1e6,
    lev:     Number(p.leverage) / 1e10,
    open:    Number(p.openPrice) / 1e10,
    liq:     Number(p.liquidationPrice) / 1e10,
    isZfp:   p.isPnl
  });
}
```

### Behaviour

- Empty body (`{ positions: [], limitOrders: [] }`) for an unknown or malformed `trader`.
- Returns only **open** positions and **resting** limits. For closed trades, use the history API.
- No auth, no allowlist.

## `GET /user-data/config`

Feature flags for `/user-data` (used by Avantis UI to gate the API for some wallets — not on-chain trading parameters).

```
GET /user-data/config?wallet=0x...
```

Response:

```json
{ "globallyEnabled": true, "enabledForWallet": false }
```

If both flags are false the trader cannot use one-click trading; this is **not** a trading-permission gate, just an API-feature flag.

## `GET /open-interests`

Per-pair OI aggregates (effective + pending), derived from the indexed event stream.

```
GET /open-interests
```

Response (array):

```json
[
  { "pairIndex": 0, "longOI": 0, "shortOI": 0, "pendingLongOI": 0, "pendingShortOI": 0 },
  ...
]
```

| Field | Meaning |
| --- | --- |
| `pairIndex` | Pair index |
| `longOI` / `shortOI` | Effective open interest, USDC (human decimal) |
| `pendingLongOI` / `pendingShortOI` | OI from events not yet effective by `validAfter` |

Returns `[]` if the Redis snapshot is missing.

## `GET /v2/open-interests`

Same `openInterests` array as above, plus per-market-maker breakdown.

```json
{
  "openInterests": [ { ... } ],
  "mmData": [
    {
      "wallet": "0x...",
      "openInterests": [ { "pairIndex": 0, "longOI": 0, "shortOI": 0 }, ... ]
    }
  ]
}
```

`mmData` reflects the configured market-maker wallets only (driven by env config in the service).

## `GET /health`

Plain text `OK`. Not JSON.

## `GET /api` and `GET /api-json`

`GET /api` serves Swagger UI. `GET /api-json` (and `/api-yaml`) serves the OpenAPI spec. Use the spec as the authoritative shape source if it ever drifts from this doc.

## Other endpoints (not recommended)

The service also registers `/positions` and `/limit-orders` which return paginated dumps **across all traders**. Agents should prefer `/user-data?trader=` — it is faster, scoped, and avoids leaking other users' data.

`PUT /refresh-positions` exists but requires an `api-key` query param and triggers an admin job; not relevant for trader agents.

## Operational notes

- Rate-limited at the gateway; expect `HTTP 429` when bursting.
- CORS: not in the open-CORS prefix list — call from server-side (or proxy) when running in a browser.
- Returns standard HTTP status codes (200 / 4xx / 5xx); no `{ success: ... }` envelope on this surface.
