# recipes

Concrete end-to-end flows. Each one starts at the user's intent and ends with a confirmed on-chain state.

Conventions used throughout:

- `0xTRADER` = the trader wallet address.
- `0xDELEGATE` = a separate delegate wallet (only used in recipes that need it).
- All examples use mainnet hosts; agent should substitute `localhost:8080` when running tx-builder locally.

## 1. Open a market long on BTC/USD with 100 USDC at 10x

User intent: "long BTC with $100 at 10x".

```
GET https://tx-builder.avantisfi.com/trade/open
  ?trader=0xTRADER
  &pair=BTC/USD
  &side=long
  &collateralUsdc=100
  &leverage=10
```

Response: `{ ok: true, data: { to, from, data, value, ... } }`. tx-builder fetches the live BTC/USD price, runs all pre-trade checks, and includes `meta.validation` so the agent can confirm headroom.

Sign and broadcast:

```ts
const { ok, data, error } = await fetch(url).then(r => r.json());
if (!ok) throw new Error(error.message);
const txHash = await wallet.sendTransaction({ to: data.to, data: data.data, value: BigInt(data.value) });
```

Then poll settlement (recipe 8) and confirm via `/user-data` (recipe 7).

## 2. Close a trade fully

User intent: "close my open BTC position".

1. Read open positions:

   ```
   GET https://core.avantisfi.com/user-data?trader=0xTRADER
   ```

2. Find the target trade. Take its `pairIndex`, `index`, and `collateral` (raw 6-decimal int). Convert: `collateralUsdc = Number(collateral) / 1e6`.

3. Build the close:

   ```
   GET https://tx-builder.avantisfi.com/trade/close
     ?trader=0xTRADER
     &pairIndex=<pairIndex>
     &tradeIndex=<index>
     &collateralUsdc=<collateralUsdc>
   ```

   Pass the full collateral to fully close. For a partial close, pass a smaller `collateralUsdc`.

4. Sign and broadcast. Poll settlement.

## 3. Place a limit or stop-limit order

User intent: "buy BTC at $60,000 if it dips" → `limit` order; "buy BTC at $80,000 if it breaks out" → `stop_limit` order.

```
GET https://tx-builder.avantisfi.com/trade/open
  ?trader=0xTRADER
  &pair=BTC/USD
  &side=long
  &orderType=limit                    # or stop_limit
  &openPrice=60000
  &collateralUsdc=200
  &leverage=5
  &takeProfit=75000
  &stopLoss=55000
```

`openPrice` is required for both `limit` and `stop_limit`.

After signing, the order rests on-chain as an open limit. To check status:

```
GET https://core.avantisfi.com/user-data?trader=0xTRADER   # appears in limitOrders[]
```

To cancel before fill:

```
GET https://tx-builder.avantisfi.com/trade/cancel
  ?trader=0xTRADER
  &pairIndex=<pairIndex>
  &tradeIndex=<index from limitOrders>
```

When the limit fills, it becomes a regular position visible under `positions[]` in `/user-data`. The fill itself can be observed via `LimitExecuted` events; settlement-status polling is not used for limit fills.

## 4. Set or update TP and SL on an existing trade

User intent: "set TP to $80k and SL to $65k on my BTC long".

1. Find `pairIndex` + `tradeIndex` from `/user-data` (recipe 2).
2. Build:

   ```
   GET https://tx-builder.avantisfi.com/tpsl/update
     ?trader=0xTRADER
     &pairIndex=1
     &tradeIndex=0
     &takeProfit=80000
     &stopLoss=65000
   ```

`takeProfit` must be `> 0`. Pass `stopLoss=0` to clear the stop. The service fetches the required Pyth update bytes by default; pass `priceUpdateData` + `priceSourcing` to keep the call fully offline (see [`tx-builder.md`](tx-builder.md)).

## 5. Delegated trading

User intent: "let my bot trade for me without holding my private key".

One-time setup (the trader signs):

```
GET https://tx-builder.avantisfi.com/delegate/set
  ?trader=0xTRADER
  &delegate=0xDELEGATE
```

Sign with `0xTRADER`'s wallet. From this point on, every subsequent trade-side call adds `&delegate=0xDELEGATE`:

```
GET https://tx-builder.avantisfi.com/trade/open
  ?trader=0xTRADER
  &delegate=0xDELEGATE
  &pair=BTC/USD&side=long&collateralUsdc=100&leverage=10
```

The response's `from` is `0xDELEGATE`. The **delegate** wallet signs and broadcasts. The trade still belongs to `0xTRADER` on-chain.

Revoke when done:

```
GET https://tx-builder.avantisfi.com/delegate/remove?trader=0xTRADER
```

Only one delegate per trader at a time. Calling `/delegate/set` again replaces the previous delegate.

## 6. Approve USDC the first time

User intent: "I've never traded on Avantis before — approve USDC first."

Unlimited approval (typical for trading bots):

```
GET https://tx-builder.avantisfi.com/token/approve?trader=0xTRADER
```

Exact-amount approval:

```
GET https://tx-builder.avantisfi.com/token/approve
  ?trader=0xTRADER
  &amountUsdc=100
```

Spender defaults to `TradingStorage`. Override with `&spender=0x...` only if a non-standard contract needs the allowance.

This is a one-shot per trader (or per `amountUsdc` if exact). Trading calls will revert with insufficient allowance until this completes.

## 7. Inspect a user's full portfolio

User intent: "show me my Avantis activity".

Three sources, combined:

| Source | What it returns |
| --- | --- |
| `core.avantisfi.com /user-data?trader=` | Current **open** positions + resting limit orders |
| `api.avantisfi.com /v2/history/portfolio/all/:trader/0/20` | Page of **all** trades (open + closed) — chronological with PnL |
| `api.avantisfi.com /v2/history/portfolio/profit-loss/:trader/grouped` | Aggregate PnL **per pair** |

Sketch:

```ts
const [open, history, pnl] = await Promise.all([
  fetch(`https://core.avantisfi.com/user-data?trader=${trader}`).then(r => r.json()),
  fetch(`https://api.avantisfi.com/v2/history/portfolio/all/${trader}/0/20`).then(r => r.json()),
  fetch(`https://api.avantisfi.com/v2/history/portfolio/profit-loss/${trader}/grouped`).then(r => r.json())
]);

if (!history.success) throw new Error(history.errorMessage);
if (!pnl.success) throw new Error(pnl.errorMessage);

const openCount = open.positions.length;
const pendingCount = open.limitOrders.length;
const pnlByPair = pnl.data.reduce((acc, row) => {
  acc[row.pairIndex] = row.total;
  return acc;
}, {});
```

Remember the **scaling differences**:

- `/user-data` returns raw ints (divide collateral by `1e6`, prices by `1e10`).
- `/v2/history/portfolio/all` returns mixed; most numeric fields are already in human decimals.
- `/v2/history/portfolio/profit-loss` returns human decimals rounded to 2 places.

For top trades, swap `profit-loss` for `/v2/history/portfolio/top/:trader/5` (top 5 by net PnL).

## 8. Wait for a market open to settle

User intent: "I just sent the open tx — when did it actually fill?"

Market opens go through a two-step lifecycle on Avantis: an `MarketOrderInitiated` event when the user submits, then a `MarketExecuted` (or canceled) event when the keeper settles it. Use `/v2/market-order-initiated/status/:txHash` to poll.

```ts
async function waitForSettlement(txHash, maxMs = 60_000) {
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

const result = await waitForSettlement(txHash);
if (result.status === 'executed') {
  console.log('Filled at', result.executed.price, 'orderId', result.orderId);
} else if (result.status === 'canceled') {
  console.log('Order was canceled by keeper');
}
```

Exponential-backoff polling avoids hammering the gateway. Typical settlement is < 5 seconds; allow up to 60 seconds for slow blocks.

If you don't have the tx hash yet (e.g. the wallet hasn't returned it), wait for `wallet.sendTransaction(...)` to resolve before polling.

## Cross-recipe pattern: hands-off agent loop

```
loop:
  - read intent (open / close / cancel / tp-sl / etc.)
  - resolve any symbol → pairIndex (data API or /pairs)
  - build call data (tx-builder /trade/* | /margin/update | /tpsl/update | /delegate/* | /token/approve)
  - if response.ok === false → surface the error, ask the user
  - hand { to, data, value } to the wallet MCP for signing
  - record the resulting tx hash
  - for market opens / closes: poll /v2/market-order-initiated/status until non-pending
  - re-read /user-data to confirm state, summarize for the user
```

This is the canonical agent control loop. Every recipe above is one branch of this loop.
