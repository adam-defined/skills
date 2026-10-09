# Token risk, safety signals and liquidity locks

Use when the user asks whether a token is safe, a scam, a rug or a honeypot, wants buy/sell tax, wants to screen risky tokens out of a list, or asks how much of a token's liquidity is locked. Docs: https://docs.codex.io/concepts/token-risk

## The primary signal: `risk`

Codex returns one assessment per token: a **verdict**, the **score** behind it, how much contract analysis it rests on (**coverage**) and the **reasons** that fired. It is a warning signal, not a safety guarantee.

| Where | Fields |
| --- | --- |
| `token`, `tokens`, `filterTokens` results (`token { risk { ... } }`) | `risk { verdict score coverage reasons flaggedAt analyzedAt }` |
| `filterPairs` results | `riskVerdict riskScore riskCoverage riskReasons riskFlaggedAt` (describe the pair's `quoteToken`, not the pool) |
| Filters on `filterTokens` and `filterPairs` | `riskVerdicts`, `riskCoverages`, `riskReasons`, `riskScore`, `riskFlaggedAt` |
| Rankings on both | `riskScore`, `riskFlaggedAt` |

Verdicts (`RiskVerdict`):

| Verdict | Meaning |
| --- | --- |
| `SCAM` | Proven by mechanical evidence (repeated honeypot simulation, 100% transfer fee) or labelled by a moderator |
| `HIGH_RISK` | Score 70 or more |
| `CAUTION` | Score 40 to 69 |
| `NEUTRAL` | Score below 40; reasons can still be present |
| `VERIFIED` | A moderator marked the token not-a-scam |
| `VERIFIED_CONTESTED` | Moderator says not-a-scam, mechanical evidence disagrees |

Coverage (`RiskAnalysisCoverage`): `ANALYZED` (contract simulation ran and was decisive), `INCONCLUSIVE` (ran, not decisive), `NOT_ANALYZED` (no contract analysis; verdict rests on holder, liquidity, trading and moderation signals only), `STALE` (an older analysis). Always show coverage next to the verdict: `NEUTRAL` + `NOT_ANALYZED` has had no honeypot or tax check.

Reasons (`RiskReasonCode`) are grouped by prefix: `PROOF_*` (honeypot simulation, non-transferable, 100% transfer fee, paused, default frozen, rug occurred), `AUTH_*` (mint, freeze, permanent delegate, transfer hook, pausable, owner not renounced, can transfer ownership), `TAX_*` (high buy/sell tax, unconfirmed honeypot, paused), `LIQ_*` (minimal, unknown, rug cliff, unlocked, draining), `HOLD_*` (top-10, dev, sniper, bundler, insider concentration), `FLOW_*` (one-way flow, wallet farm, wash dominated, bundled launch, mechanical trading), `REP_*` (serial rugger, risky creator, scammer funded, imitator symbol/name). New codes are added over time; handle unknown codes gracefully.

### Rules that matter

- A null `risk`, or a null field inside it, means Codex has no assessment. Treat it as unknown, never as safe. Filtering by `riskVerdicts` also drops unassessed tokens.
- `SCAM`, `VERIFIED` and `VERIFIED_CONTESTED` come from proof or a human, so their `score` is usually null. A high score alone never produces `SCAM`.
- The score is a capped heuristic (each reason adds points, plus 15 when three or more families fire together), not a probability. `AUTH_*` reasons report a capability without scoring it, so you cannot rebuild the score from the reasons list.
- `flaggedAt` is when the assessment last **changed**, not when it was last checked.
- `filterTokens` hides `isScam: true` tokens by default. To list `SCAM` verdicts, add `includeScams: true` inside `filters`. `isVerified: true` (keeps `isScam: false` tokens) is not the same as `riskVerdicts: [VERIFIED]`.
- Values inside one filter are ORed; different filters are ANDed.

### Templates

Read one token:

```graphql
query TokenRisk($input: TokenInput!) {
  token(input: $input) {
    symbol
    risk { verdict score coverage reasons flaggedAt }
  }
}
```

```json
{ "input": { "address": "0xC02aaA39b223FE8D0A0e5C4F27eAD9083C756Cc2", "networkId": 1 } }
```

Hide risky tokens from a list:

```graphql
query SaferTokens($filters: TokenFilters, $rankings: [TokenRanking]) {
  filterTokens(filters: $filters, rankings: $rankings, limit: 25) {
    results {
      liquidity
      volume24
      token { address networkId symbol risk { verdict score coverage } }
    }
  }
}
```

```json
{
  "filters": { "network": [8453], "riskVerdicts": ["NEUTRAL", "VERIFIED"], "riskCoverages": ["ANALYZED"], "liquidity": { "gte": 10000 } },
  "rankings": [{ "attribute": "volume24", "direction": "DESC" }]
}
```

Find likely honeypots among pairs:

```graphql
query HoneypotPairs($filters: PairFilters, $rankings: [PairRanking]) {
  filterPairs(filters: $filters, rankings: $rankings, limit: 25) {
    results {
      riskVerdict
      riskScore
      riskCoverage
      riskReasons
      pair { address networkId }
    }
  }
}
```

```json
{
  "filters": { "network": [8453], "riskReasons": ["PROOF_HONEYPOT_SIM", "TAX_HONEYPOT_UNCONFIRMED"] },
  "rankings": [{ "attribute": "riskScore", "direction": "DESC" }]
}
```

## The actual tax numbers: contract simulator (Growth or Enterprise, beta)

`risk` says a token has a high tax or failed a sell. The numbers come from the simulator, which runs a buy and a sell against one of the token's pools.

1. `simulateTokenContract(input: { simulateLiveContractInput: { contractAddress, networkId } })` returns `{ result simulationId error }` at once; the analysis runs in the background.
2. `getSimulateTokenContractResults(contractAddress, networkId, simulationId)` returns the rows; or subscribe to `onSimulateTokenContract(contractAddress, networkId)`.

```graphql
mutation AnalyzeToken($input: SimulateTokenContractInput!) {
  simulateTokenContract(input: $input) { result simulationId error }
}
```

```json
{ "input": { "simulateLiveContractInput": { "contractAddress": "0x89d8cb38067b55f820f29a9e12d0ce18682a2bfc", "networkId": 8453 } } }
```

```graphql
query TokenTaxes($contractAddress: String!, $networkId: Int!, $simulationId: String) {
  getSimulateTokenContractResults(contractAddress: $contractAddress, networkId: $networkId, simulationId: $simulationId) {
    results {
      status
      verdict
      verdictReason
      swap { buyTax sellTax buySuccess sellSuccess }
      liquidity { pairAddress }
    }
  }
}
```

```json
{ "contractAddress": "0x89d8cb38067b55f820f29a9e12d0ce18682a2bfc", "networkId": 8453, "simulationId": "<simulationId from AnalyzeToken>" }
```

Reading a result: `verdict` is the answer (`TRADEABLE`, `HONEYPOT`, or `INDETERMINATE` with a `verdictReason`); `status` only tracks the pipeline and `buySuccess` / `sellSuccess` are true on indeterminate rows too, so read `verdict` first. `buyTax` / `sellTax` are decimal-fraction strings (`"0.05"` is 5%, `"1"` is 100%; a honeypot usually shows `sellTax: "1"` with `sellSuccess: false`). Tax belongs to the pool in `liquidity.pairAddress`, so two runs on the same token can differ (a Uniswap V4 pool id is 32 bytes). Pass the `simulationId` returned by the mutation: the submission writes a `PENDING` row first and the finished row lands under the same `uuid` within seconds, so poll until that row has a `verdict`. Without `simulationId` you get every stored analysis, newest first, and an older finished row can be mistaken for the new result; an empty list means never analyzed. One submission per token and network every five minutes, otherwise `TOO_MANY_REQUESTS`.

## Supporting signals (keep, but read `risk` first)

- `isScam`, `isVerified` (moderator decisions), `potentialScam` and `potentialScamReasons` (`MinimumLiquidity`, `LiquidityRugPull`, `LiquidityUnknown`, `SuspiciousWalletActivity`, `AbnormalBuyerRatio`, `DevConcentration`, `SniperConcentration`, `BundledLaunch`, `ForeignVenueLaunch`, `SellerHub`) on `filterTokens` results. `LiquidityUnknown` means the liquidity checks could not run.
- Authority: `mintable`, `freezable` on `token`; on Robinhood (4663) mint authority is not resolved and reads null.
- Holder cohorts on `filterTokens`: `top10HoldersPercent`, `devHeldPercentage`, `sniperHeldPercentage`, `bundlerHeldPercentage`, `insiderHeldPercentage`, `suspiciousHeldPercentage` (deduplicated union of snipers, bundlers and insiders; one-check screen) and their `*Count` twins. They are null for non-launchpad tokens and 0 for a launchpad token with no labelled holders. `tokenWalletStats(input: { tokenAddress, networkId })` returns the addresses behind each cohort (capped at 200) plus `devAddress`.
- Market cap sentinel: since September 2026 a market cap that overflowed or exceeded $5T reads 0 (older data could show 9223372036854775807). `circulatingMarketCap` can exceed `marketCap` when a third-party circulating supply is stale after burns; prefer `marketCap` then.
- Creator: `token { creator { address category identityLabels tokensCreatedCount tokensMigratedCount } }` (`category` `TOKEN_CREATOR`, `NOTORIOUS`, etc.).

## Liquidity locks: pick the endpoint by question

| Question | Operation | Notes |
| --- | --- | --- |
| How much of this token's liquidity is locked? | `liquidityMetadataByToken(tokenAddress, networkId)` | `lockedLiquidityPercentage` is 0 to 1; `totalLiquidityUsd`, `lockedLiquidityUsd`, `lockBreakdown` by protocol |
| How much of this pair's liquidity is locked? | `liquidityMetadata(pairAddress, networkId)` | `lockedLiquidity { active inactive lockBreakdown }`, `liquidity { active inactive }` |
| Who holds the locks, when do they vest? | `liquidityLocksV2` | Per-pair list: `lockedPercent` is 0 to 100, `locked = permanentLocked + vestedLocked`, holders with `entityId`, `displayName`, `lockProtocol`, `unlockAt`. No USD. Do not sum across pools or use as the headline |
| Legacy | `liquidityLocks` | Deprecated, event-derived, missed older locks. Do not use |

Lock data is read from on-chain state across 16 networks (burned LP, locker vaults such as UNCX, PinkSale, Team Finance, Raydium lock; Uniswap V4 launch hooks on Robinhood and Base; Meteora DAMM v2, Orca and Raydium CLMM position locks). When a lock cannot be proven, none is reported: 0% means "not verified", not "free to withdraw". A bonding-curve token has no pool before graduation, so it correctly shows no lock. `lockedLiquidityPercentage` is also a filter on `filterPairs` and a field on `PairFilterResult`.

```graphql
query TokenLockShare($tokenAddress: String!, $networkId: Int!) {
  liquidityMetadataByToken(tokenAddress: $tokenAddress, networkId: $networkId) {
    totalLiquidityUsd
    lockedLiquidityUsd
    lockedLiquidityPercentage
    lockBreakdown { lockProtocol amountLockedUsd }
  }
}
```

```json
{ "tokenAddress": "0x6982508145454ce325ddbe47a25d4ec3d2311933", "networkId": 1 }
```
