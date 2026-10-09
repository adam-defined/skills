# Codex Supergraph API Reference

## Endpoint model

- HTTP GraphQL: `https://graph.codex.io/graphql`
- WebSocket GraphQL: `wss://graph.codex.io/graphql`
- Docs index for agents: `https://docs.codex.io/llms.txt`

## Auth matrix

| Mode | Headers | Supports | Notes |
| ---- | ------- | -------- | ----- |
| API key | `Authorization: <key>` | query, mutation, subscription | Standard path |
| Short-lived token | `Authorization: Bearer <token>` | query, mutation, subscription | Token-scoped limits |

Plans: the entry plan ("Almost free", one-time $1 activation) allows 10,000 requests a month and query-only access to market data. Wallet, holder, balance, webhook and contract-simulator operations need Growth or Enterprise. Firehose streams (whole-network bars, prices or events) are enabled per API key on enterprise agreements. WebSocket access can also be disabled per key (close code 4403 "Websockets are not enabled").

## Common network IDs

| Network | `networkId` |
| ------- | ----------- |
| Ethereum | `1` |
| Base | `8453` |
| Solana | `1399811149` |
| BNB Chain | `56` |
| Robinhood Chain | `4663` |
| Arbitrum | `42161` |
| Polygon | `137` |
| Optimism | `10` |
| Avalanche | `43114` |
| Unichain | `130` |
| HyperEVM | `999` |
| Monad | `143` |
| Arc | `5042` |
| World Chain | `480` |
| Sonic | `146` |
| X Layer | `196` |
| MegaETH | `4326` |
| Tempo | `4217` |

Run `getNetworks` once per session for the full list. Networks get retired (for example DFK, Degen Chain, Odyssey Chain and opBNB in September 2026), so never trust a cached list across sessions.

## Session preflight

```graphql
query GetNetworks {
  getNetworks {
    id
    name
  }
}
```

Use to validate `networkId` before price/event/chart requests. `getNetworkStatus(networkIds: [...])` returns `lastProcessedBlock` and `lastProcessedTimestamp` per network when indexing lag is the question.

## Operation matrix

| Task | Operation | Type | Notes |
| ---- | --------- | ---- | ----- |
| Network list | `getNetworks` | Query | Run once per session |
| Network stats ranking | `filterNetworks` | Query | Liquidity, volume, tx counts per network |
| Discover/search tokens | `filterTokens` | Query | Recommended first step; max 200 per page |
| Token categories | `categories`, `categoryTokens` | Query | `CANONICAL` and `NARRATIVE` categories |
| Snapshot token prices | `getTokenPrices` | Query | Max 25 inputs per call (extras are truncated) |
| Pair metadata/stats | `pairMetadata`, `getDetailedPairStats` | Query | Pair id: `pairAddress:networkId` |
| Pair screener | `filterPairs` | Query | Includes pool fee, lock and risk filters |
| Pair bars (OHLCV) | `getBars` | Query | Max 1500 datapoints |
| Token bars | `getTokenBars` | Query | Aggregate bars; max 1500 datapoints |
| Token events | `getTokenEvents` | Query | Cursor-paginated |
| Maker events | `getTokenEventsForMaker` | Query | Wallet scoped |
| Token risk | `token { risk }`, risk filters | Query | See token-risk.md |
| Contract simulator | `simulateTokenContract`, `getSimulateTokenContractResults` | Mutation / Query | Growth+, beta |
| Liquidity locks | `liquidityMetadataByToken`, `liquidityMetadata`, `liquidityLocksV2` | Query | See token-risk.md |
| Wallet screener | `filterWallets` | Query | Growth+ |
| Wallet detail | `detailedWalletStats`, `walletChart` | Query | Growth+ |
| Token wallets | `filterTokenWallets`, `tokenTopTraders`, `tokenWalletStats` | Query | Growth+ |
| Balances | `balances`, `refreshBalances` | Query / Mutation | Growth+ |
| Holders | `holders`, `top10HoldersPercent` | Query | Growth+ for `holders` |
| Launchpads | `filterLaunchpads` | Query | Beta |
| Live token price | `onPriceUpdated` | Subscription | Single token, one update per block |
| Live token prices | `onPricesUpdated` | Subscription | Multi token batch |
| Live pair stats | `onPairMetadataUpdated` | Subscription | Pair updates |
| Live bars | `onBarsUpdated`, `onTokenBarsUpdated` | Subscription | Chart updates |
| Live events | `onTokenEventsCreated`, `onEventsCreated`, `onEventsCreatedByMaker` | Subscription | `commitmentLevel` argument for pre-confirmation data |
| Live screener | `onFilterTokensUpdated` | Subscription | Same arguments as `filterTokens`; returns `updates` and `removedTokenIds` |
| Live launchpads | `onLaunchpadTokenEventBatch`, `onLaunchpadTokenEvent` | Subscription | Token snapshots |
| Live balances / holders / stats | `onBalanceUpdated`, `onHoldersUpdated`, `onDetailedTokenStatsUpdated` | Subscription | |
| Create short-lived tokens | `createApiTokens` | Mutation | Needs long-lived key |
| Manage short-lived tokens | `apiTokens`, `apiToken`, `deleteApiToken` | Query/Mutation | Not available from short-lived token |
| Webhooks | `createWebhooks`, `deleteWebhooks`, `getWebhooks` | Mutation / Query | Long-lived key only |

## Pagination

- `filterTokens`, `filterPairs`, `filterWallets`, `filterLaunchpads`, `categoryTokens`: `limit` + `offset`; the response returns `count` and `page` (tokens) or `offset`.
- `getTokenEvents`, `getTokenEventsForMaker`, `holders`, `balances`, `getWebhooks`, `predictionTrades`: `cursor` strings; pass the returned cursor back.
- `getBars` / `getTokenBars`: `from`, `to`, `resolution`, optional `countback` (max 1500 datapoints).

## WebSocket constraints

- Protocol: `graphql-transport-ws`
- Send `connection_init` with `Authorization` in payload
- Wait for `connection_ack`
- Unsubscribe with `complete`
- One message per pool per block for event streams; price streams emit at most once per token per block.

## Common failures

| Symptom | Likely cause | Fix |
| ------- | ------------ | --- |
| 401 / UNAUTHENTICATED | Missing or invalid API auth | Validate key/token and header format |
| 402 Payment Required | No credential at all | Add the API key, or use the codex-gateway (MPP) flow |
| 429 / Too Many Requests | Rate limit exceeded | Back off with exponential delay and retry |
| NOT_AUTHORIZED on wallet / holder / balance fields | Plan gating | Growth or Enterprise plan; not a data bug |
| 4403 on WebSocket connect | WebSockets disabled on this key | Use a key with WebSocket access |
| "tokenId required" on `onTokenBarsUpdated` | Key lacks the BarFeed entitlement | Pass a `tokenId`, or ask for the firehose entitlement |
| GraphQL validation error | Input shape mismatch | Check operation args and variable types; `NumberFilter` bounds are `Float`, not `Int` |
| Empty results, no error | Wrong `networkId`, or plan gating | Validate with `getNetworks`; check the plan |

Requests rejected for auth or plan reasons are not counted or billed. Not-found responses are, since a read happened.

See [gotchas.md](gotchas.md) for detailed failure patterns (symbol formats, pagination, rate limits, data semantics).
