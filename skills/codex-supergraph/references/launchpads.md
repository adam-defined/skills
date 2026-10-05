# Launchpads

Use when the user asks about pump.fun or any launchpad, new token launches, bonding curves, graduations and migrations, or wants to rank launchpads themselves. Docs: https://docs.codex.io/launchpads

## Which operation

| Need | Operation | Notes |
| --- | --- | --- |
| Stream new launchpad tokens | `onLaunchpadTokenEventBatch(input)` (preferred) or `onLaunchpadTokenEvent(input)` | Payload is a token snapshot (market cap, holders, buy/sell counts per hour, sniper/bundler/insider/dev held percentages, fees), not a transaction. See template #22. With no input you get the firehose for every launchpad; filter by `protocols`, `launchpadNames`, `networkId`, `eventType` |
| Screen launchpad tokens | `filterTokens` with `launchpadProtocol`, `launchpadName`, `launchpadCompleted`, `launchpadMigrated`, `launchpadGraduationPercent`, `launchpadCompletedAt`, `launchpadMigratedAt` filters | Same 100+ filters and rankings as any token screen |
| Rank launchpads | `filterLaunchpads(filters, launchpads, networks, scope, rankings, limit, offset)` | Beta. Funnel counts (`tokensCreated*`, `tokensCompleted*`, `tokensMigrated*`), `migrationRate`, `avgGraduationPercent`, volume, fees and active tokens over 1h / 4h / 12h / 24h / 1w. `scope: network` (per network) or `global` |
| Token launch alerts server-side | `createWebhooks(input: { tokenLaunchEventWebhooksInput: { webhooks: [...] } })` | `TOKEN_LAUNCH_EVENT`: exactly one of `creatorAddress` or `launchpadName` in the condition, optional `tokenAddress` and `networkIds` |
| Who sniped / bundled / insider-bought | `tokenWalletStats(input: { tokenAddress, networkId })` | Addresses capped at 200 per cohort |

## Lifecycle

`eventType` (`LaunchpadTokenEventType`): `Deployed` (earliest signal; name, symbol and image may not be populated yet), `Created` (metadata available), `Updated` (during trading), `Completed` (bonding curve full), `Migrated` (liquidity moved to a DEX), plus `Unconfirmed*` variants. `Completed` is a legacy state; monitor graduations with `Migrated`. Protocols without a bonding curve (Zora, Clanker, Baseapp) go straight to `Created` and never complete or migrate; Doppler-style protocols manage liquidity along ticks and leave `graduationPercent`, `completed` and `completedAt` null.

## Names and protocols

- `launchpadProtocol` is the enum `LaunchpadTokenProtocol` (`Pump`, `PumpMayhem`, `FourMeme`, `RaydiumLaunchpad`, `BoopFun`, `MeteoraDBC`, `Virtuals`, `Clanker`, `ClankerV4`, `Baseapp`, `ZoraV4`, `Printr`, `NadFun`, `Doppler`, `Flaunch`, ...). `launchpadName` is a display string that attributes a token to a specific surface sharing a protocol (`"Pump.fun"`, `"Flap"`, `"Four.meme"`, `"pons"`, `"LaunchLab"`, `"MeteoraDBC"`, `"Virtuals"`, `"Bankr"`, `"BAGS"`, ...). Names are case-sensitive; take them from https://docs.codex.io/launchpads (the Launchpad Names table) rather than guessing.
- Coverage as of October 2026: 60+ launchpads across Solana, Base, Robinhood, Arc, BNB, Ethereum, Monad, Arbitrum, Avalanche, Unichain, X Layer, Mantle, MegaETH and Polygon. New launchpads are added weekly; the changelog lists them.
- `UniswapCCA` labels tokens launched through Uniswap's shared launch contract, which carries no on-chain attribution to a specific launchpad. On Robinhood it is usually pools.trade, but anyone can use the same contract.
- `launchpad: null` on a token that visibly came from a launchpad means the launchpad is not indexed or labelled on that network (for example some BAGS tokens on Robinhood); it is not a verdict on the token.

## Liquidity on bonding curves

A curve's `liquidity` is the quote token it actually holds (SOL, WBNB, USDT, USDC), not the value of its unsold token inventory at the curve's own price (changed September 2026). Expect small numbers on pre-graduation curves; a curve holding under $100 of quote reports no liquidity. Curves that were idle before the change keep their old figure until their next trade.

## Template: rank launchpads by 24h migrations

```graphql
query TopLaunchpads($filters: LaunchpadFilters, $rankings: [LaunchpadRanking!]) {
  filterLaunchpads(filters: $filters, scope: global, rankings: $rankings, limit: 10) {
    count
    results {
      launchpadName
      launchpadProtocol
      networkIds
      tokensCreated24
      tokensCompleted24
      tokensMigrated24
      migrationRate24
      volume24
      activeTokens24
    }
  }
}
```

```json
{ "filters": { "tokensCreated24": { "gte": 10 } }, "rankings": [{ "attribute": "tokensMigrated24", "direction": "DESC" }] }
```

## Template: tokens about to graduate on pump.fun

```graphql
query NearGraduation($filters: TokenFilters, $rankings: [TokenRanking]) {
  filterTokens(filters: $filters, rankings: $rankings, limit: 25) {
    results {
      liquidity
      marketCap
      volume1
      holders
      token {
        address
        symbol
        launchpad { launchpadName graduationPercent completed migrated }
        risk { verdict coverage }
      }
    }
  }
}
```

```json
{
  "filters": { "network": [1399811149], "launchpadName": ["Pump.fun"], "launchpadCompleted": false, "launchpadGraduationPercent": { "gte": 80 }, "volume1": { "gte": 1000 } },
  "rankings": [{ "attribute": "graduationPercent", "direction": "DESC" }]
}
```
