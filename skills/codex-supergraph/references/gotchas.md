# Codex Supergraph Gotchas

Common failure points. Check here first when a query returns unexpected results.

## Validate against the schema on 400 errors

If a query returns a 400 or GraphQL validation error, fetch the latest schema and check your query against it:

```
https://graph.codex.io/schema/latest.graphql
```

Field names, input shapes, and argument structures change over time. Don't guess — validate. Deprecated fields still resolve but carry `@deprecated(reason: ...)` in the SDL; prefer the replacement named there.

## Input objects vs flat args

Many queries wrap their arguments in an `input` or `query` object rather than accepting flat args. Common examples:

- `getTokenEvents` takes `query: EventsQueryInput!` (not flat `address`/`networkId`)
- `getTokenEventsForMaker` takes `query: MakerEventsQueryInput!` (not flat `maker`/`address`)
- `filterTokenWallets` takes `input: FilterTokenWalletsInput!`
- `filterWallets`, `balances`, `tokenTopTraders`, `tokenWalletStats`, `detailedWalletStats` take `input: ...Input!`
- `walletChart` takes `input: WalletChartInput!` (with `range: { start, end }`, not `from`/`to`)
- `holders` takes `input: HoldersInput!`
- `filterTokens`, `filterPairs`, `filterLaunchpads` take flat `filters` / `rankings` / `limit` / `offset` arguments; `filterTokens` additionally `tokens`, `phrase`, `statsType`, `useAggregatedStats`

If you get a validation error about missing required args, check whether the schema expects an input wrapper.

## `NumberFilter` bounds are Floats

`gte`, `gt`, `lte`, `lt` on every `NumberFilter` are `Float`. A variable declared `Int` or `Int!` is rejected by validation. Declare the variable as `Float`, pass the whole filters object as one variable, or inline the literal.

## Composite ID formats

Several operations use `address:networkId` composite IDs:

- `getBars` / `getTokenBars`: `symbol` is `pairAddress:networkId` / `tokenAddress:networkId`
- `pairMetadata`: `pairId` is `pairAddress:networkId`
- `top10HoldersPercent`, `holders`, `onTokenBarsUpdated`, `onHoldersUpdated`: `tokenId` is `tokenAddress:networkId`
- `filterTokenWallets`: `tokenIds` array uses `tokenAddress:networkId`
- `filterTokens(tokens: [...])`, `categoryTokens`, `balances(input: { tokens })`: `tokenAddress:networkId`
- `refreshBalances`: native tokens are `native:<networkId>`

## `trendingScore24` is a sort attribute, not a field

You can rank by `trendingScore24` in `filterTokens` rankings, but it is not a selectable field on the result type. Don't include it in the selection set.

Cross-domain trap: tokens rank by `trendingScore24` (no "h"); prediction markets rank by `trendingScore24h` (with "h", and it *is* selectable there). Swapping them returns empty results rather than erroring, so verify which domain you're in.

## `filterTokens` pagination

Max 200 results per call. Use `offset` as an input arg to paginate. The response returns `page` (not `offset`) and `count`.

## `filterTokens` trending queries need `statsType`

When ranking by `trendingScore24`, set `statsType: "FILTERED"` or you'll get zero/null scores. Also use `trendingIgnored: false` and `potentialScam: false` in filters to exclude noise.

## `filterTokens` hides scams by default

`isScam: true` tokens are excluded unless `filters.includeScams: true`. Any query that wants `riskVerdicts: [SCAM]` needs it. `creatorAddress` in `TokenFilters` is deprecated; use `creatorAddresses: [...]`.

## Liquidity fields: which number you get

- Default mode (`useAggregatedStats` off): `TokenFilterResult.liquidity` is the token's **top pair** liquidity, measured on the side the price route exits through (usually the quote side). `totalLiquidityUsd` is the token's **total liquidity**: the combined liquidity of its 15 deepest pools, leaving out pools with no trade in the last 7 days, counting only the real money each pool can reach in one or two trades into the network's reference tokens. It is a floor of what could be withdrawn, not a TVL. Null until computed.
- `useAggregatedStats: true`: `liquidity` returns the same value as `totalLiquidityUsd` on networks where the real-money total is enabled (verified October 2026 on Ethereum, Base, BNB and Solana). On other networks `liquidity` is the token's aggregated liquidity and the two fields differ; there `totalLiquidityUsd` can still overstate what the pools hold (seen on Robinhood Chain, Monad, X Layer and Polygon in October 2026). When the two disagree, trust the smaller one and cross-check with `listPairsWithMetadataForToken`.
- Hub tokens that other tokens pair against (WETH, SOL, VIRTUAL, HYPE) read a `totalLiquidityUsd` below the sum of their pools. That is by design: a pool whose other side routes back through the hub itself does not count as backing.
- Both are recomputed when the token trades, so a token with no trade since the definition changed keeps its previous value.
- All token-level liquidity values are one-sided. Only `pairMetadata` gives both sides: `token0` / `token1` each expose `pooled` and `price`; side USD is `pooled × price`; pool TVL is the sum. Never double a one-sided value.
- `getTokenPrices.liquidityUsd` is the token's reserve-derived liquidity across routable pools and can differ from `totalLiquidityUsd`.
- Bonding-curve pairs (pump.fun, Flap, Four.meme, Meteora DBC, LaunchLab, Virtuals, ...) report the quote reserve the curve holds, not the value of unsold inventory. A curve under $100 of quote reports no liquidity. Expect small numbers before graduation.
- Uniswap v4 pools that have not been backfilled return `pair { pooled { token0 token1 invalidReserves } }` with zeroes and `invalidReserves: true`. That means "not backfilled yet", not "empty"; such pairs also read liquidity 0 and may carry `MinimumLiquidity` in `potentialScamReasons` until backfilled.
- Codex plans to make aggregated stats the default for `filterTokens`; when that happens the default `liquidity` changes meaning. Check the changelog if a liquidity figure jumps.

## Market cap

`marketCap` is contract supply × price per network (WETH on Base is not ETH). `circulatingMarketCap` uses a third-party (CoinGecko) circulating supply when mapped and is then a global figure, so it can exceed `marketCap`; prefer `marketCap` when they disagree. Overflowed or above-$5T caps read 0 (older data showed 9223372036854775807; treat that as invalid).

## `Event` price fields

The `Event` type has no top-level `priceUsd`. USD totals are `token0SwapValueUsd` / `token1SwapValueUsd`; base-token amounts are `token0ValueBase` / `token1ValueBase`. The per-trade execution price is inside the union: `data { ... on SwapEventData { priceUsd priceBaseToken } }`. Candles do not use it (see below). `tradeSource { id displayName }` names the app a trade was placed through; populated on Solana and EVM networks, null when there is no signal.

## Candles are the pool's post-block price

Bars take `o/h/l/c` from the pool price after each block (`token0PoolValueUsd` on events), one value per block shared by every trade in it, not from individual trade prices. A trade's `priceUsd` and the chart's header price are different quantities (execution price in one pool vs the token price from the top pair or weighted across pools).

## Closed candles settle a few seconds late, new ones appear a few seconds late

A closed 1m bar keeps updating up to ~3.5s past its boundary and there is no "final" flag; wait boundary + 5s and re-fetch trailing bars for corrections. After the first trade of a new minute, the new 1m/5m bar can take 6 to 16s to become queryable while coarser bars update in 1 to 2s. Drive a live price label from `getTokenPrices` or a 15m+ close, never from the last 1m close.

## `getTokenEvents` timestamp bounds

`timestamp { from, to }` is compared against a sub-second internal time while the returned `timestamp` is rounded to the second, so an event whose timestamp equals `from` can be excluded (per slot on Solana). Pad both bounds by 1 to 2s and dedupe on `transactionHash` + `logIndex`.

## `getTokenEventsForMaker` filters per page

`tokenAddress` is applied per 200-event page, so an active wallet over a wide window can return 0 items plus a cursor. Use a tight timestamp window or keep paging the cursor.

## `holders` response uses `items`, not `holders`

The `HoldersResponse` type returns the holder list in `items`, not a `holders` field. Each `Balance` object has `address`, `balance`, `shiftedBalance`, `balanceUsd` — there is no `sharedPct` field. `filterContracts: true` removes pools, lockers and other contract wallets.

## Wallet data is plan-gated and keyed on the signer

`filterWallets`, `filterTokenWallets`, `tokenTopTraders`, `detailedWalletStats`, `walletChart`, `holders`, `balances` and webhooks need Growth or Enterprise. Wallet records key on `maker` (the transaction signer), so router or custodial buys attribute to the router. `filterTokenWallets.count` is rows on the page, not a total. `statsYear1` is deprecated (returns 30-day unique tokens); use `statsYear`.

## Firehose entitlements are per API key

Network-wide streams (`onTokenBarsUpdated(networkId)` without `tokenId`, `onPriceUpdated` / `onTokenEventsCreated` without an address) need BarFeed / PriceFeed / EventFeed enabled on that specific key. "tokenId required" means the key lacks it. One API key per application on the entry plan.

## `listPairsWithMetadataForToken` field names

The result type uses `volume` (not `volume24`) and does not have a `price` field.

## `getBars` missing timestamps

Always include `t` in the selection set. Without it the bars are unplottable. The fields are `t o h l c volume`.

## `getBars` max datapoints

Max 1500 datapoints per request. Narrow the time window or increase the resolution if you hit the limit.

## `onTokenBarsUpdated` bar fields

Its `aggregates.r1 { usd { o h l c volume } }` shape differs from `getBars`; `v` on `IndividualBarData` is deprecated in favour of `volume` (a String).

## `networkId` validation

Always validate `networkId` against `getNetworks` before using it. A wrong ID returns empty results, not an error. Common IDs: Ethereum=`1`, Solana=`1399811149`, Base=`8453`, Robinhood=`4663`.

## Subscription fan-out and cadence

Use `onPricesUpdated` (batch) instead of opening many `onPriceUpdated` (single) subscriptions. Batch input supports ~25 tokens per subscription. Price streams emit at most once per token per block, not per swap; event streams deliver one message per pool per block. Transport is WebSocket only (no NATS or Kafka); webhooks are the other push path.

## Commitment levels

`commitmentLevel: [Preprocessed, Processed, Confirmed]` on event and bar subscriptions. `Preprocessed` is Solana events only, pre-execution, and many such events never confirm. `Processed` returns Base Flashblocks ahead of confirmation. Use them only when earliest signal matters more than finality.

## Rate limits return 429, not GraphQL errors

Rate limit responses are HTTP 429, not wrapped in GraphQL `errors`. Check the HTTP status, not just the response body.

## Short-lived tokens can't manage themselves

`apiTokens`, `apiToken`, and `deleteApiToken` are not available when authenticated with a short-lived token. Use the long-lived API key for token management.

## Webhook conditions

`TOKEN_TRANSFER_EVENT` webhooks have no amount or USD condition (only token, network, address, direction); filter client-side on `shiftedAmount × getTokenPrices`. `MARKET_CAP_EVENT` webhooks evaluate the token's price, not a pair's; `pairAddress` on them is deprecated. Comparison values in conditions are strings.

## Token metadata is a snapshot

`name` and `symbol` are captured at index time and not re-read if the contract owner renames the token. Solana tokens whose Metaplex metadata was missed at discovery can stay `UNKNOWN` / `?`.

## Prediction market `bestAskCT` is the implied probability

For stablecoin-collateral markets, `bestAskCT` of 0.65 means 65% implied probability. Prefer CT (Collateral Token) values over USD values.

## Prediction trader ID format

Trader IDs use the format `address:Protocol` (e.g., `0x1234...:Polymarket`). Missing the protocol suffix will return no results. For Polymarket the address is the proxy wallet, not the user's signing EOA.

## Prediction OHLC values are strings

All OHLC fields (`o`, `h`, `l`, `c`) in prediction market bars are strings, not numbers. Parse them to floats before rendering. Timestamps are unix seconds — multiply by 1000 for JavaScript milliseconds.

## `predictionEventBars` has no per-outcome prices

`predictionEventBars` only returns aggregated volume, liquidity, and open interest. For per-outcome price data, use `predictionMarketBars` or `predictionEventTopMarketsBars`.

## `filterPredictionEvents` vs `filterPredictionMarkets`

Events are containers grouping related markets. Use `filterPredictionEvents` for discovery, then `filterPredictionMarkets(eventIds)` for market-level pricing within an event. Exclude `market-maker` and `high-frequency-bot` via `excludeTraderLabels` on `filterPredictionTraderMarkets` for a human leaderboard. `POSITION_REDEEMED` replaced the deprecated `PAYOUT_REDEMPTION` trade type.

## `competitiveScore24h` is markets-only

`competitiveScore24h` is available on `filterPredictionMarkets` but not on `filterPredictionEvents`. It measures how close the outcome prices are.
