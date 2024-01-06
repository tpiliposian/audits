## [M-01] Impact of Fixed Vega Parameter on Options Pricing

### Lines of code

https://github.com/code-423n4/2023-11-panoptic/blob/f75d07c345fd795f907385868c39bafcd6a56624/contracts/SemiFungiblePositionManager.sol#L130-L135
https://github.com/code-423n4/2023-11-panoptic/blob/f75d07c345fd795f907385868c39bafcd6a56624/contracts/SemiFungiblePositionManager.sol#L1250-L1329

### Vulnerability details

The `SemiFungiblePositionManager.sol` contract employs a fixed parameter, `VEGOID`, assigned a constant value of 2: `uint128 private constant VEGOID = 2` used in calculating premiums associated with liquidity utilization. This parameter acts as a multiplier, adjusting the sensitivity of premium calculations to changes in liquidity, akin to Vega in options.

In financial terms, Vega (or VEGOID in this context) measures the sensitivity of an option's price to changes in implied volatility. However, the fixed nature of `VEGOID` overlooks the dynamic nature of market conditions where volatility (liquidity sensitivity) can vary significantly.

Implied volatility (IV) is a key determinant in options pricing, representing the market's estimation of an asset's future volatility. A higher IV signifies increased uncertainty regarding an asset's price, usually leading to higher option premiums. Vega, in turn, measures an option's price sensitivity concerning a 1% change in implied volatility. It is pivotal for assessing an option's potential to gain value before expiration.

https://www.sciencedirect.com/topics/mathematics/implied-volatility#:~:text=Implied%20volatility%20is%20calculated%20by,price%20of%20the%20option%2C%20for

https://corporatefinanceinstitute.com/resources/derivatives/vega/

So, the fixed `VEGOID` parameter overlooks the dynamic nature of implied volatility, leading to significant repercussions in options pricing:

- Failure to adjust the `VEGOID` parameter for changing market conditions could result in mispriced premiums, potentially causing financial losses for users.
- Inflexibility in accounting for varying implied volatility levels may lead to risks associated with options.
- Market fluctuations may cause underestimation or overestimation of premiums, leading to potential mispricing of protocol or user assets.

 ### Proof of Concept

```solidity
  uint128 private constant VEGOID = 2;
  ```

  The function that updates the Owed and Gross account liquidities:

  ```solidity
      function _getPremiaDeltas(
        uint256 currentLiquidity,
        int256 collectedAmounts
    ) private pure returns (uint256 deltaPremiumOwed, uint256 deltaPremiumGross) {
        // extract liquidity values
        uint256 removedLiquidity = currentLiquidity.leftSlot();
        uint256 netLiquidity = currentLiquidity.rightSlot();

        // premia spread equations are graphed and documented here: https://www.desmos.com/calculator/mdeqob2m04
        // explains how we get from the premium per liquidity (calculated here) to the total premia collected and the multiplier
        // as well as how the value of VEGOID affects the premia
        // note that the "base" premium is just a common factor shared between the owed (long) and gross (short)
        // premia, and is only seperated to simplify the calculation
        // (the graphed equations include this factor without separating it)
        unchecked {
            uint256 totalLiquidity = netLiquidity + removedLiquidity;

            uint128 premium0X64_base;
            uint128 premium1X64_base;

            {
                uint128 collected0 = uint128(collectedAmounts.rightSlot());
                uint128 collected1 = uint128(collectedAmounts.leftSlot());

                // compute the base premium as collected * total / net^2 (from Eqn 3)
                premium0X64_base = Math
                    .mulDiv(collected0, totalLiquidity * 2 ** 64, netLiquidity ** 2)
                    .toUint128();
                premium1X64_base = Math
                    .mulDiv(collected1, totalLiquidity * 2 ** 64, netLiquidity ** 2)
                    .toUint128();
            }

            {
                uint128 premium0X64_owed;
                uint128 premium1X64_owed;
                {
                    // compute the owed premium (from Eqn 3)
                    uint256 numerator = netLiquidity + (removedLiquidity / 2 ** VEGOID);

                    premium0X64_owed = Math
                        .mulDiv(premium0X64_base, numerator, totalLiquidity)
                        .toUint128();
                    premium1X64_owed = Math
                        .mulDiv(premium1X64_base, numerator, totalLiquidity)
                        .toUint128();

                    deltaPremiumOwed = uint256(0).toRightSlot(premium0X64_owed).toLeftSlot(
                        premium1X64_owed
                    );
                }
            }

            {
                uint128 premium0X64_gross;
                uint128 premium1X64_gross;
                {
                    // compute the gross premium (from Eqn 4)
                    uint256 numerator = totalLiquidity ** 2 -
                        totalLiquidity *
                        removedLiquidity +
                        ((removedLiquidity ** 2) / 2 ** (VEGOID));
                    premium0X64_gross = Math
                        .mulDiv(premium0X64_base, numerator, totalLiquidity ** 2)
                        .toUint128();
                    premium1X64_gross = Math
                        .mulDiv(premium1X64_base, numerator, totalLiquidity ** 2)
                        .toUint128();
                    deltaPremiumGross = uint256(0).toRightSlot(premium0X64_gross).toLeftSlot(
                        premium1X64_gross
                    );
                }
            }
        }
    }
  ```

### Tools Used

Manual review.

### Recommended Mitigation Steps

Short term: Implement an adjustable `VEGOID` within defined limits, allowing authorized entities to update it based on observed market conditions.

Long Term: Develop algorithms to adaptively adjust VEGOID concerning observed liquidity changes or shifts in market volatility and establish transparent guidelines for VEGOID updates and educate stakeholders about its pivotal role in pricing accuracy, e.g here:

https://panoptic.xyz/research/streamia-vs-black-scholes

> **Note:** This issue was downgraded to QA: As long as the relationship between the premium multiplier and liquidity utilization holds as documented, but anyway, I decided to share it here.
