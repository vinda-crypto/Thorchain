# THORChain Improvement Proposal (TIP): Rapid Auto Swap with Fixed Protocol Fee & Arbitrage Discount

**Status:** Draft / Community Review  
**Author:** [Your Name / Community]  
**Date:** April 2026  
**Version:** 2.0 (Revised based on technical audit)

## Introduction

THORChain currently applies slip-based fees on both user swaps and subsequent arbitrageur rebalancing trades. This **double-fee burden** makes THORChain less competitive for large-volume, price-sensitive traders compared to centralized or other decentralized venues that do not penalize arbitrageurs.

Despite being the only viable option for very large trades, THORChain captures only **10-15%** of volume on recommended routes. The protocol’s minimum slip fee floors (e.g., `StreamingSwapMinBPFee`) further reduce efficiency when competing at the basis-point level.

Additional issues include:
- Volume double-counting through the RUNE pool (a $1 user swap often generates ~$2 on-chain volume).
- No economic discrimination between price-insensitive users and highly fee-elastic arbitrageurs.

This proposal introduces **Rapid Auto Swap** for end users and an **Arbitrage Discount** for Trade Accounts to improve competitiveness while aiming to maintain or grow total protocol revenue.

## Proposal

### Part 1: Rapid Auto Swap for End Users

When a user sets `streaming_quantity=0` (auto mode) **and** `rapid_yes=true` in the memo, the swap follows the new path:

1. **Subswap Count Determination**
   - Use new Mimir: `RapidAutoTargetSlipBp` (suggested initial: **2 bp**; previous version suggested 1 bp).
   - Calculate number of sub-swaps `k` so each sub-swap’s **raw slippage** ≈ target.
   - Formula (exact):

   s = (T * X) / (10000 - T)
   k = ceil(x / s) = ceil( x * (10000 - T) / (T * X) )

   - Practical approximation (used in current streaming logic):

   k ≈ ceil( (x * 10000) / (T * X) )

   - `VirtualRuneDepth` calculation updated to use `RapidAutoTargetSlipBp`.

2. **No Per-Subswap Slip Fee**
   - Each sub-swap uses the **pure raw XYK** formula:

out_i = (x_i * Y_i) / (x_i + X_i)

   - No additional explicit liquidity fee floor per sub-swap.

3. **Fixed Protocol Fee on Total Outbound**
   - After all sub-swaps:
   
   total_raw_out = Σ out_i
   final_outbound = total_raw_out × (1 - RapidAutoProtocolFeeBps / 10000)
   
- New Mimir: `RapidAutoProtocolFeeBps` (suggested initial: **12 bp**).
- Deducted fee sent to **THORChain Reserve** as system income.

4. **Fallback**
- If `rapid_yes=false` or `streaming_quantity ≠ 0` → revert to current slip-based fee model.

### Part 2: Arbitrage Discount for Trade Accounts

New Mimir: `RapidSwapLimitMinBps` (suggested: **2 bp** — double the user target for conservatism).

A swap qualifies for the discount only if **all** conditions are met:
- Single-block **limit swap** (no streaming)
- Executed from a **Trade Account**
- Moves pool price **toward** Oracle price: `|P_pool' - P_oracle| < |P_pool - P_oracle|`
- Oracle price is fresh (`age < MAX_ORACLE_AGE`, e.g., 10 blocks) and deviation within threshold

**Fee logic:**
- If qualifies: `fee = max(actual_slip, RapidSwapLimitMinBps)`
- Else: `fee = max(actual_slip, L1SlipMinBps)` (or `TradeAccountsSlipMinBps`)

This reduces fees for qualifying arbitrage when actual slip falls between old and new minimum.

## Parameter Summary

| Parameter                    | Type          | Suggested Initial Value | Description |
|-----------------------------|---------------|--------------------------|-----------|
| RapidAutoTargetSlipBp       | Mimir (bp)   | 2                        | Target raw slippage per sub-swap for Rapid Auto mode |
| RapidAutoProtocolFeeBps     | Mimir (bp)   | 12                       | Fixed protocol fee deducted from total outbound (to Reserve) |
| RapidSwapLimitMinBps        | Mimir (bp)   | 2                        | Minimum fee for qualifying Trade Account limit swaps |
| StreamingSwapMinBPFee       | Existing     | 5 (unchanged)            | Current minimum for normal streaming |
| L1SlipMinBps                | Existing     | (current value)          | Normal L1 minimum slip fee |
| TradeAccountsSlipMinBps     | Existing     | (current value)          | Trade Account fallback minimum |

**Note:** All new parameters are fully governable via Mimir voting.

## Execution Model Clarifications & Trade-offs

- **Same-block vs. Spaced Subswaps**: If executed fully in one block (possible via Advanced Swap Queue + rapid iteration), chained raw outputs equal a single large XYK swap. Slippage reduction benefit appears primarily when sub-swaps are spaced (allowing arb rebalancing). The proposal benefits most from **configurable intervals** (including 0 for true rapid).
- **Revenue Shift**: Explicit liquidity fees move partially from LPs → Reserve. Volume growth must offset this for LPs.
- **Multi-hop Routes**: Fixed fee applied once on final total outbound (clarify in implementation to avoid over-charging).

## Example Comparison

**Current Model** (2350 ETH.DAI → ETH.USDT example):

| Metric                        | Value          |
|-------------------------------|----------------|
| Pool depths (approx)          | DAI/RUNE: 130.71k/334.11k<br>USDT/RUNE: 2330k/5960k |
| Pool price                    | 1 USDT ≈ 1.0007133 DAI |
| Quoted slip + liquidity fee   | 21.1 bp        |
| Fee to THORChain              | 2.47 USDT      |
| Quoted estimated received     | 2343.38 USDT   |
| Actual received               | 2327.05 USDT   |
| Total effective cost to user  | 90.6 bp        |

**Proposed Rapid Auto Model**:

| Metric                        | Value          | Change |
|-------------------------------|----------------|--------|
| Quoted slip + THORChain fees  | 14.1 bp        | Lower raw impact |
| Fee to THORChain              | 2.817 USDT     | Higher from user side |
| Quoted estimated received     | 2345.01 USDT   | Improved quote |
| Actual received               | Expected significantly closer to quote | Better execution |

**Key Insight**: Users gain better execution (lower effective slippage) in exchange for a predictable fixed protocol fee. Arbitrageurs gain lower minimum fees, tightening pools.

## Assumptions

1. Arbitrageurs are highly fee-elastic.
2. Users are less fee-elastic and will accept a fixed fee for superior execution on large trades.
3. Net volume growth offsets any per-swap revenue reduction.
4. Oracle price remains reliable for arb classification.
5. Rapid execution (including same-block where possible) reduces overall latency.
6. Protocol improvements continue to increase single-block fill rate at low slippage.

## Key Performance Indicators (KPIs)

Monitor post-deployment via dashboards and adjust parameters via Mimir:

| KPI Category               | Specific Metrics |
|----------------------------|------------------|
| **Volume Capture**        | % of recommended route volume captured<br>Total swap volume (USD)<br>User vs. arb volume split |
| **Execution Quality**     | Average effective slippage (bp)<br>Quote-to-actual deviation<br>Average sub-swap count for Rapid mode |
| **Revenue & Yield**       | Total system income (Reserve + LP fees)<br>LP yield delta (before/after)<br>Protocol revenue per $1M volume |
| **Pool Efficiency**       | Average price drift from oracle<br>Arb frequency & success rate<br>Pool depth utilization |
| **Network Health**        | Rapid mode adoption rate<br>Block gas impact<br>Failed rapid swaps rate |

**Review Period**: Minimum 4–6 weeks of live data before any parameter changes.

## Potential Risks & Mitigations

- **LP Revenue Shift** — Start with conservative `RapidAutoProtocolFeeBps` (e.g., 15–20 bp) and monitor yields.
- **Same-Block Overload** — Enforce max `k` limit and leverage Advanced Swap Queue rapid iteration.
- **Oracle Manipulation** — Add age + deviation thresholds; fallback to normal fees on stale data.
- **Gaming** — Monitor Trade Account behavior; adjust `RapidSwapLimitMinBps` if needed.
- **Edge Cases** — Tiny swaps (fixed fee may hurt), extremely large swaps (gas limits), multi-hop fee application.

## Conclusion

This proposal addresses four core issues:
1. Double fee burden on users + arbs
2. Low volume capture on recommended routes
3. Volume double-counting and inefficiency
4. Lack of fee discrimination between users and arbitrageurs

The design trades per-swap revenue for higher total volume and better competitiveness. A virtuous cycle of increased volume → deeper liquidity → more integrations → higher RUNE demand is expected.

Community governance should monitor the defined KPIs and tune parameters (`RapidAutoProtocolFeeBps`, `RapidSwapLimitMinBps`, etc.) as data emerges.

**Next Steps**:
- Community discussion and simulations
- Testnet deployment with real arbitrage bots
- Formal code review and implementation (new swap type branch, quote endpoint updates, memo flag support)

Feedback and improvements welcome.

---

*This proposal builds on existing THORChain primitives (Streaming Swaps, Trade Accounts, Advanced Swap Queue, Bifrost Oracle) and aims for minimal disruption while delivering measurable competitiveness gains.*
   
