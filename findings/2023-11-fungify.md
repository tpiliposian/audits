## [M-01] Missing slippage protection in the LiquidateBorrow function

### Description

`Comptroller.sol` lines 294-379

The `batchLiquidateBorrow` function in `Comptroller.sol` calculates the amount of funds that will be transferred in a liquidation. This computation relies on asset prices retrieved from an oracle through the `oracle_.getUnderlyingPrice` call. However, it's important to highlight the absence of slippage protection in this process.

There's no guarantee that the amount returned by the `batchLiquidateBorrow` function corresponds to the current market price. This is because the transaction that updates the price feed might be mined before the actual call to batchLiquidateBorrow. As a result, users may experience liquidation amounts that differ from their expectations.

Consider a scenario where a user initiates a `batchLiquidateBorrow` function. Due to an oracle malfunction or significant price fluctuations, the amount of collateral transferred from the module might be much lower than the user would have anticipated based on the current market conditions.

It is recommended to introduce a parameter that would empower the caller to set a threshold for the minimum acceptable value for the liquidation. By doing so, users can explicitly state their assumptions about the liquidation and ensure that the collateral payout remains as profitable as expected.

### Code Snippet

```solidity
function batchLiquidateBorrow(address borrower, Liquidatables[] memory liquidatables, CTokenInterface[] memory cTokenCollaterals) external nonReentrant returns (uint[][2] memory results) {
        require(adminWhitelist[msg.sender], "unauthorized");
        require(!seizeGuardianPaused, "seize is paused");

        (uint err, uint beforeRatio) = getAccountDebtRatio(borrower);
        require(err == uint(Error.NO_ERROR) && beforeRatio != 0, "INSUFFICIENT_SHORTFALL");

        results[0] = new uint[](liquidatables.length);
        results[1] = new uint[](cTokenCollaterals.length);

        PriceOracle oracle_ = oracle; 

        uint liquidatedValueTotal;
        {
            for (uint i = 0; i < liquidatables.length;) {

                require(markets[liquidatables[i].cToken].isListed, "market not listed");

                if (liquidatables[i].amount != 0) {
                    results[0][i] = CErc20Interface(liquidatables[i].cToken)._liquidateBorrow(msg.sender, borrower, liquidatables[i].amount);
                } else {
                    results[0][i] = CErc721Interface(liquidatables[i].cToken)._liquidateBorrow(msg.sender, borrower, liquidatables[i].nftIds);
                }

                uint priceMantissa = oracle_.getUnderlyingPrice(CToken(liquidatables[i].cToken));
                require(priceMantissa != 0, "PRICE_ERROR");

                liquidatedValueTotal = mul_ScalarTruncateAddUInt(Exp({mantissa: priceMantissa}), results[0][i], liquidatedValueTotal);

                unchecked { i++; }
            }
            require(liquidatedValueTotal != 0, "error");
        }

        {
            uint liquidatedValueRemaining = liquidatedValueTotal;
            for (uint i = 0; i < cTokenCollaterals.length && liquidatedValueRemaining != 0;) {

                require(markets[address(cTokenCollaterals[i])].isListed, "market not listed");

                {
                    /* We calculate the number of collateral tokens that will be seized */
                    uint seizeTokens = liquidateCalculateSeizeTokensNormed(address(cTokenCollaterals[i]), liquidatedValueRemaining);
                    uint actualSeizeTokens;

                    uint borrowerBalance = cTokenCollaterals[i].balanceOf(borrower);
                    if (borrowerBalance < seizeTokens) {
                        // can't seize more collateral than owned by the borrower
                        actualSeizeTokens = borrowerBalance;
                    } else {
                        actualSeizeTokens = seizeTokens;
                    }

                    actualSeizeTokens = cTokenCollaterals[i]._seize(msg.sender, borrower, actualSeizeTokens);
                    require(actualSeizeTokens != 0, "token seizure failed");

                    uint actualRepayAmount = liquidatedValueRemaining;
                    if (actualSeizeTokens != seizeTokens) {
                        actualRepayAmount = actualRepayAmount * actualSeizeTokens / seizeTokens;
                    }

                    liquidatedValueRemaining = liquidatedValueRemaining > actualRepayAmount ?
                        liquidatedValueRemaining - actualRepayAmount :
                        0;

                    results[1][i] = actualSeizeTokens;
                }

                unchecked { i++; }
            }

            require(liquidatedValueRemaining == 0, "LIQUIDATE_SEIZE_TOO_LITTLE");
        }

        /* We emit a LiquidateBorrow event */
        //emit LiquidateBorrow(liquidator, borrower, nftIds, repayInterest, cTokenCollaterals, seizeTokensList);

        uint afterRatio;
        (err, afterRatio) = getAccountDebtRatio(borrower);
        
        // we allow Error.TOO_LITTLE_INTEREST_RESERVE here as long as debt ratio is improved or unchanged
        require((err == uint(Error.NO_ERROR) || err == uint(Error.TOO_LITTLE_INTEREST_RESERVE))
            && afterRatio <= beforeRatio, "seize too much");

        return results;
    }
```

### Remediation

Add e.g. `minimumLiquidatedValue` parameter and add a check verifying that the
reward value is greater than or equal to the `minimumLiquidatedValue` value.

## Full Report

Full audit report can be found [here](https://github.com/Hexens/Smart-Contract-Review-Public-Reports/blob/main/Fungify_Nov23_(Public).pdf)
