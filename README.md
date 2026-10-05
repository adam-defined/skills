# Codex Skills

Agent Skills for integrating [Codex](https://codex.io/) on-chain data APIs into AI-powered applications.

## Skills

### `skills/codex-supergraph`
Query the Codex Supergraph GraphQL API for real-time and historical on-chain data: token prices, screeners, pair metadata, OHLCV bars, trades, wallets and balances, holders, launchpads, token risk, liquidity locks, prediction markets, and live WebSocket subscriptions.

- **Auth**: API key (header `Authorization: <key>`)
- **Setup**: Get an API key at [dashboard.codex.io/signup](https://dashboard.codex.io/signup) (one-time $1 activation, no recurring cost on the entry plan)
- **Entry point**: [`skills/codex-supergraph/SKILL.md`](skills/codex-supergraph/SKILL.md)

### `skills/codex-gateway`
Pay per query through the Machine Payment Protocol (MPP) 402 flow when no API key is available. Queries only.

- **Entry point**: [`skills/codex-gateway/SKILL.md`](skills/codex-gateway/SKILL.md)

## Installation

```bash
npx skills add Codex-Data/skills -g --yes
```

If the installer stops with `PromptScript does not support global skill installation`, target the agents you use instead of every detected one (skills CLI issue [vercel-labs/skills#1352](https://github.com/vercel-labs/skills/issues/1352)):

```bash
npx skills add Codex-Data/skills -g -a claude-code -a cursor -y
```

## Specification

These skills follow the [Agent Skills specification](https://agentskills.io/specification).

## Official Links

- [Codex](https://codex.io/)
- [Codex Docs](https://docs.codex.io/)
- [GraphQL Schema](https://graph.codex.io/schema/latest.graphql)

## License

MIT
