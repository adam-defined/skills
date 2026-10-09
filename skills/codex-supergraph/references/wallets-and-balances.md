# Wallets, PnL, balances and holders

Use when the user asks about a wallet's trades, profit and loss, portfolio or balances, smart-money or top-trader lists, holder lists, or wallet labels.

## Plan gating comes first

`filterWallets`, `filterTokenWallets`, `tokenTopTraders`, `detailedWalletStats`, `walletChart`, `holders`, `balances`, `refreshBalances` and webhooks need a **Growth or Enterprise** plan. On the entry plan they return empty results or an "upgrade your plan" error; that is not a data bug. `getTokenEventsForMaker` works on every plan, but the entry plan's 10,000 requests a month will not sustain polling.

## Which operation

| Need | Operation | Notes |
| --- | --- | --- |
| Rank wallets across the network | `filterWallets(input: FilterWalletsInput!)` | Filters and rankings over 1d / 1w / 30d / 1y windows: `volumeUsd*`, `realizedProfitUsd*`, `realizedProfitUsdExNative*` (strips native-token exposure), `winRate*`, `swaps*`, `uniqueTokens*`, `avgHoldPeriodSec*`, plus `scammerScore`, `botScore`, `ethosScore`, `tokensCreatedCount`, `tokensMigratedCount`, `categories: [WalletCategory!]`, `networkId`. `includeLabels` / `excludeLabels` (behavioral labels), `includeTradeSourceIds` / `excludeTradeSourceIds` (apps traded through, e.g. `axiom`), `phrase` |
| Top wallets for one token | `filterTokenWallets(input: { tokenIds: ["address:networkId"] })` | Windowed PnL per wallet per token (`realizedProfitUsd1d`, `amountBoughtUsd1d`, ...). `tokenIds` is current, `tokenId` deprecated. `count` is rows on the page, not a total |
| Top traders for one token, simple | `tokenTopTraders(input: { tokenAddress, networkId, tradingPeriod })` | `tradingPeriod` DAY / WEEK / MONTH / YEAR; `ranking { attribute: realizedProfitUsd \| volumeUsd }`; descending returns only positive realized profit, ascending only negative (biggest losers first) |
| One wallet's profile | `detailedWalletStats(input: { walletAddress, networkId? })` | Fixed windows `statsDay1`, `statsWeek1`, `statsDay30`, `statsYear` (type `WindowedWalletStatsYear`; `statsYear1` is deprecated and returns 30-day unique tokens). `statsUsd` holds money values, `statsNonCurrency` counts. `networkBreakdown` per network. `wallet { category identityLabels displayName firstFunding ... }` |
| One wallet's history series | `walletChart(input: { walletAddress, networkId, range: { start, end }, resolution })` | Arbitrary date range, `data { timestamp volumeUsd realizedProfitUsd swaps }` |
| Wallet's trades | `getTokenEventsForMaker(query: { maker, networkId, tokenAddress? })` | Live twin `onEventsCreatedByMaker(input: { makerAddress })` |
| Portfolio | `balances(input: BalancesInput!)` | See below |
| Holders of a token | `holders(input: { tokenId, limit, cursor, sort, filterContracts })` | Items in `items`, plus `count`, `top10HoldersPercent`, `status`. `filterContracts: true` drops pools, lockers and other contract wallets. Live twin `onHoldersUpdated(tokenId)` |
| Who are the snipers / bundlers / insiders | `tokenWalletStats(input: { tokenAddress, networkId })` | Addresses (cap 200), counts, held percentages, `devAddress` |
| Label vocabulary | `walletLabelTypes` | Returns the `WalletLabelType` list |

## Balances

```graphql
query Portfolio($input: BalancesInput!) {
  balances(input: $input) {
    cursor
    items {
      tokenId
      networkId
      shiftedBalance
      uiBalance
      balanceUsd
      tokenPriceUsd
      liquidityUsd
      tokenLastTradedTimestamp
      updatedAtBlock
      token { symbol name }
    }
  }
}
```

```json
{ "input": { "walletAddress": "0xd8dA6BF26964aF9D7eEd9e03E53415D37aA96045", "networks": [1, 8453], "includeNative": true, "removeScams": true, "sortBy": "USD_VALUE", "limit": 50 } }
```

- `BalancesInput`: `walletAddress`, `networks`, `tokens` (specific `address:networkId` ids), `includeNative`, `removeScams`, `sortBy` (`BALANCE` | `USD_VALUE`), `sortDirection`, `uiAmountMode` (`SCALED` default | `RAW`), `limit`, `cursor`.
- Portfolio value: sum `balanceUsd`. Gate on `liquidityUsd` and `tokenLastTradedTimestamp` so illiquid or dead tokens do not inflate the total. `USD_VALUE` ordering is best effort across pages.
- `uiBalance` is display-only. For Solana Token-2022 scaled-UI mints and Base B20 tokens it is scaled by the token's multiplier while `tokenPriceUsd` is per raw token, so never multiply `uiBalance` by `tokenPriceUsd`; use `balanceUsd`.
- `removeScams: true` removes only tokens labelled as scams. Unlabelled dust stays; filter client-side if you want aggressive cleanup.
- Native balances need a traced EVM network; `includeNative: true` returns no native row on untraced chains (the docs page lists them). Networks with a native ERC-20 alias return one holding with `assetId`, `balanceDecimals`, `isBalanceAlias`.
- `refreshBalances(input: [{ walletId, tokenId }])` re-reads specific balances on demand (`native:<networkId>` for native tokens). Live twin `onBalanceUpdated(walletAddress)`.

## Reading wallet quality

- Behavioral `labels` (strings on filter results) and `scammerScore` / `botScore` for trust; `realizedProfitUsd*`, `winRate*`, `wins` / `losses` for performance; `avgHoldPeriodSec*` for style (null when under $1 of cost basis was sold in the window).
- `identityLabels` (`WalletLabel`: `CEX`, `DEX`, `BRIDGE`, `KOL`, `VC`, `FOUNDER`, `WHALE`, `INSIDER`, `MIXER`, `SANCTIONED`, `BLACKLISTED`, `HACKER`, `RUG_PULLER`, `MEV_BOT`, plus behavioral `BOT`, `SCAMMER`, `VOLUME_BOT`) are populated only where a wallet has been attributed. `BOT` and `WHALE` are common; `CEX` is null on most exchange wallets. For exchange attribution use `wallet.category == EXCHANGE` and `wallet.displayName` (for example "Binance 14" on Ethereum; Solana exchange wallets are categorized but mostly unnamed).
- `Wallet.category` (`WalletCategory`): `NORMIE`, `TOKEN_CREATOR`, `EXCHANGE`, `DEFI_EXCHANGE`, `PAIR`, `PAIR_TOKEN_HOLDER`, `POOL_AUTHORITY`, `STAKING_VAULT`, `NOTORIOUS`, `BURN`, `LOCKER`.
- `Wallet.firstFunding` is the inbound transfer in the wallet's first indexed transaction, not a full-history lookup. The index starts mid-October 2025 with no backfill (older wallets return null). Coverage is ~99% on EVM and only 20 to 60% on Solana (plain SOL transfers, CPI-funded and airdrop-first wallets are missed). There is no filter on funding.
- `tradeSourceIds` on wallets and `tradeSource` on events mean "has traded through this app", never "is owned by"; populated on Solana and EVM, null when there is no signal.
- Polymarket traders: `predictionTraders(input: { traderIds: ["<address>:Polymarket"] })` or `wallet { polymarket { ... } }`; the id is the Polymarket proxy wallet, not the signing EOA. Hyperliquid perps are not indexed (HyperEVM DEX swaps are).

## PnL definitions and known distortions

- `tokenAcquisitionCostUsd` is the average-cost basis of `purchasedTokenBalance` (bought minus sold through tracked swaps). `realizedProfitUsd` is proceeds minus the average cost of the units sold. Bought USD minus sold USD equals acquisition cost minus realized profit. Transfers out (router fee skims) do not reduce `purchasedTokenBalance`.
- Records key on `maker`, the transaction signer. Buys routed through a custodial or router wallet attribute to that wallet, not the beneficiary.
- Pools with a static fee above 25% are dropped from pricing, so trades through them carry no USD and PnL is overstated. On Uniswap V4 hook pools the swap event reports pre-fee amounts, so bought quantity and sold USD can be overstated by the hook's take (4 to 8% seen on Robinhood).
- Low-liquidity launchpad tokens before mid-August 2026 may have no `filterTokenWallets` rows at all; those rows are not backfilled.

## Template: rank wallets

```graphql
query SmartMoney($input: FilterWalletsInput!) {
  filterWallets(input: $input) {
    count
    results {
      address
      networkId
      labels
      identityLabels
      realizedProfitUsd30d
      realizedProfitUsdExNative30d
      winRate30d
      swaps30d
      uniqueTokens30d
      avgHoldPeriodSec30d
      scammerScore
      botScore
    }
  }
}
```

```json
{
  "input": {
    "filters": { "networkId": 1399811149, "volumeUsd30d": { "gte": 100000 }, "swaps30d": { "gte": 50 }, "winRate30d": { "gte": 0.6 }, "botScore": { "lte": 30 } },
    "excludeLabels": ["BOT", "SCAMMER"],
    "rankings": [{ "attribute": "realizedProfitUsd30d", "direction": "DESC" }],
    "limit": 25
  }
}
```

## Template: top traders for a token

```graphql
query TopTraders($input: TokenTopTradersInput!) {
  tokenTopTraders(input: $input) {
    items {
      walletAddress
      realizedProfitUsd
      realizedProfitPercentage
      volumeUsd
      buys
      sells
      tokenBalance
      labels
      wallet { category displayName }
    }
  }
}
```

```json
{ "input": { "tokenAddress": "0x6982508145454ce325ddbe47a25d4ec3d2311933", "networkId": 1, "tradingPeriod": "WEEK", "ranking": { "attribute": "realizedProfitUsd", "direction": "DESC" }, "limit": 20 } }
```
