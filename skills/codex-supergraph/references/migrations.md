# Migrating to Codex from another provider

Use when the user is moving from, replacing, or comparing against Birdeye, Bitquery, CoinGecko, Dune Sim, Mobula, or Serialized. Each table maps a source endpoint to the Codex operation that covers it; look the operation up in query-templates.md or the schema before writing a query. The full guides, with notes per endpoint, side-by-side examples, gaps and an AI migration prompt, are at https://docs.codex.io/migrations/<provider> (birdeye, bitquery, coingecko, dune-sim, mobula, serialized).

Conventions that trip every migration: network ids are integers (run `getNetworks`); token and pair ids are `address:networkId`; timestamps are unix seconds; USD amounts are strings; percent changes are decimals (0.05 = 5%); `getTokenPrices` takes at most 25 inputs; `filterTokens` returns at most 200 rows per page; wallet, holder and balance queries need a Growth or Enterprise plan; Codex returns swap and token-lifecycle events, not arbitrary transfers; historical prices come from `getTokenPrices` with a `timestamp` input.

## Birdeye

Guide: https://docs.codex.io/migrations/birdeye

| Birdeye | Codex |
|---|---|
| `/defi/price`, `/defi/multi_price`, `/defi/historical_price_unix` | `getTokenPrices` (add `timestamp` for a past price) |
| `/defi/history_price`, `/defi/ohlcv`, `/defi/v3/ohlcv` | `getTokenBars` |
| `/defi/ohlcv/pair`, `/defi/v3/ohlcv/pair`, `/defi/ohlcv/base_quote` | `getBars` (`quoteToken` to invert) |
| `/defi/price_volume/*`, `/defi/v3/price/stats/*`, `/defi/v3/token/market-data`, `/defi/v3/token/trade-data/*`, `/defi/v3/all-time/trades/*` | `getDetailedTokenStats` |
| `/defi/token_overview` | `token` + `getDetailedTokenStats` |
| `/defi/v3/token/meta-data/single`, `/defi/token_security`, `/defi/token_creation_info` | `token` |
| `/defi/v3/token/meta-data/multiple` | `tokens` |
| `/defi/v3/token/exit-liquidity*` | `liquidityMetadata`, `liquidityMetadataByToken` |
| `/defi/token_trending` | `filterTokens` ranked by `trendingScore24` |
| `/defi/v3/token/list*`, `/defi/tokenlist`, `/defi/v3/token/meme/*` | `filterTokens` (launchpad filters for meme lists) |
| `/defi/v2/tokens/new_listing` | `filterTokens` ranked by `createdAt`, or `onLatestTokens` |
| `/defi/v3/search` | `filterTokens(phrase:)` (`$SYMBOL` for exact symbol) |
| `/smart-money/v1/token/list` | `filterTokens` + `filterWallets` |
| `/defi/v3/pair/overview/single` / `multiple` | `getDetailedPairStats` / `getDetailedPairsStats` |
| `/defi/v2/markets` | `listPairsWithMetadataForToken` |
| `/defi/txs/*`, `/defi/v3/txs*`, `/defi/v3/token/txs*`, `/defi/v3/token/mint-burn-txs` | `getTokenEvents` (pair address in `query.address` for pair trades) |
| `/defi/v3/token/holder` | `holders` |
| `/holder/v1/distribution`, `/token/v1/holder-profile` | `token { top10HoldersPercent }` + `holders` |
| `/token/v1/holder-positions` | `filterTokenWallets` |
| `/token/v1/holder/chart` | Partial: `onHoldersUpdated` (live count only) |
| `/v1/wallet/token_list`, `/v1/wallet/token_balance`, `/wallet/v2/token-balance`, `/token/v1/holder/batch` | `balances` |
| `/v1/wallet/tx_list`, `/trader/txs/seek_by_time` | `getTokenEventsForMaker` |
| `/wallet/v2/current-net-worth`, `/wallet/v2/pnl*` | `detailedWalletStats` (+ `filterWallets`) |
| `/wallet/v2/net-worth*`, `/wallet/v2/pnl/chart` | `walletChart` |
| `/wallet/v2/leaderboard`, `/trader/gainers-losers` | `filterWallets` ranked by `realizedProfitUsd*` |
| `/defi/v2/tokens/top_traders` | `tokenTopTraders` |
| `/wallet/v2/balance-change` | `onBalanceUpdated` |
| `/wallet/v2/tx/first-funded` | `detailedWalletStats { wallet { firstFunding } }` |
| `/defi/networks`, `/v1/wallet/list_supported_chain` | `getNetworks` |
| `/defi/v3/txs/latest-block` | `blocks` |
| `/utils/v1/credits` | Usage page in the Codex dashboard |

Not covered: perpetuals, non-swap wallet transfers, RPC-style account and transaction data, identity and domain resolution, first-buyer and wallet-tag analytics, historical liquidity timeseries.

## Mobula

Guide: https://docs.codex.io/migrations/mobula

| Mobula | Codex |
|---|---|
| `/1/market/data` | `getTokenPrices` + `getDetailedTokenStats` |
| `/1/market/multi-data`, `/1/market/multi-prices`, `/2/token/price`, `/2/token/price-at` | `getTokenPrices` (add `timestamp` for a past price) |
| `/2/market/details` | `getDetailedTokenStats` or `getDetailedPairStats` |
| `/2/token/details` | `token` + `getDetailedTokenStats` |
| `/1/metadata`, `/2/asset/details`, `/2/token/security` | `token` |
| `/1/multi-metadata` | `tokens` |
| `/1/market/sparkline` | `tokenSparklines` |
| `/2/token/ath`, `/1/all`, `/1/market/query`, `/2/market/lighthouse`, `/1/metadata/categories` | `filterTokens` |
| `/1/search`, `/2/fast-search` | `filterTokens(phrase:)` |
| `/2/pulse`, `/1/pulse` | `filterTokens` with launchpad filters |
| `/2/token/logo-reuses` | Partial: `filterTokens` scam signals |
| `/1/market/pair` | `getDetailedPairStats` |
| `/1/market/pairs`, `/2/token/markets` | `listPairsWithMetadataForToken` |
| `/1/market/blockchain/pairs` | `filterPairs` |
| `/1/market/blockchain/stats` | `getNetworkStats` |
| `/1/market/history`, `/1/market/multi-history`, `/2/token/price-history`, `/2/asset/price-history`, `/2/token/ohlcv-history` | `getTokenBars` |
| `/1/market/history/pair`, `/2/market/ohlcv-history` | `getBars` |
| `/2/token/trades*`, `/2/token/trade`, `/1/market/trades/pair`, `/2/trades/filters` | `getTokenEvents` |
| `/2/token/holder-positions` | `holders` + `filterTokenWallets` |
| `/2/token/trader-positions` | `tokenTopTraders` |
| `/1/token/first-buyers` | Partial: earliest `getTokenEvents` |
| `/1/wallet/portfolio`, `/1/wallet/multi-portfolio`, `/2/wallet/holdings` | `balances` |
| `/1/wallet/history`, `/2/wallet/positions-history`, `/2/wallet/position-history` | `walletChart` |
| `/2/wallet/positions`, `/2/wallet/position` | `detailedWalletStats` + `filterTokenWallets` |
| `/2/wallet/analysis` | `detailedWalletStats` |
| `/1/wallet/trades`, `/2/wallet/trades`, `/2/wallet/activity`, `/1/wallet/transactions`, `/2/token/dev-history` | `getTokenEventsForMaker` |
| `/1/wallet/labels`, `/2/wallet/labels*` | Partial: `filterWallets` |
| `/1/blockchains`, `/metadata`, `/2/system-metadata` | `getNetworks` |
| `/2/usage` | Usage page in the Codex dashboard |

Not covered: perpetuals, swap and bridge execution, prediction-market trading (Codex has Polymarket and Kalshi data, read-only), raw transfers and transactions, funding and deployer tracing, DeFi protocol positions, news.

## CoinGecko

Guide: https://docs.codex.io/migrations/coingecko. CoinGecko coin ids (`bitcoin`) must be resolved to contract addresses first; `{network}` and `{platform}` become a `networkId`.

| CoinGecko | Codex |
|---|---|
| `/onchain/networks/{n}/tokens/{a}`, `.../info`, `/coins/{id}`, `/coins/{id}/contract/{a}` | `token` (+ `getDetailedTokenStats`) |
| `/onchain/networks/{n}/tokens/multi/{a}` | `tokens` |
| `/onchain/simple/networks/{n}/token_price/{a}`, `/simple/price`, `/simple/token_price/{platform}` | `getTokenPrices` |
| `/onchain/networks/{n}/tokens/{a}/ohlcv/{tf}`, `/coins/{id}/market_chart*`, `/coins/{id}/ohlc*` | `getTokenBars` |
| `/coins/{id}/history` | `getTokenPrices` with `timestamp` |
| `/onchain/networks/{n}/tokens/{a}/trades`, `/onchain/networks/{n}/pools/{p}/trades` | `getTokenEvents` |
| `/onchain/networks/{n}/tokens/{a}/top_holders` | `holders` (+ `top10HoldersPercent`) |
| `/onchain/networks/{n}/tokens/{a}/top_traders` | `tokenTopTraders` |
| `/onchain/networks/{n}/tokens/{a}/holders_chart` | Partial: `onHoldersUpdated` (live count only) |
| `/onchain/tokens/info_recently_updated`, `/coins/markets`, `/coins/list`, `/coins/list/new`, `/token_lists/*` | `filterTokens` |
| `/coins/top_gainers_losers` | `filterTokens` ranked by `change24` |
| `/search`, `/search/trending` | `filterTokens(phrase:)`, ranked by `trendingScore24` |
| `/onchain/networks/{n}/pools/{p}` / `pools/multi/{a}` | `getDetailedPairStats` / `getDetailedPairsStats` |
| `/onchain/networks/{n}/pools/{p}/info` | `pairMetadata` |
| `/onchain/networks/{n}/tokens/{a}/pools`, `/coins/{id}/tickers` (DEX only) | `listPairsWithMetadataForToken` |
| `/onchain/networks/{n}/pools/{p}/ohlcv/{tf}` | `getBars` |
| `/onchain/networks/{n}/pools`, `new_pools`, `trending_pools`, `/onchain/pools/megafilter`, `/onchain/search/pools`, `/onchain/pools/trending_search`, `.../dexes/{dex}/pools` | `filterPairs` |
| `/onchain/networks`, `/asset_platforms` | `getNetworks` |
| `/onchain/networks/{n}/dexes` | `filterExchanges` |
| `/key` | Usage page in the Codex dashboard |
| `/onchain/categories*`, `/coins/categories*`, `/coins/{id}/*_supply_chart`, `/simple/supported_vs_currencies`, `/exchange_rates` | Not supported (Codex categories are `categories` / `categoryTokens`; prices are USD only) |

Not covered: CEX tickers, exchanges and order books, derivatives, NFTs, treasury and entity data, global market aggregates, historical supply timeseries.

## Bitquery

Guide: https://docs.codex.io/migrations/bitquery

| Bitquery | Codex |
|---|---|
| `DEXTrades`, `DEXTradeByTokens`, DEX-program `Solana { Instructions }` | `getTokenEvents` (+ `getDetailedTokenStats` for aggregates) |
| `DEXTrades` filtered by trader | `getTokenEventsForMaker` |
| Latest `Trade.PriceInUSD`, price at a past block or time | `getTokenPrices` (add `timestamp` for a past price) |
| Crypto Price API OHLC, or OHLC bucketed from trades | `getTokenBars` (token), `getBars` (pair) |
| `Currencies` / token fields, `TokenSupplyUpdates` | `token`, `tokens` (`info.circulatingSupply`, `info.totalSupply`) |
| Safety signals inferred from raw data | `token` safety fields and `risk` |
| `EVM { Holders }` | `holders` (+ `top10HoldersPercent`) |
| `BalanceUpdates` to current holdings | `balances` |
| PnL grouped by trader | `detailedWalletStats`, `walletChart` |
| Top traders for a token | `tokenTopTraders` |
| Custom profitable-wallet aggregation | `filterWallets` |
| Trending or searched tokens | `filterTokens` (`trendingScore24`, `phrase`) |
| Pool fields on trades, ranked pools | `pairMetadata`, `getDetailedPairStats`, `listPairsForToken`, `filterPairs` |
| `Trade.Dex`, chain list, `Block` fields | `filterExchanges`, `getNetworks`, `blocks` |
| pump.fun via instructions | `onLaunchpadTokenEvent` + `token { launchpad }` |
| Polymarket `OrderFilled` / `ConditionResolution` events | `filterPredictionEvents` family (Polymarket and Kalshi) |

Not covered: mempool, raw logs, calls and instructions on arbitrary contracts, ad-hoc cube aggregations, historical point-in-time balances, arbitrary transfers, NFTs, chains Codex does not index (Bitcoin, Cardano, Tron, XRP and others; run `getNetworks` for the current list).

## Dune Sim

Guide: https://docs.codex.io/migrations/dune-sim

| Dune Sim | Codex |
|---|---|
| `/v1/evm/balances/{address}` (+ `/token/{t}`, `/stablecoins`) | `balances` (pass `tokens` to narrow, up to 200) |
| `/beta/svm/balances/{address}` | `balances` with `networks: [1399811149]` |
| `/v1/evm/activity/{address}`, `/v1/evm/transactions/{address}`, `/beta/svm/transactions/{address}` | `getTokenEventsForMaker` (swaps only) |
| `/v1/evm/token-info/{address}` | `token` + `getTokenPrices` |
| `/v1/evm/token-holders/{chain}/{address}` | `holders` |
| `/v1/evm/search/tokens` | `filterTokens` |
| `/v1/evm/supported-chains` | `getNetworks` |
| `/v1/evm/defi/positions/{address}` | Partial: `liquidityMetadata`, `liquidityMetadataByToken` |
| Balances webhook | `onBalanceUpdated` or a `TOKEN_TRANSFER_EVENT` webhook |
| Activities webhook | `onEventsCreatedByMaker` or a `TOKEN_PAIR_EVENT` webhook |
| `/v1/evm/collectibles/*`, `/v1/evm/defi/supported-protocols`, Transactions webhook | Not supported |

Not covered: NFTs, per-wallet DeFi positions, raw transactions and decoded calls, a dedicated stablecoin endpoint.

## Serialized

Guide: https://docs.codex.io/migrations/serialized

| Serialized | Codex |
|---|---|
| `/v1/token` | `filterTokens(tokens: [...])`, or `token` + `getDetailedTokenStats` |
| `/v1/token/price`, `/v1/prices/native` | `getTokenPrices` (wrapped native token for native prices) |
| `/v1/token/stats` | `getDetailedTokenStats` |
| `/v1/token/metadata` | `token`, `tokens` |
| `/v1/token/sparklines` | `tokenSparklines` |
| `/v1/search` | `filterTokens(phrase:)` |
| `/v1/token/security` | `token` (+ `risk`) + `liquidityMetadataByToken` |
| `/v1/token/dev-tokens` | `filterTokens(filters: { creatorAddresses })` + `detailedWalletStats` |
| `/v1/token/holders` | `holders` + `filterTokenWallets` |
| `/v1/token/top-traders` | `tokenTopTraders` |
| `/v1/pool/top-traders` | `filterTokenWallets` (per token, not per pool) |
| `/v1/token/pools` | `listPairsWithMetadataForToken` |
| `/v1/pool` | `pairMetadata` |
| `/v1/pool/data`, `/v1/pools/data` | `getDetailedPairStats`, `getDetailedPairsStats` |
| `/v1/token/ohlcv` / `/v1/pool/ohlcv` | `getTokenBars` / `getBars` |
| `/v1/token/trades`, `/v1/pool/trades`, `/v1/pools/trades` | `getTokenEvents` (`query.maker` for one wallet) |
| `/v1/wallet/positions` | `balances` + `filterTokenWallets` |
| `/v1/wallet/closed-positions` | `filterTokenWallets` (rows with zero `tokenBalanceLive`) |
| `/v1/wallet/trades` | `getTokenEventsForMaker` |
| `/v1/wallet/pnl`, `/v1/wallet/equity/history` | `detailedWalletStats`, `walletChart` |
| `/v1/wallet/profile`, `/v1/wallet/funding` | `detailedWalletStats { wallet }` (`identityLabels`, `firstFunding`) |
| `/v1/pulse`, `/v1/screener` | `filterTokens` (launchpad filters for pulse), `filterPairs` |
| `/v1/meta/factories` | `filterLaunchpads` + `getExchanges` |
| `/v1/meta/chains` | `getNetworks` |
| `/v1/usage*` | Usage page in the Codex dashboard |
| `/v1/audit/*`, `/v1/wallet/transfers`, `/v1/pools/sparklines` | Not supported |

Not covered: contract source auditing, deployer risk verdicts, wallet transfers, ENS / Basename / `.sol` resolution (Codex returns a resolved `displayName`), wash-trade flags, pool-level top traders and sparklines.
