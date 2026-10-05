# Migrating to Codex from another provider

Use when the user is moving from, replacing, or comparing against Birdeye, Bitquery, CoinGecko, Dune Sim, Mobula, or Serialized. Each list maps a source endpoint to the Codex operation that covers it; look the Codex operation up in query-templates.md or the schema before writing a query. The full guides, with side-by-side examples, gaps and an AI migration prompt, are at https://docs.codex.io/migrations/<provider> (birdeye, bitquery, coingecko, dune-sim, mobula, serialized).

Conventions that trip every migration: network ids are integers (run `getNetworks`); token and pair ids are `address:networkId`; timestamps are unix seconds; USD amounts are strings; percent changes are decimals (0.05 = 5%); `getTokenPrices` takes at most 25 inputs; `filterTokens` returns at most 200 rows per page; wallet, holder and balance queries need a Growth or Enterprise plan.


## Birdeye

Triggers: "migrating from Birdeye", "Birdeye equivalent". Guide: https://docs.codex.io/migrations/birdeye


Prices and OHLCV:
- `GET /defi/price` → `getTokenPrices` — Pass a single-element array.
- `GET /defi/multi_price`, `POST /defi/multi_price` → `getTokenPrices` — Native batch input, max 25 tokens per call (anything over is truncated…
- `GET /defi/historical_price_unix` → `getTokenPrices` with a `timestamp` input — Pass the unix timestamp on the input to get the price at that moment.
- `GET /defi/history_price` → `getTokenBars` — Token-level OHLCV; pair-level via `getBars`.
- `GET /defi/ohlcv`, `/defi/v3/ohlcv` → `getTokenBars` — Codex supports 1-second up to weekly (`7D`) intervals.
- `GET /defi/ohlcv/pair`, `/defi/v3/ohlcv/pair` → `getBars` — Pair-scoped OHLCV.
- `GET /defi/ohlcv/base_quote` → `getBars` with `quoteToken` — Invert the pair to quote in the other token.
- `GET /defi/price_volume/single`, `POST /defi/price_volume/multi` → `getDetailedTokenStats` — Price and volume in one query, plus much more.
- `GET /defi/v3/price/stats/single`, `POST /defi/v3/price/stats/multiple` → `getDetailedTokenStats` — Stats over multiple timeframes.

Token data:
- `GET /defi/token_overview` → `token` + `getDetailedTokenStats` — One GraphQL request returns metadata, stats, safety, and launchpad con…
- `GET /defi/v3/token/meta-data/single` → `token` — Richer payload than Birdeye's, including social links and image URLs.
- `GET /defi/v3/token/meta-data/multiple` → `tokens(ids: [{ address, networkId }])` — Batch token metadata.
- `GET /defi/v3/token/market-data` (single + multiple) → `getDetailedTokenStats`
- `GET /defi/v3/token/trade-data/single`, `/defi/v3/token/trade-data/multiple` → `getDetailedTokenStats` — Trade stats are part of detailed token stats.
- `GET /defi/token_security` → `token` — Safety fields (`isScam`, `mintable`, `freezable`, `creatorAddress`, to…
- `GET /defi/token_creation_info` → `token` — `createdAt`, `creatorAddress`.
- `GET /defi/v3/token/exit-liquidity`, `/defi/v3/token/exit-liquidity/multiple` → `liquidityMetadata` + `liquidityMetadataByToken` — Plus `liquidityLocksV2` for locked-LP context.

Discovery and search:
- `GET /defi/token_trending` → `filterTokens` ranked by `trendingScore24`
- `GET /defi/v3/token/list`, `GET /defi/v3/token/list/scroll`, `GET /defi/tokenlist` → `filterTokens` — Filters and rankings collapse into one query.
- `GET /defi/v2/tokens/new_listing` → `filterTokens` ranked by `createdAt` — Or subscribe to `onTokenLifecycleEventsCreated` / `onLatestTokens`.
- `GET /defi/v3/search` → `filterTokens(phrase: ...)` — Use `$SYMBOL` for exact symbol matches.
- `GET /defi/v3/token/meme/detail/single`, `GET /defi/v3/token/meme/list` → `filterTokens` + launchpad context — Codex models meme launches as launchpad lifecycle events (pump.fun, Le…
- `GET /smart-money/v1/token/list` → `filterTokens` plus `filterWallets` —  for the smart-money pattern.

Pairs and markets:
- `GET /defi/v3/pair/overview/single` → `getDetailedPairStats` — Pair-level trade stats (volume, buys/sells, price change) are included…
- `GET /defi/v3/pair/overview/multiple` → `getDetailedPairsStats`
- `GET /defi/v2/markets` → `listPairsForToken` + `listPairsWithMetadataForToken` — All venues for a token.

Trades:
- `GET /defi/txs/token`, `GET /defi/v3/token/txs`, `GET /defi/txs/token/seek_by_time` → `getTokenEvents` — Swap and lifecycle events for a token; pass a `timestamp` range for th…
- `GET /defi/txs/pair`, `GET /defi/txs/pair/seek_by_time` → `getTokenEvents` — Pass the pair address in `query: { address: ... }`.
- `GET /defi/v3/txs`, `GET /defi/v3/txs/recent` → `getTokenEvents`
- `GET /defi/v3/token/txs-by-volume` → `getTokenEvents` with `priceUsdTotal` filter — Filter swaps by USD size.
- `GET /defi/v3/token/mint-burn-txs` (Solana) → `getTokenEvents` with `eventDisplayType: [Mint, Burn]` — Mint/burn lifecycle events.
- `GET /defi/v3/all-time/trades/single`, `POST /defi/v3/all-time/trades/multiple` → `getDetailedTokenStats` (`statsUsd.volume`, `statsNonCurrency.transactions`) — Aggregate metrics, not raw rows.

Holders:
- `GET /defi/v3/token/holder` → `holders` — Ranked holder list.
- `GET /holder/v1/distribution` → `top10HoldersPercent` field on `token` — Concentration in one field.
- `GET /token/v1/holder-profile` → `top10HoldersPercent` on `token` + `holders` — Combine concentration with the ranked holder list.
- `GET /token/v1/holder/chart` → Partial via `onHoldersUpdated` — Codex streams live holder counts; historical timeseries is not a first…
- `GET /token/v1/holder-positions` → `filterTokenWallets` — Wallets ranked by per-token PnL.
- `POST /token/v1/holder/batch` → `balances` (per wallet) — One call per wallet; aliases let you batch in a single GraphQL request.

Wallets:
- `GET /v1/wallet/token_list` → `balances` — Wallet portfolio with prices. Native balances on EVM chains require tr…
- `GET /v1/wallet/token_balance`, `POST /wallet/v2/token-balance` → `balances` with `tokens: [...]` — Pass the token IDs (`address:networkId`) you want; max 200 per request.
- `GET /v1/wallet/list_supported_chain` → `getNetworks` — Network catalog used everywhere else in the API.
- `GET /v1/wallet/tx_list` → `getTokenEventsForMaker` — Codex returns swap events for a wallet; raw transfers are not exposed.
- `GET /wallet/v2/current-net-worth` → `detailedWalletStats` — PnL, volume, swap counts.
- `GET /wallet/v2/net-worth`, `/wallet/v2/net-worth-details`, `POST /wallet/v2/net-worth-summary/multiple` → `walletChart` + `detailedWalletStats` — Use GraphQL aliases to batch multiple wallets in one request.
- `GET /wallet/v2/pnl`, `/wallet/v2/pnl/summary`, `GET /wallet/v2/pnl/multiple`, `POST /wallet/v2/pnl/details` → `detailedWalletStats` + `filterWallets`
- `GET /wallet/v2/pnl/chart` → `walletChart` — PnL and volume over time; resolutions `60`, `240`, `1D`, `7D`.
- `GET /wallet/v2/leaderboard` → `filterWallets` ranked by `realizedProfitUsd*` — Discover top wallets across all networks, not just a preset board.
- `GET /wallet/v2/balance-change` → `onBalanceUpdated` subscription — Live balance changes; historical reconstruction requires combining eve…
- `POST /wallet/v2/tx/first-funded` → `wallet.firstFunding` on `detailedWalletStats` — Available for indexed wallets (wallets with trading activity), mostly…

Traders:
- `GET /defi/v2/tokens/top_traders` → `tokenTopTraders` — Top buyers/sellers/PnL for a token.
- `GET /trader/txs/seek_by_time` → `getTokenEventsForMaker`
- `GET /trader/gainers-losers` → `filterWallets` ranked by `realizedProfitUsd*` — Far richer filtering than Birdeye's endpoint.

Utility:
- `GET /defi/networks` → `getNetworks` — Returns `networkId`s you'll use everywhere else.
- `GET /defi/v3/txs/latest-block` → `blocks` — Look up blocks by `blockNumbers` or `timestamps`; pass the current tim…
- `GET /utils/v1/credits` → Codex usage in the dashboard

Not covered by Codex: Perpetuals data** (`/perps/v1/*`). Codex is a spot-trading API. If your product depends on open positions, liq…; Wallet-level transfers (non-swap)** (`/wallet/v2/transfer*`, `/token/v1/transfer*`). Codex returns swap and to…; RPC-style blockchain data** (`/blockchain/v1/account/*`, `/blockchain/v1/transaction/detail`, `/blockchain/v1/…; Wallet identity and domain resolution** (`/identity/v1/single`, `/identity/v1/multiple`, `/identity/v1/domains…; First-buyers and wallet-tag analytics** (`/token/v1/first-buyers`, `/token/v1/wallet-tags-tracker`). No first-…; Historical liquidity timeseries** (`/defi/v3/liquidity/ohlc/*`, `/defi/v3/liquidity/history/token`). Codex exp…

## Mobula

Triggers: "migrating from Mobula", "Mobula equivalent". Guide: https://docs.codex.io/migrations/mobula


Prices and market data:
- `GET /1/market/data` → `getTokenPrices` + `getDetailedTokenStats` — Single-asset price and market stats. Metadata via `token`.
- `GET /1/market/multi-data` → `getTokenPrices` — Native batch input; pass an array. Enrich with `tokens` for metadata.
- `GET /1/market/multi-prices`, `POST /1/market/multi-prices` → `getTokenPrices` — Max 25 inputs per call (anything over is truncated); chunk larger batc…
- `GET /2/token/price`, `POST /2/token/price` → `getTokenPrices` — Current USD price for a token.
- `GET /2/token/price-at`, `POST /2/token/price-at` → `getTokenPrices` with a `timestamp` input — Pass the unix timestamp on the input to get the price at that moment.
- `GET /2/market/details`, `POST /2/market/details` → `getDetailedTokenStats` or `getDetailedPairStats` — Token-level vs. pair-level stats depending on whether you pass a token…
- `GET /2/token/details`, `POST /2/token/details` → `token` + `getDetailedTokenStats` — One GraphQL request returns metadata, stats, safety, and launchpad con…
- `GET /2/asset/details`, `POST /2/asset/details` → `token` — Asset-level metadata.
- `GET /1/market/sparkline` → `tokenSparklines` — Compact price series for sparkline UIs.
- `GET /2/token/ath`, `POST /2/token/ath` → `filterTokens` — ATH/ATL are first-class: `athPrice`, `atlPrice`, `athFdv`, `atlFdv`, `…
- `GET /2/market/lighthouse` → `filterTokens` + `getDetailedTokenStats` — No single "quality score", but `filterTokens` exposes the raw signals…

Token metadata and search:
- `GET /1/metadata` → `token` — Metadata, social links, and image URLs on one object.
- `GET /1/multi-metadata` → `tokens(ids: [{ address, networkId }])` — Batch token metadata.
- `GET /1/all` → `filterTokens` — Codex doesn't dump the full asset universe; filter to what you need.
- `GET /1/blockchains` → `getNetworks` — Returns the `networkId`s you use everywhere else.
- `GET /1/search`, `GET /2/fast-search`, `POST /2/fast-search` → `filterTokens(phrase: ...)` — Use `$SYMBOL` for exact symbol matches; combine with rankings and filt…
- `GET /2/token/security` → `token` + `filterTokens` — Safety flags (`isScam`, `mintable`, `freezable`, `creatorAddress`, `to…
- `GET /2/token/logo-reuses` → Partial via `filterTokens` — No logo-fingerprint match, but `potentialScam`, `isScam`, and holder-c…
- `GET /1/metadata/categories` → `filterTokens` — Category surfaces map to filters/rankings rather than a taxonomy dump.
- `GET /1/metadata/news` → Not supported — Codex is market/onchain data, not editorial news.

Pairs and markets:
- `GET /1/market/pair` → `getDetailedPairStats` — Pair-level trade stats (volume, buys/sells, price change). Static meta…
- `GET /1/market/pairs`, `GET /2/token/markets` → `listPairsForToken` + `listPairsWithMetadataForToken` — All venues for a token.
- `GET /1/market/blockchain/pairs` → `filterPairs` — Discover and rank pairs across a network with filters and sorting.
- `GET /1/market/blockchain/stats` → `getNetworkStats` — Per-network trading activity.

Charts and OHLCV:
- `GET /1/market/history` → `getTokenBars` — Token-level OHLCV including volume.
- `GET /1/market/multi-history` → `getTokenBars` — One call per token; GraphQL aliases batch them in a single request.
- `GET /2/token/price-history`, `POST /2/token/price-history`, `GET /2/asset/price-history` → `getTokenBars` — Token/asset price series.
- `GET /2/token/ohlcv-history`, `POST /2/token/ohlcv-history` → `getTokenBars` — Codex supports 1-second up to weekly (`7D`) resolutions.
- `GET /1/market/history/pair`, `GET /2/market/ohlcv-history`, `POST /2/market/ohlcv-history` → `getBars` — Pair-scoped OHLCV; use `quoteToken` to invert the pair.

Trades:
- `GET /2/token/trades`, `POST /2/token/trades` → `getTokenEvents` — Swap events for a token; filter by USD size, direction, and time range.
- `GET /2/token/trades-enriched`, `POST /2/token/trades-enriched` → `getTokenEvents` + `filterTokenWallets` — Codex returns maker addresses on events; join per-wallet PnL from `fil…
- `GET /2/token/trade` → `getTokenEvents` — Filter to a single transaction hash.
- `GET /1/market/trades/pair` → `getTokenEvents` — Pass the pair address in `query: { address: ... }`.
- `GET /2/trades/filters` → `getTokenEvents` — Bulk trade pulls with the same filter set.

Holders and traders:
- `GET /2/token/holder-positions`, `POST /2/token/holder-positions` → `holders` + `filterTokenWallets` — `holders` for the ranked balance list; `filterTokenWallets` for per-to…
- `GET /2/token/trader-positions`, `POST /2/token/trader-positions` → `tokenTopTraders` — Top buyers/sellers/PnL for a token.
- `GET /1/token/first-buyers` → Partial via `getTokenEvents` — Approximate from the earliest events for a token; no curated first-buy…

Wallets:
- `GET /1/wallet/portfolio`, `GET /2/wallet/holdings`, `POST /2/wallet/holdings` → `balances` — Wallet portfolio with prices. Native balances on EVM chains require tr…
- `GET /1/wallet/multi-portfolio` → `balances` — Alias multiple wallets in one GraphQL request.
- `GET /1/wallet/history` → `walletChart` — Net worth / PnL over time; resolutions `60`, `240`, `1D`, `7D`.
- `GET /2/wallet/positions`, `POST /2/wallet/positions`, `GET /2/wallet/position` → `detailedWalletStats` + `filterTokenWallets` — Aggregate PnL/volume from `detailedWalletStats`; per-token positions f…
- `GET /2/wallet/positions-history`, `GET /2/wallet/position-history` → `walletChart` — Historical PnL/value curve.
- `GET /2/wallet/analysis` → `detailedWalletStats` — Win rate, PnL, swap counts, volume.
- `GET /1/wallet/trades`, `GET /2/wallet/trades`, `POST /2/wallet/trades` → `getTokenEventsForMaker` — Swap events for a wallet; filter by token if needed.
- `GET /2/wallet/activity`, `POST /2/wallet/activity` → `getTokenEventsForMaker` — Codex returns swap and lifecycle activity, not arbitrary transfers.
- `GET /1/wallet/transactions` → `getTokenEventsForMaker` — Swap events; raw transfers are not exposed (see Gaps).
- `POST /1/wallet/labels`, `GET /2/wallet/labels`, `GET /2/wallet/labels/search` → Partial via `filterWallets` — Codex discovers wallets by behavior/performance rather than curated en…
- `GET /2/wallet/defi-positions` → Not supported — Codex tracks token balances and swaps, not DeFi protocol positions (st…

Discovery, screening, and launchpads:
- `GET /1/market/query`, `GET /1/all` → `filterTokens` — Screener-style filtering and ranking in one query.
- `GET /1/search`, `GET /2/fast-search`, `POST /2/fast-search` → `filterTokens(phrase: ...)` — Search is a `phrase` argument on the same endpoint; use `$SYMBOL` for…
- `GET /2/pulse`, `POST /2/pulse`, `GET /1/pulse` → `filterTokens` + launchpad context — Filter on launchpad fields for new/bonding/bonded discovery (pump.fun…
- `GET /2/token/dev-history` → Partial via `getTokenEventsForMaker` — Query the deployer address as a maker; no dedicated deployer-history e…

Utility:
- `GET /1/blockchains` → `getNetworks` — Network catalog with `networkId`s.
- `GET /2/usage` → Codex usage in the dashboard — Credit and request metering live in the dashboard.
- `GET /metadata`, `GET /2/system-metadata` → `getNetworks` + API Reference — System-level config is exposed per-resource rather than in one metadat…

Not covered by Codex: Perpetuals data and execution** (`/2/perp/*`, `/2/wallet/positions/perp/*`, `/1/market/cefi/funding-rate`, per…; Swap and bridge execution** (`/2/swap/quoting`, `/2/swap/send`, `/2/bridge/*`). Codex is read-only market data…; Prediction-market trading** (Mobula's `/2/pm/*` CLOB order/auth/redeem flow). Codex exposes Polymarket and Kal…; Raw transfers and raw transactions** (`/1/wallet/raw-transactions`, `/1/wallet/token-transfers`, `/1/wallet/nf…; Wallet intelligence** (`/2/wallet/funding` funding-source tracing, `/2/wallet/deployer`, `/2/wallet/labels` an…; DeFi protocol positions** (`/2/wallet/defi-positions`). Codex tracks token balances and swap activity, not sta…

## CoinGecko

Triggers: "migrating from CoinGecko", "CoinGecko equivalent". Guide: https://docs.codex.io/migrations/coingecko


OnChain DEX API: tokens:
- CoinGecko OnChain → Codex equivalent — Notes
- `GET /onchain/networks/{network}/tokens/{address}` → `token` — Richer metadata including safety signals, launchpad context, social li…
- `GET /onchain/networks/{network}/tokens/multi/{addresses}` → `tokens` — Batch token metadata.
- `GET /onchain/networks/{network}/tokens/{address}/info` → `token`
- `GET /onchain/simple/networks/{network}/token_price/{addresses}` → `getTokenPrices` — For the `include_*` flags (`include_market_cap`, `include_24hr_vol`, `…
- `GET /onchain/networks/{network}/tokens/{address}/ohlcv/{timeframe}` → `getTokenBars`
- `GET /onchain/networks/{network}/tokens/{address}/trades` → `getTokenEvents` — CoinGecko's token-level trades endpoint is Analyst+ and capped at the…
- `GET /onchain/networks/{network}/tokens/{address}/top_holders` → `holders` (+ `top10HoldersPercent` for the aggregate)
- `GET /onchain/networks/{network}/tokens/{address}/top_traders` → `tokenTopTraders` — Supports time-range filters and PnL data.
- `GET /onchain/networks/{network}/tokens/{address}/holders_chart` → Partial via `onHoldersUpdated` — Live count; historical chart is not a first-class endpoint.
- `GET /onchain/tokens/info_recently_updated` → `filterTokens` ranked by recent activity — Cross-network on CoinGecko; filter by `network` in Codex if you need t…

OnChain DEX API: pools and pairs:
- CoinGecko OnChain → Codex equivalent — Notes
- `GET /onchain/networks/{network}/pools/{pool}` → `getDetailedPairStats`
- `GET /onchain/networks/{network}/pools/multi/{addresses}` → `getDetailedPairsStats`
- `GET /onchain/networks/{network}/tokens/{address}/pools` → `listPairsForToken`
- `GET /onchain/networks/{network}/pools/{pool}/info` → `pairMetadata`
- `GET /onchain/networks/{network}/pools/{pool}/trades` → `getTokenEvents` with a pair filter — Last 300 trades over 24 hours vs. Codex's full event history.
- `GET /onchain/networks/{network}/pools/{pool}/ohlcv/{timeframe}` → `getBars` — Both APIs cover sub-minute candles; Codex extends one step further wit…
- `GET /onchain/networks/{network}/pools` → `filterPairs` with `filters: { network: [<id>] }` — Top pools on a single network.
- `GET /onchain/networks/{network}/new_pools`, `/onchain/networks/new_pools` → `filterPairs` ranked by `createdAt` — Or subscribe to `onTokenLifecycleEventsCreated`.
- `GET /onchain/networks/trending_pools`, `/onchain/networks/{network}/trending_pools` → `filterPairs` ranked by `trendingScore24`
- `GET /onchain/pools/trending_search` → `filterPairs(phrase: ..., rankings: [{ attribute: trendingScore24 }])` — Trending matches for a search phrase.
- `GET /onchain/networks/{network}/dexes/{dex}/pools` → `filterPairs` filtered by exchange
- `GET /onchain/pools/megafilter` → `filterPairs` — Rich filter clauses across all networks.

OnChain DEX API: networks, dexes, search:
- CoinGecko OnChain → Codex equivalent — Notes
- `GET /onchain/networks` → `getNetworks`
- `GET /onchain/networks/{network}/dexes` → `filterExchanges`
- `GET /onchain/search/pools` → `filterPairs(phrase: ...)`
- `GET /onchain/categories`, `GET /onchain/categories/{id}/pools` → Not supported — See Gaps — Codex doesn't curate DEX pool categories.

CoinGecko market data (coin-ID based):
- `GET /simple/price?ids=bitcoin,ethereum` → `getTokenPrices` (+ `filterTokens`) — Resolve coin IDs to contract addresses first; see Coin IDs vs contract…
- `GET /simple/token_price/{platform}?contract_addresses=...` → `getTokenPrices` — Direct mapping; `{platform}` becomes `networkId`. Same caveat as above…
- `GET /coins/{id}` → `token` + `getDetailedTokenStats` (+ `filterTokens` for market cap) — Resolve ID first. `token` returns metadata, safety, launchpad; `getDet…
- `GET /coins/{id}/market_chart`, `/market_chart/range` → `getTokenBars` — OHLCV from 1-second up to weekly (`7D`).
- `GET /coins/{id}/ohlc`, `/ohlc/range` → `getTokenBars`
- `GET /coins/{id}/contract/{address}/market_chart`, `/market_chart/range` → `getTokenBars` — Already contract-addressed; no slug resolution.
- `GET /coins/{id}/history?date=...` → `getBars` with a single bar covering the date
- `GET /coins/markets` → `filterTokens` with ranking — Filter by network, liquidity, market cap; rank by volume, trending, etc.
- `GET /coins/list`, `/token_lists/{asset_platform_id}/all.json` → `filterTokens` (optionally `filters: { network: [<id>] }`) — Filter at the point of use instead of maintaining a full list.
- `GET /coins/list/new` → `filterTokens` ranked by `createdAt`, or `onTokenLifecycleEventsCreated`
- `GET /coins/top_gainers_losers` → `filterTokens` ranked by `change24` — `change24` is the 24h price-change ranking attribute on tokens (the pa…
- `GET /coins/{id}/tickers` → `listPairsForToken` + `listPairsWithMetadataForToken` — DEX pairs only; CEX tickers are out of scope.
- `GET /coins/{id}/contract/{contract_address}` → `token` — The contract-address form is already the Codex native shape.
- `GET /coins/{id}/circulating_supply_chart`, `/total_supply_chart` (+ `/range`) → Not supported — See Gaps — no historical supply timeseries. Current supply is on `toke…
- `GET /search?query=...` → `filterTokens(phrase: "$SYMBOL", ...)` — Use `$SYMBOL` prefix for exact symbol matches.
- `GET /search/trending` → `filterTokens(rankings: [{ attribute: trendingScore24, direction: DESC }])` — Tokens only; trending NFTs and categories are out of scope.
- `GET /coins/categories`, `/coins/categories/list` → Not supported — See Gaps.
- `GET /simple/supported_vs_currencies`, `/exchange_rates` → Not supported — Codex returns USD only. Pair with an FX provider.

Utilities:
- `GET /asset_platforms` → `getNetworks` — Returns `networkId`s you'll use everywhere else.
- `GET /ping` → Not needed — Codex doesn't require health-checking.
- `GET /key` → Codex usage in the dashboard — CoinGecko's `/key` returns your plan's usage, rate limits, and remaini…

Not covered by Codex: CEX tickers, exchange metadata, and order-book data** (`/coins/{id}/tickers` for centralized exchanges, `/exch…; Derivatives and futures** (`/derivatives`, `/derivatives/exchanges`, `/derivatives/exchanges/{id}`, `/derivati…; NFTs** (`/nfts/*`, NFT floor prices, collection data, trending NFTs from `/search/trending`). Codex is a fungi…; Treasury holdings and entity data** (the current entity-based surface: `/entities/list`, `/{entity}/public_tre…; Global market data aggregates** (`/global`, `/global/decentralized_finance_defi`, `/global/market_cap_chart`).…; Historical supply timeseries** (`/coins/{id}/circulating_supply_chart`, `/total_supply_chart`, and their `/ran…

## Bitquery

Triggers: "migrating from Bitquery", "Bitquery equivalent". Guide: https://docs.codex.io/migrations/bitquery


DEX trades and events:
- Bitquery dataset / pattern → Codex equivalent — Notes
- `EVM { DEXTrades }`, `Solana { DEXTrades }` → `getTokenEvents` — One normalized swap event per trade. `DEXTrades` is one row per swap…
- `EVM { DEXTradeByTokens }`, `Solana { DEXTradeByTokens }` → `getTokenEvents` (+ `getDetailedTokenStats` for aggregates) — `DEXTradeByTokens` emits two rows per trade (one per side) for easier…
- `DEXTrades` filtered by `Trade.Buy.Buyer` / trader → `getTokenEventsForMaker` — Swap events for a single wallet.
- `Solana { Instructions }` for a DEX program → `getTokenEvents` with `eventDisplayType` — Codex decodes swap/mint/burn events for you; raw instruction decoding…

Prices and OHLCV:
- Bitquery dataset / pattern → Codex equivalent — Notes
- `Trade.PriceInUSD` read off the latest `DEXTrades` row → `getTokenPrices` — Direct current price by `address:networkId`; up to 25 inputs per reque…
- Historical price via `DEXTrades` at a past block/time → `getTokenPrices` with a `timestamp` input — Price at a moment, without reconstructing it from trades.
- Crypto Price API `Trade { Price { Ohlc { … } } }`, or hand-rolled OHLC from `DEXTradeByTokens` bucketed by `Block { Time(interval:) }` → `getTokenBars` (token) / `getBars` (pair) — First-class OHLCV, resolutions `1S`–`7D`. No candle assembly.

Token data:
- Bitquery dataset / pattern → Codex equivalent — Notes
- `Currencies` / token fields on trades → `token` + `tokens` — Richer metadata: social links, image URLs, description, launchpad cont…
- `TokenSupplyUpdates` (supply) → `token.info` (`circulatingSupply`, `totalSupply`) — Current supply on the token object.
- No first-class equivalent (infer from raw data) → Safety fields on `token` — `isScam`, `mintable`, `freezable`, `creatorAddress`, `top10HoldersPerc…
- Aggregated `DEXTradeByTokens` for volume/txn stats → `getDetailedTokenStats` — Multi-timeframe volume, transactions, buy/sell counts in one call.

Holders and balances:
- Bitquery dataset / pattern → Codex equivalent — Notes
- `EVM { Holders }` (formerly `TokenHolders(date:)`, removed June 2026) → `holders` (+ `top10HoldersPercent`) — Ranked holders with balances; concentration returned on the same respo…
- `BalanceUpdates` aggregated to a wallet's current holdings → `balances` — Portfolio with USD pricing inline (`balanceUsd`, `tokenPriceUsd`); pas…
- `BalanceUpdates` reconstructed at a past date → Not supported — Codex returns current balances; historical point-in-time balances are…
- `Transfers` (arbitrary token transfers) → `getTokenEvents` / `getTokenEventsForMaker` — Codex indexes DEX swap and lifecycle events, not arbitrary transfers.…

Wallets and traders:
- Bitquery dataset / pattern → Codex equivalent — Notes
- `DEXTrades` grouped by trader, PnL computed yourself → `detailedWalletStats` — Realized PnL, volume, swap counts as first-class fields.
- Wallet performance over time (hand-rolled) → `walletChart` — PnL and volume time-series; resolutions `60`, `240`, `1D`, `7D`.
- Top traders for a token (aggregated `DEXTradeByTokens`) → `tokenTopTraders` — Top buyers/sellers/PnL, `tradingPeriod` = `DAY \
- "Find profitable wallets" (custom aggregation) → `filterWallets` — Query wallets by PnL, win-rate, or volume across all networks.

Discovery, pairs, and markets:
- Bitquery dataset / pattern → Codex equivalent — Notes
- Aggregated `DEXTradeByTokens` ("GMGN-style" trending queries) → `filterTokens` ranked by `trendingScore24` — First-class ranked discovery; no aggregation to write. See Discover To…
- Custom token search via `Currencies` filters → `filterTokens(phrase: ...)` — Use `$SYMBOL` for exact symbol matches.
- DEX pool / market fields on trades → `pairMetadata`, `getDetailedPairStats`, `listPairsForToken` — First-class pair concept with stats over multiple timeframes.
- Ranked pool discovery (aggregated) → `filterPairs` — Rich filter clauses across all networks.
- DEX/protocol list (from `Trade.Dex`) → `filterExchanges`
- Chain list → `getNetworks` — Returns the `networkId`s you'll use everywhere else.
- Block lookups (`Block` fields) → `blocks` — Look up blocks by number or timestamp.

Launchpads and prediction markets:
- Bitquery dataset / pattern → Codex equivalent — Notes
- pump.fun via `Solana { Instructions }` (program `pump`) + `TokenSupplyUpdates` → `onLaunchpadTokenEvent` + `launchpad` fields on `token` — Codex abstracts the full launchpad lifecycle (bonding curve, graduatio…
- Polymarket via `EVM(network: matic)` events (`OrderFilled`, `ConditionResolution`) → `filterPredictionEvents` family — Higher-level event/market/trader data covering **both Polymarket and K…

Not covered by Codex: Mempool / pending transactions** (`EVM(mempool: true)`). Codex indexes confirmed onchain data, not the pending…; Raw and decoded logs, calls, and instructions across arbitrary contracts** (`Events`, `Calls`, Solana `Instruc…; Ad-hoc aggregations over any onchain field.** Bitquery's cube model lets you group and aggregate almost anythi…; Historical point-in-time balances.** Bitquery reconstructs a wallet's balance at any past date from `BalanceUp…; NFT trades and collection analytics.** Codex is a fungible-token API. Pair with a dedicated NFT provider (Rese…; Non-EVM/UTXO chains Codex doesn't index** (Bitcoin, Litecoin, Cardano, Tron, XRP, and others). Codex covers 80…

## Dune Sim

Triggers: "migrating from Dune Sim", "Dune Sim equivalent". Guide: https://docs.codex.io/migrations/dune-sim

- Sim endpoint → Codex equivalent — Notes
- `GET /v1/evm/balances/{address}` → `balances` query — Native + ERC-20 with USD pricing inline (`balanceUsd`, `tokenPriceUsd`…
- `GET /v1/evm/balances/{address}/token/{token_address}` → `balances` with `tokens: ["<address>:<networkId>"]` — Same query, narrowed to specific tokens (up to 200 per request).
- `GET /v1/evm/balances/{address}/stablecoins` → `balances` with a curated stablecoin `tokens` list, or `filterTokens` — No dedicated stablecoin endpoint; pass the stablecoin token IDs you ca…
- `GET /v1/evm/activity/{address}` → `getTokenEventsForMaker` query — Sim returns transfers, NFT moves, approvals, swaps, and decoded contra…
- `GET /v1/evm/transactions/{address}` → `getTokenEventsForMaker` query — Same DEX-only caveat. Codex doesn't expose raw transactions with gas…
- `GET /v1/evm/collectibles/{address}` → Not supported — Codex is a fungible-token API. See "Gaps" below.
- `GET /v1/evm/token-info/{address}?chain_ids=...` → `token` query + `getTokenPrices` — Codex returns richer metadata: safety signals, launchpad context, 19 s…
- `GET /v1/evm/token-holders/{chain_id}/{address}` → `holders` query — Returns ranked holders with balances. `top10HoldersPercent` is returne…
- `GET /v1/evm/search/tokens?query=...` → `filterTokens` query — Far more powerful: rank by trending score, volume, market cap, plus fi…
- `GET /v1/evm/defi/positions/{address}` → Partial via `liquidityMetadata` / `liquidityMetadataByToken` — Codex exposes pair-level liquidity and lock breakdowns, not aggregated…
- `GET /v1/evm/defi/supported-protocols` → Not directly supported — Codex doesn't aggregate per-wallet DeFi positions, so there's no proto…
- `GET /v1/evm/supported-chains` → `getNetworks` query — Returns the full list of networks Codex indexes, including chain IDs.…
- `GET /beta/svm/balances/{address}` → `balances` with `networks: [1399811149]` — Same query, different network ID. Note Sim's SVM endpoints use `chains…
- `GET /beta/svm/transactions/{address}` → `getTokenEventsForMaker` — Same shape as EVM; Sim's response wraps raw RPC data, Codex returns de…
- Sim Balances webhook (`POST /beta/evm/subscriptions/webhooks`, `type: balances`) → `onBalanceUpdated` subscription or `createWebhooks` with `TOKEN_TRANSFER_EVENT` — Codex's `TOKEN_TRANSFER_EVENT` filters by target wallet with direction…
- Sim Activities webhook (`type: activities`) → `onEventsCreatedByMaker` subscription or `TOKEN_PAIR_EVENT` webhook — Subscription gives per-wallet streams; `TOKEN_PAIR_EVENT` accepts a `m…
- Sim Transactions webhook (`type: transactions`) → No direct equivalent — Codex doesn't fan out raw txs. Closest is `TOKEN_TRANSFER_EVENT` for t…

Not covered by Codex: NFTs (ERC-721 / ERC-1155 collectibles).** Codex is a fungible-token API. If NFT data is core to your product,…; Aggregated per-wallet DeFi positions.** Codex exposes pair-level liquidity via `liquidityMetadata` and `liquid…; Raw transaction-level data and decoded contract calls.** Codex returns trading events, not every transaction a…; Dedicated stablecoin endpoint.** Use `filterTokens` with a maintained list of stablecoin addresses.

## Serialized

Triggers: "migrating from Serialized", "Serialized equivalent". Guide: https://docs.codex.io/migrations/serialized


Token snapshot, prices, and metadata:
- `GET /v1/token`, `POST /v1/token` → `filterTokens(tokens: [...])` or `token` + `getDetailedTokenStats` — `filterTokens` with a `tokens` list returns metadata, price, market ca…
- `GET /v1/token/price`, `POST /v1/token/price` → `getTokenPrices` — Lean price tier. Max 25 inputs per call; chunk larger batches. Market…
- `GET /v1/token/stats` → `getDetailedTokenStats` — Serialized returns 5m / 1h / 6h / 24h. Codex returns 5m / 1h / 4h / 12…
- `GET /v1/token/metadata`, `POST /v1/token/metadata` → `token`, `tokens` — `info` (supply, images, description) and `socialLinks` on the same obj…
- `POST /v1/token/sparklines`, `POST /v1/pools/sparklines` → `tokenSparklines` — Token-level sparklines. Pool-level sparklines have no direct twin; use…
- `GET /v1/prices/native` → `getTokenPrices` — Pass each network's wrapped native token (WETH, WSOL, WBNB) as an input.
- `GET /v1/search` → `filterTokens(phrase: ...)` — Address or free text. Use `$SYMBOL` for exact symbol matches; combine…

Security and deployer:
- `GET /v1/token/security`, `POST /v1/token/security` → `token` + `filterTokens` + `liquidityMetadataByToken` — Mint and freeze authority (`mintable`, `freezable`), `isScam`, `creato…
- `GET /v1/token/dev-tokens` → `filterTokens(filters: { creatorAddresses: [...] })` + `detailedWalletStats` — Every token a wallet deployed, with the same stats as any other token.…
- `GET /v1/audit/contract`, `GET /v1/audit/chains` → Not supported — Contract source auditing is Serialized's separate Audit product. See t…

Holders and traders:
- `GET /v1/token/holders` → `holders` + `filterTokenWallets` — `holders` for the ranked balance list with `count`; `filterTokenWallet…
- `GET /v1/token/top-traders` → `tokenTopTraders` — `tradingPeriod` is `DAY`, `WEEK`, `MONTH`, or `YEAR`. For `all`, or to…
- `GET /v1/pool/top-traders` → `filterTokenWallets` — Codex ranks traders per token, not per pool.

Pools and markets:
- `GET /v1/token/pools` → `listPairsWithMetadataForToken` — All venues for a token with liquidity, price, and volume. The `pool` o…
- `GET /v1/pool` → `pairMetadata` — Static definition: exchange, tokens, fee, creation. `pairId` is `"<pai…
- `GET /v1/pool/data`, `POST /v1/pools/data` → `getDetailedPairStats`, `getDetailedPairsStats` — Windowed volume, buys / sells, price change per pool. The plural form…
- `GET /v1/pool/ohlcv`, `POST /v1/pool/ohlcv` → `getBars` — Pair-scoped OHLCV; `symbol` is `"<pairAddress>:<networkId>"`, use `quo…
- `GET /v1/pool/trades`, `POST /v1/pools/trades` → `getTokenEvents` — Pass the pool address in `query: { address, networkId }`. Filter by `m…
- `GET /v1/meta/factories` → `filterLaunchpads` + `getExchanges` — Launchpads and DEX exchanges are separate catalogs on Codex.
- `GET /v1/meta/chains` → `getNetworks` — Returns the `networkId`s you use everywhere else. `getNetworkStatus` g…

Charts and OHLCV:
- `GET /v1/token/ohlcv`, `POST /v1/token/ohlcv` → `getTokenBars` — Token-level OHLCV from the token's top pair. `from` / `to` are unix se…
- `GET /v1/pool/ohlcv` → `getBars` — Pool-level OHLCV.

Trades:
- `GET /v1/token/trades` → `getTokenEvents` — Pass the token address and Codex reads its top pair; pass a pool addre…
- `POST /v1/token/trades` (makers filter) → `getTokenEvents(query: { maker: ... })` or `getTokenEventsForMaker` — Single maker on a token via the `maker` filter; one wallet across toke…
- `GET /v1/pool/trades`, `POST /v1/pools/trades` → `getTokenEvents` — Pool address in `query.address`. Batch pools with aliases.

Wallets:
- `GET /v1/wallet/positions`, `POST /v1/wallet/positions` → `balances` + `filterTokenWallets` — `balances` for holdings with USD value (`networks` is an array, so one…
- `GET /v1/wallet/closed-positions`, `POST /v1/wallet/closed-positions` → `filterTokenWallets` — Filter to rows where `tokenBalanceLive` is zero; `realizedProfitUsd1y`…
- `GET /v1/wallet/trades`, `POST /v1/wallet/trades` → `getTokenEventsForMaker` — Swap events for a wallet, newest first, with `cursor` pagination. Filt…
- `GET /v1/wallet/pnl`, `POST /v1/wallet/pnl` → `detailedWalletStats` + `walletChart` — `detailedWalletStats` returns realized PnL, wins / losses (win rate is…
- `GET /v1/wallet/equity/history` → `walletChart` — Net worth and PnL over time; resolutions `60`, `240`, `1D`, `7D`.
- `GET /v1/wallet/profile`, `POST /v1/wallet/profile` → `detailedWalletStats` (`wallet` object) — `displayName`, `avatarUrl`, `category`, `identityLabels`, socials (`tw…
- `GET /v1/wallet/funding` → `detailedWalletStats` (`wallet.firstFunding`) — First inbound transfer that funded the wallet. Coverage starts October…
- `GET /v1/wallet/transfers` → Not supported — Codex returns swap and token-lifecycle events, not arbitrary transfers…

Discovery, screening, and launchpads:
- `GET /v1/pulse` (`new`, `bonding`, `graduated`) → `filterTokens` on launchpad fields — One query per column with the filters above. Stream via `onFilterToken…
- `GET /v1/screener` (`trending`, `volume`, `marketCap`, `createdAt`) → `filterTokens`, `filterPairs` — Rank by `trendingScore*`, `volume*`, `marketCap`, or `createdAt` over…
- `GET /v1/search` → `filterTokens(phrase: ...)` — Search is an argument on the same endpoint.

Utility:
- `GET /v1/meta/chains` → `getNetworks` — Network catalog with `networkId`s.
- `GET /v1/usage`, `/v1/usage/history`, `/v1/usage/breakdown` → Codex usage in the dashboard — Request metering and per-endpoint breakdowns live in the dashboard.

Not covered by Codex: Contract source auditing** (`/v1/audit/contract`, `/v1/audit/chains`, and `dexPaid` on `/v1/token/security`).…; Deployer risk verdicts** (`/v1/token/dev-tokens` aggregate read). Codex lists every token a wallet created via…; Wallet transfers** (`/v1/wallet/transfers`). Codex returns swap and token-lifecycle events, not deposits, with…; Name resolution** (ENS, Basename, `.sol` on `/v1/wallet/profile`). Codex exposes a resolved `displayName` and…; Wash-trade flags** (`isWash` on trades). No Codex equivalent. Codex exposes `tradeSource` and maker labels ins…; Pool-level top traders and sparklines** (`/v1/pool/top-traders`, `/v1/pools/sparklines`). Codex ranks traders…
