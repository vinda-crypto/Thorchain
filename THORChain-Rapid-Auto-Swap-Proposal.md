# THORChain Improvement Proposal: Rapid Auto Swap with Fixed Protocol Fee & Arbitrage Discount

**Status:** Draft  
**Date:** April 2026

## Introduction

THORChain currently charges slip-based fees on both the user's swap and the arbitrageur's rebalancing trade. This double fee burden makes THORChain uncompetitive for large-volume, price-sensitive traders, especially compared to competitors that do not charge arbitrageurs. As a result, THORChain captures only 10-15% of the volume on recommended routes, despite being the only option for very large trades.

THORChain is unique. Even though its model is based on continuous liquidity, it is executed in blocks most of the time. Our minslipfee sets a floor on how much the pool moves in any given swap. This makes swaps inefficient, and when we and our competitors are competing at the basis points level, this matters. To refine THORChain's tool chest, this proposal introduces a new swap mode—Rapid Auto Swap—which improves competitiveness while preserving protocol revenue.

Another core economic problem is that THORChain double-counts volume because all swaps go through a RUNE pool. If the assets have external liquidity outside of THORChain, a $1 user swap will likely generate $2 in on-chain volume. However, THORChain does not discriminate how the volume is generated. This increases inefficiencies in the pools and between subswaps, costing swappers more than necessary. We treat volume from normal users and arbitrageurs essentially the same. Yet on THORChain—or in fact on any exchange—swappers and arbitrageurs are two sides of the same coin.

Arbitrageurs are highly fee-elastic. A reduction in their cost directly increases pool efficiency, reduces price drift, and ultimately attracts more user volume. This proposal also introduces RapidSwapLimitMinBps —a parameter that separates the fee structures for users and arbitrageurs, making THORChain competitive and providing greater efficiency to the protocol.

## Proposal

### Part 1: Rapid Auto Swap for End Users

When a user sets `streaming_quantity=0` (auto mode) and `rapid_swap=true`, the swap executes as follows:

1. **Subswap count determination**  
   Instead of using the normal `streaming_min_slip_bp` (e.g., 5 bp), the protocol uses a new Mimir variable:  
   **RapidAutoTargetSlipBp** (suggested initial value: 1 bp).  

   The existing streaming swap algorithm calculates the number of subswaps `k` needed so that each subswap’s raw slippage ≈ RapidAutoTargetSlipBp.  

   THORChain’s slippage is derived from the continuous liquidity pool (CLP) model:

   slip (fraction) = x / (x + X)

   where `x` is the sub-swap size and `X` is the pool depth in the input asset before the sub-swap.  

To achieve a target slippage of `T` bp (where `T` = RapidAutoTargetSlipBp or `streaming_min_slip_bp`), we solve for the maximum allowable sub-swap size `s`:  

s / (s + X) = T / 10000   →   s = (T * X) / (10000 - T)

Then the required number of sub-swaps `k` is:  

k = ceil(x / s) = ceil( (x * (10000 - T)) / (T * X) )

Simplified practical formula (widely used in current streaming logic):  

k ≈ ceil( (x * 10000) / (T * X) )

Where:  
- `x` = total input amount of the swap  
- `X` = current pool depth in the input asset  
- `T` = target slippage in basis points (RapidAutoTargetSlipBp = 1, or current `streaming_min_slip_bp` = 5)  

**Note:** VirtualRuneDepth calculation uses RapidAutoTargetSlipBp to replace `streaming_min_slip_bp`.

2. **No slip fee per subswap**  
Each sub-swap uses the pure raw XYK formula without any enforced minimum slip fee floor.  
The output amount for each sub-swap `i` is calculated as:

out_i = (x_i * Y_i) / (x_i + X_i)

where:  
- `x_i` = size of the i-th sub-swap (typically `x_i = x / k`),  
- `X_i` = current pool depth of the inbound asset before the sub-swap,  
- `Y_i` = current pool depth of the outbound asset before the sub-swap.

3. **Fixed protocol fee on total outbound**  
After all `k` sub-swaps are completed, the total raw outbound amount is calculated by summing the individual outputs:

total_raw_out = Σ out_i   (for i = 1 to k)

A fixed protocol fee is then applied using the new governance parameter **RapidAutoProtocolFeeBps** (suggested initial value: 12 bp):  

final_outbound = total_raw_out × (1 – Fee to Thorchain / 10000)

The deducted fee is sent to the THORChain Reserve as system income.

4. **Fallback**  
If `rapid_swap = false` or `streaming_quantity ≠ 0`, the swap reverts to the current slip‑based fee model (no change).

### Part 2: Arbitrage Discount for Trade Accounts

The proposal introduces **RapidSwapLimitMinBps**, a new Mimir parameter, set to 2 basis points (double of RapidAutoTargetSlipBp). For a qualifying swap, the fee is the greater of the actual slip and the new lower minimum (2 bp). This reduces the fee whenever the actual slip falls between the old minimum and the new minimum. A swap only qualifies for RapidSwapLimitMinBps if it meets all of the following conditions:

- It is a single‑block limit swap (no streaming).  
- It is executed from a Trade Account  
- The swap moves the pool price toward the Oracle price.  
`|P_pool' - P_oracle| < |P_pool - P_oracle|`

**Fallback**  
If the swap qualifies then `fee = max(actual_slip, RapidSwapLimitMinBps)`  
If the swap does NOT qualify then `fee = max(actual_slip, L1SlipMinBps)` (or `TradeAccountsSlipMinBps` if using Trade Accounts)  

If the oracle price is older than `MAX_ORACLE_AGE` (e.g., 10 blocks) or the price deviation exceeds a threshold, the swap does not qualify for the discount and uses the normal fee model.

## Example

### Current Model Calculation

| Row | Description                              | ETH.DAI          | ETH.USDT         | Total              | Equation / Note                  |
|-----|------------------------------------------|------------------|------------------|--------------------|----------------------------------|
| i   | Input in ETH.DAI                         | 2350             | -                | -                  | -                                |
| n   | Subswaps                                 | 18               | -                | -                  | -                                |
| m   | Multiplier                               | 1                | -                | -                  | -                                |
| -   | Rune price                               | 0.39 | 0.39 | -             | -                                |
| X   | Pool depth (X)                           | 130710           | 5960000          | -                  | -                                |
| Y   | Pool depth (Y)                           | 334110           | 2330000          | -                  | -                                |
| x   | Sub-swap size (i/n/m)                    | 130.55 | 333.04 | -                 | -                                |
| Y/X | Infinite external liquidity ratio        | 2.55 | 0.39 | -               | Y/X                              |
| a   | Reference Price (Valuey from eq 3)       | 333.71 | 130.20 | -                | valuey = xY/X                    |
| b   | Raw slippage (equation 2)                | 333.38 | 130.19 | -                 | output = xY/(x+X)                |
| c   | y (equation 6)                           | 333.04| 130.18 | -                  | y = (x Y X) / (x + X)^2          |
| s   | Raw slip in rune (a - b)                 | 0.332 | 0.00727 | -              | -                                |
| s/a | Slip (equation 4)                        | 0.000997 | 0.0000558 | 0.00105| slip = x/(x+X)          |
| f   | Liquidity fee in rune (b - c)            | 0.332 | 0.00727 | -                | x²Y/(x+X)²                       |
| -   | Liquidity fee in %                       | 0.0009968 | 0.00005587 | 0.00105 | -                      |
| n*m*f | Total Liquidity fee in USDT            | -                | -                | 2.47 | -                                |
| n*m*c | Quote receive by user in ETH.USDT      | -                | -                | 2343.37 | -                                |
| -   | No Slippage/fee in ETH.USDT              | -                | -                | 2348.32 | -                                |
| (f+s)/a | Total cost in %                        | 0.00199 | 0.000111 | 0.0021 | -                    |
| -   | Actual received in ETH.USDT              | -                | -                | 2327.050        | -                                |
| -   | Actual received (ratio)                  | -                | -                | 0.9909 | -                                |

- User swaps 2350 ETH.DAI → ETH.USDT.  
- ETH.DAI/RUNE pool depth: 130.71k/334.11k, ETH.USDT/RUNE pool depth: 2330k/5960k.  
- Pool price: 1USDT=1.0007133DAI  
- Quoted slip+Liquidity fee: 21.1bp  
- fee to Thorchain: 2.47USDT  
- Quoted estimated ETH.USDT received: 2343.38 USDT  
- Actual received: 2327.05 USDT  
- Total cost to user: 90.6bp

### Proposed Model 1 Calculation

| Row | Description                              | ETH.DAI          | ETH.USDT         | Total              | Equation / Note                  |
|-----|------------------------------------------|------------------|------------------|--------------------|----------------------------------|
| i   | Input in ETH.DAI                         | 2350             | -                | -                  | -                                |
| n   | Subswaps                                 | 18               | -                | -                  | -                                |
| m   | Multiplier                               | 5                | -                | -                  | -                                |
| e   | Fee to Thorchain                         | 0.0012           | -                | -                  | -                                |
| -   | Rune price                               | 0.391 | 0.390 | -             | -                                |
| X   | Pool depth (X)                           | 130710           | 5960000          | -                  | -                                |
| Y   | Pool depth (Y)                           | 334110           | 2330000          | -                  | -                                |
| x   | Sub-swap size (i/n/m)                    | 26.111 | 66.729| -                 | -                                |
| Y/X | Infinite external liquidity ratio        | 2.556 | 0.3909 | -               | Y/X                              |
| a   | Reference Price (Valuey from eq 3)       | 66.743 | 26.087 | -                | valuey = xY/X                    |
| b   | Raw slippage (equation 2)                | 66.729| 26.087 | -                 | output = xY/(x+X)                |
| s   | Raw slip in rune (a - b)                 | 0.0133 | 0.000292 | -             | -                                |
| s/a | Slip (equation 4)                        | 0.0001997| 0.000011196 | 0.000211 | slip = x/(x+X)     |
| -   | (intermediate)                           | 0.066651 | 0.00146| -                | -                                |
| (n*m*b*e)/rune price | Total Liquidity fee in USDT     | -                | -                | 2.817 | -                                |
| n*m*b*(1-e) | Quote receive by user in ETH.USDT   | -                | -                | 2345.01 | -                                |
| -   | No Slippage/fee in USDT                  | -                | -                | 2348.325 | -                                |
| -   | Total cost in %                          | 0.0001997 | 0.000011196 | 0.001411 | -                 |

- User swaps 2350 ETH.DAI → ETH.USDT.  
- ETH.DAI/RUNE pool depth: 130.71k/334.11k, ETH.USDT/RUNE pool depth: 2330k/5960k.  
- Pool price: 1USDT=1.0007133DAI  
- Quoted slip + Thorchain fees: 14.1bp (Lower slip)  
- fee to Thorchain: 2.817USDT (higher fees from User side)  
- Quoted estimated ETH.USDT received: 2345.01 USDT (wins more quote)  
- Actual received should be a lot higher and be closer to quoted amount.

## Assumptions

Following assumptions are made for this proposal.

1. Arbitrageurs are highly fee-elastic – A reduction in per‑swap fees for rapid limit swaps will significantly increase arbitrage activity, leading to tighter pool pricing and reduced price drift.  

2. Users are less fee-elastic than arbitrageurs – Users will tolerate a fixed protocol fee (e.g., 12 bp or higher) in exchange for better execution (lower raw slippage), especially for large trades.  

3. Volume growth from user side will offset lower per‑swap revenue from arb side – The increased competitiveness will attract enough additional volume that total system income remains stable or grows, even though the fee per dollar swapped may decrease.  

4. Oracle price is efficient and reliable to identify arbitrageurs’ orders for discount.  

5. Rapid swaps can consistently execute within the same block, reducing total execution time due to higher subswap counts.  

6. Arbitrageurs’ liquidity on the protocol can react quick enough for utilize this improvement.  

7. Protocol continues to improve to increase the number of rapid swaps. Ideally most swaps can be filled in one block at 2bp raw slippage.

### Key Performance Indicators (KPIs)

| Metric                                      | How to Measure                                                                 | Target                                                                 |
|---------------------------------------------|--------------------------------------------------------------------------------|------------------------------------------------------------------------|
| Market share on recommended routes          | Compare THORChain’s routed volume at partner’s level                           | Higher % share that is routed towards Thorchain                        |
| Average effective fee for large swaps       | Calculate (inbound - outbound) / inbound for swaps using Rapid Auto Swap mode. | Lower than current slip‑based model for same size.                     |
| Arbitrage activity volume                   | Volume of rapid limit swaps that execute within 1 block of a user subswap.     | Close to capacity of number of subswaps fit a block                    |
| Pool price deviation from external reference| Average absolute difference between pool price                                 | Reduce pool and oracle price difference to within 3-5bp                |
| Total swap volume                           | On‑chain volume                                                                | Significant increase over 3 months time after launch.                  |
| Protocol revenue                            | Sum of all fees collected (slip + outbound + new fixed fee).                   | Remain stable or grow despite lower per‑swap fees.                     |
| Protocol congestion                         | Block times increase significantly due to many subswaps.                       | Reduce RapidAutoTargetSlipBp or cap k lower via Mimir.                 |
| Arbitrage discount abuse                    | Rapid limit swap volume explodes but pool efficiency does not improve.         | Restrict discount further (e.g., only for Trade Accounts with minimum balance). |

## Conclusion

This proposal tackles the four core problems raised in the introduction:

1. Double fee burden – Both users and arbitrageurs currently pay slip fees, making THORChain uncompetitive.  

2. Low volume capture – THORChain captures only 10–15% of recommended route volume, despite being the only option for very large trades.  

3. Volume double counting – A $1 user swap generates $2 in on‑chain volume, inflating metrics and masking underlying inefficiency.  

4. No discrimination between users and arbitrageurs – The protocol treats two fundamentally different actors identically, leading to suboptimal fee design.

There is a clear trade‑off: per‑swap revenue may decrease for some trades. However, the goal is not to maximize revenue per swap—it is to maximize total system income by capturing higher volume. Lower fees attract more traders, and more volume compensates for lower per‑swap fees.  

There is also a secondary, network‑level effect. As THORChain grows in volume and fee revenue, it gains visibility and credibility. New projects and previously unconnected protocols will seek to integrate with THORChain, driving further demand for RUNE and creating a virtuous cycle of higher volume, deeper liquidity, and even more integrations.  

The community should monitor the KPIs outlined above and adjust RapidAutoProtocolFeeBps and RapidSwapLimitMinBps as needed via Mimir governance. With these changes, THORChain will be in a more competitive position to compete and adjust accordingly.
