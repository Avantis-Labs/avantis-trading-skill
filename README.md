# avantis-trading-skill

A plug-and-play Cursor Agent Skill that teaches any LLM agent to trade Avantis perps on Base end-to-end — read pair config, build unsigned transaction call data, look up positions and history, validate trades pre-signature, and poll order status.

Pairs naturally with Base's MCP wallet for signing: this skill returns ready-to-sign `{ to, data, value }` payloads; the wallet signs and broadcasts. No private keys, no RPC URLs, no on-chain side effects in the skill itself.

## Install

```bash
git clone https://github.com/Avantis-Labs/avantis-trading-skill.git ~/.cursor/skills/avantis-trading
```

That's the whole install. The skill auto-loads whenever the user mentions Avantis, perps, opening or closing a leveraged position on Base, ZFP, take-profit / stop-loss, delegated trading, or asks about their Avantis portfolio.

## What you get

End-to-end documentation for every public Avantis API an agent needs:

| Surface | Base URL | What |
| --- | --- | --- |
| **tx-builder** | `https://tx-builder.avantisfi.com` | Build unsigned call data for open / close / cancel / margin / TP-SL / delegate / approve |
| **data API** | `https://data.avantisfi.com/v2/trading` | Live pair + group + OI snapshot (overrides pre-applied) |
| **core backend** | `https://core.avantisfi.com` | Open positions, limit orders, OI per pair |
| **history API** | `https://api.avantisfi.com` | Closed trades, PnL, top trades, referral stats, market-order settlement status |

The skill ships a single entry point ([`SKILL.md`](SKILL.md)) plus reference files for each surface. Workflows, scaling rules, validation semantics, enum mappings, and end-to-end recipes are all included.

## How auto-invocation works

The `description` field in [`SKILL.md`](SKILL.md) lists trigger terms (`Avantis`, `avantis.trade`, `perp`, `ZFP`, `take-profit`, `delegated trading`, etc.). When the user's prompt matches any of them, the agent loads the skill automatically. No `disable-model-invocation` is set, so it's available wherever Cursor — or any compatible agent runtime — is running.

## Signing the call data

This skill never touches a private key. To sign and broadcast the call data it returns:

- **Base MCP wallet** — pass `to`, `data`, `value` (parsed with `BigInt`) directly to the wallet's send-transaction tool.
- **Direct viem** — see the snippet in [`SKILL.md`](SKILL.md) under *"Signing the response"*.

## Live API documentation

If you want to inspect the underlying services directly:

- Swagger UI for tx-builder: <https://tx-builder.avantisfi.com/docs>
- OpenAPI JSON for tx-builder: <https://tx-builder.avantisfi.com/openapi.json>
- Swagger UI for core backend: <https://core.avantisfi.com/api>

## Layout

```
avantis-trading-skill/
├── SKILL.md          # Main entry — auto-loaded by the agent
├── tx-builder.md     # Reference: tx-builder.avantisfi.com
├── data-api.md       # Reference: data.avantisfi.com/v2/trading
├── core-backend.md   # Reference: core.avantisfi.com
├── history-api.md    # Reference: api.avantisfi.com /v2/history + market-order status
├── contracts.md      # Trading contract addresses, enums, scaling, thresholds
└── recipes.md        # 8 end-to-end flows
```

## License

MIT — fork, repackage, or embed in your own MCP / agent toolkit freely.
