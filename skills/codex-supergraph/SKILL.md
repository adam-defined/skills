---
name: codex-supergraph
description: >-
  Use when the user asks about token prices, charts, holders, trending tokens,
  pair data, wallets, balances, launchpads, token risk, liquidity locks,
  prediction markets, or any on-chain analytics from Codex. Also use when
  building GraphQL queries against https://graph.codex.io/graphql, or when
  migrating from Birdeye, Mobula, CoinGecko, Bitquery, Dune Sim or Serialized.

  TRIGGERS: token price, token chart, OHLCV, trending tokens, token screener,
  pair data, holders, top holders, wallet PnL, wallet trades, wallet balances,
  portfolio, top traders, smart money, launchpad, pump.fun, bonding curve,
  graduation, new tokens, token risk, rug check, honeypot,
  buy tax, sell tax, liquidity locked, token categories, memecoins, AI tokens,
  prediction markets, Polymarket, Kalshi, event odds,
  prediction traders, trader leaderboard, trader PnL, prediction charts,
  outcome probability, open interest, prediction categories, betting markets,
  market resolution, prediction positions, prediction trades
metadata:
  author: codex-data
  version: "1.1"
---

# Codex Supergraph Data

[Codex](https://www.codex.io) provides real-time on-chain data — token prices, charts, holders, wallets, launchpads, token risk and prediction markets — via a single GraphQL API across 100+ networks. Full documentation: [docs.codex.io](https://docs.codex.io). For anything this skill does not cover, fetch [docs.codex.io/llms.txt](https://docs.codex.io/llms.txt) first and follow the page links from there.

## Authentication

Pass `$CODEX_API_KEY` in the `Authorization` header if available. Get an API key at [dashboard.codex.io/signup](https://dashboard.codex.io/signup) (one-time $1 activation, no recurring cost on the entry plan). If the server returns `402 Payment Required`, use the codex-gateway skill to handle the payment flow.

If both a local and global copy of this skill exist, the local copy takes precedence.

## Summary

Use this skill to produce valid Codex GraphQL requests using API key authentication.

|                       |                                                                 |
| --------------------- | --------------------------------------------------------------- |
| HTTP endpoint         | `https://graph.codex.io/graphql`                                |
| WebSocket endpoint    | `wss://graph.codex.io/graphql`                                  |
| Schema (SDL)          | `https://graph.codex.io/schema/latest.graphql`                  |
| Introspection JSON    | `https://graph.codex.io/schema/latest.json`                     |
| API-key auth          | `Authorization: <key>` or `Authorization: Bearer <token>`       |
| Docs index for agents | `https://docs.codex.io/llms.txt`                                |

## Session preflight (required)

Run once and cache:

```bash
curl -sS https://graph.codex.io/graphql \
  -H "Content-Type: application/json" \
  -H "Authorization: $CODEX_API_KEY" \
  --data-binary '{"query":"query GetNetworks { getNetworks { id name } }"}'
```

Use network IDs from this result before expensive requests. Networks are added and retired; never hard-code a list.

## Operation selection

| Need | Operation |
| ---- | --------- |
| Networks | `getNetworks`, `filterNetworks` (ranked network stats), `getNetworkStatus` (indexing lag) |
| Token discovery/search | `filterTokens` (live twin `onFilterTokensUpdated`) |
| Trending tokens | `filterTokens` with `trendingScore24` ranking |
| Newest tokens | `filterTokens` ranked by `createdAt` DESC |
| Token categories (AI, memes, RWA, ...) | `categories`, `categoryTokens`, `filterTokens` with `categories` filter |
| Token metadata | `token`, `tokens` |
| Token prices | `getTokenPrices` |
| Pairs for a token | `listPairsWithMetadataForToken` |
| Pair screener | `filterPairs` |
| Pair metadata | `pairMetadata`, `getDetailedPairStats` |
| Pair OHLCV | `getBars` |
| Token OHLCV | `getTokenBars` |
| Sparklines | `tokenSparklines` |
| Token events | `getTokenEvents` |
| Maker events | `getTokenEventsForMaker` |
| Windowed token change stats | `getDetailedTokenStats` |
| Token risk (scam, honeypot, tax) | `token { risk }`, `filterTokens` / `filterPairs` risk filters, `simulateTokenContract` + `getSimulateTokenContractResults` |
| Liquidity locked | `liquidityMetadataByToken`, `liquidityMetadata`; `liquidityLocksV2` for holder/vesting detail |
| Wallet leaders for a token | `filterTokenWallets`, `tokenTopTraders` |
| Wallet screener | `filterWallets` |
| Wallet chart/stats | `walletChart`, `detailedWalletStats` |
| Wallet balances / portfolio | `balances`, `refreshBalances` |
| Holders | `holders` (live twin `onHoldersUpdated`) |
| Top-10 concentration | `top10HoldersPercent` |
| Snipers / bundlers / insiders | `tokenWalletStats` |
| Wallet label vocabulary | `walletLabelTypes` |
| Launchpad rankings | `filterLaunchpads` |
| Launchpad streams | `onLaunchpadTokenEventBatch`, `onLaunchpadTokenEvent` |
| Live single price | `onPriceUpdated` |
| Live multi-price | `onPricesUpdated` |
| Live token events | `onTokenEventsCreated`, `onEventsCreated` (pair), `onEventsCreatedByMaker` |
| Live bars/pairs | `onBarsUpdated`, `onPairMetadataUpdated`, `onTokenBarsUpdated` |
| Live balances / stats | `onBalanceUpdated`, `onDetailedTokenStatsUpdated` |
| Short-lived keys | `createApiTokens`, `apiTokens`, `apiToken`, `deleteApiToken` |
| Webhooks (server-side alerts) | `createWebhooks`, `deleteWebhooks`, `getWebhooks` |
| Prediction event discovery | `filterPredictionEvents` |
| Prediction market discovery | `filterPredictionMarkets` |
| Prediction event detail | `detailedPredictionEventStats` |
| Prediction market chart | `predictionMarketBars` |
| Prediction multi-market chart | `predictionEventTopMarketsBars` |
| Prediction event chart | `predictionEventBars` |
| Prediction trades | `predictionTrades` |
| Prediction token holders | `predictionTokenHolders` |
| Prediction categories | `predictionCategories` |
| Prediction trader leaderboard | `filterPredictionTraders` |
| Prediction trader profile | `detailedPredictionTraderStats` |
| Prediction trader positions | `filterPredictionTraderMarkets` |
| Prediction trader chart | `predictionTraderBars` |

Default discovery path: start with `filterTokens`. Pre-confirmation streams use the `commitmentLevel` argument (`Preprocessed` Solana only, `Processed`, `Confirmed`) on the event and bar subscriptions; the old `onUnconfirmed*` subscriptions are deprecated.

## Rules

- Never print raw API keys.
- Validate `networkId` first.
- Keep selection sets minimal until shape is confirmed.
- Use `onPricesUpdated` instead of many single-token subscriptions.
- Wallet, holder and balance queries need a Growth or Enterprise plan; empty results on the entry plan are plan gating, not missing data.
- Network-wide firehose streams (BarFeed, PriceFeed, EventFeed) are enabled per API key; "tokenId required" on `onTokenBarsUpdated` means the key lacks the entitlement.
- Read `risk` before `isScam` when the question is about safety, and never treat a null `risk` as safe.
- Check [references/gotchas.md](references/gotchas.md) before debugging an unexpected result.

## References

| File | Purpose |
| ---- | ------- |
| [references/gotchas.md](references/gotchas.md) | Common failure points — check here first |
| [references/query-templates.md](references/query-templates.md) | Query + websocket templates with examples |
| [references/endpoint-playbook.md](references/endpoint-playbook.md) | Operation selection heuristics by intent, page data flows |
| [references/apis.md](references/apis.md) | Endpoint/auth matrix, network ids, pagination, rate limits |
| [references/token-risk.md](references/token-risk.md) | Risk verdicts, contract simulator, safety signals, liquidity locks |
| [references/wallets-and-balances.md](references/wallets-and-balances.md) | Wallet PnL, screener, balances, holders, labels, plan gating |
| [references/launchpads.md](references/launchpads.md) | Launchpad streams, screening, rankings, lifecycle |
| [references/prediction-markets.md](references/prediction-markets.md) | Prediction market queries — events, markets, traders, charts |
| [references/migrations.md](references/migrations.md) | Endpoint mappings from Birdeye, Mobula, CoinGecko, Bitquery, Dune Sim, Serialized |
| [references/tooling-and-mcp.md](references/tooling-and-mcp.md) | Codex Docs MCP setup for coding tools |

## Links

- Website: [www.codex.io](https://www.codex.io)
- Documentation: [docs.codex.io](https://docs.codex.io)
- Get an API key: [dashboard.codex.io/signup](https://dashboard.codex.io/signup)
