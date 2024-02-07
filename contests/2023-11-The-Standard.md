# The Standard - Findings Report

# Table of contents
- ## [Contest Summary](#contest-summary)
- ## [Results Summary](#results-summary)

- ## Medium Risk Findings
    - ### [M-01. Wrong Burning Mechanism Leading to Wrong Calculations of Collateral and Minted](#M-01)
    - ### [M-02. Wrong Slippage Implementation](#M-02)
- ## Low Risk Findings
    - ### [L-01. Unrestricted fee rates can cause the stability of the protocol](#L-01)


# <a id='contest-summary'></a>Contest Summary

### Sponsor: The Standard

### Dates: Dec 27th, 2023 - Jan 10th, 2024

[See more contest details here](https://www.codehawks.com/contests/clql6lvyu0001mnje1xpqcuvl)

# <a id='results-summary'></a>Results Summary

### Number of findings:
   - High: 0
   - Medium: 2
   - Low: 1



		
# Medium Risk Findings

## <a id='M-01'></a>M-01. Wrong Burning Mechanism Leading to Wrong Calculations of Collateral and Minted            

### Relevant GitHub Links
	
https://github.com/Cyfrin/2023-12-the-standard/blob/91132936cb09ef9bf82f38ab1106346e2ad60f91/contracts/SmartVaultV3.sol#L169-L175

## Summary

In the current implementation of the `EURO` burn function within `SmartVaultV3.sol`, there are issues with the fee calculation and collateral requirement assessment. The issues arise when a user intends to burn a certain amount of `EURO`s and the associated fee, leading to potential undercollateralization risks.

## Vulnerability Details

Firstly, the burn fee, calculated at for example 2% of the `EURO`s to be burned, is currently deducted from the amount the user intends to burn, rather than being an independent transaction. This results in the user needing additional `EURO`s to cover the fee separately from the amount they want to burn (in case the user wants to burn all the `EURO`s it will revert).
Secondly, when a user burns a portion of their `EURO`s, the function correctly executes the `burn`, but fails to adjust the collateral requirement in proportion to the `EURO`s burned.

```solidity
    function burn(uint256 _amount) external ifMinted(_amount) {
        uint256 fee = _amount * ISmartVaultManagerV3(manager).burnFeeRate() / ISmartVaultManagerV3(manager).HUNDRED_PC();
        minted = minted - _amount;
        EUROs.burn(msg.sender, _amount);
        IERC20(address(EUROs)).safeTransferFrom(msg.sender, ISmartVaultManagerV3(manager).protocol(), fee);
        emit EUROsBurned(_amount, fee);
    }
```

Example Scenario:

Consider a scenario where a user has 1 Ether as collateral and has minted 1000 `EURO`s. Should they decide to burn 500 `EURO`s:

1. The user has minted 1000 `EURO`s and intends to burn 500 `EURO`s.
2. With a burn fee rate of e.g. 2%, the user needs 510 `EURO`s (500 + 10 fee) to execute the burn operation. But as function works, fee is calculated 10 `EURO`s, then `minted` parameter becomes `minted = minted - _amount`: `1000 - 500 = 500`.
3. Then 500 `EURO`s burned, and user has to pay 10 `EURO`s as a fee.
4. After the successful burn and paid fee, the user remains with 490 `EURO`s and 1 Ether as a collateral.

Now the user can adjust their collateral with `minted` parameter, so the user needs to have as much collateral as it intended to for 500 `EURO`s ( `minted = 500` ), but the user has actually 490 `EURO`s. So the user can adjust their collateral for 490 `EURO`s and be liquidated.

## Impact

Undercollateralization Risk: Users might face undercollateralization if the collateral is not adjusted according to the remaining `EURO` balance after the `burn`.

Liquidation: Inadequate collateral coverage for the `EURO`s can lead to the liquidation of the vault, causing a loss of assets for the user.

## Tools Used

Manual review.

## Recommendations

Revise the `burn` function to ensure that the fee is calculated separately from the `EURO`s to be burned, rather than being deducted from the user's existing `EURO` balance. After each `burn` operation, recalculate the required collateral based on the remaining `EURO` balance to maintain proper collateralization.
## <a id='M-02'></a>M-02. Wrong Slippage Implementation            

### Relevant GitHub Links
	
https://github.com/Cyfrin/2023-12-the-standard/blob/91132936cb09ef9bf82f38ab1106346e2ad60f91/contracts/SmartVaultV3.sol#L206-L212

## Summary

The code employs an on-chain slippage calculation mechanism through the `calculateMinimumAmountOut` function, utilized within the `swap` function. Consequently, this function may return `0` when the collateralization is deemed adequate.

```solidity
        return collateralValueMinusSwapValue >= requiredCollateralValue ?
            0 : calculator.eurToToken(getToken(_outTokenSymbol), requiredCollateralValue - collateralValueMinusSwapValue);
```

## Vulnerability Details

The absence of slippage checks and minimum return amount validations in the code could result in trades occurring at suboptimal prices, potentially leading to the reception of fewer tokens than would be expected at prevailing fair market rates. This vulnerability might expose the vault owner to risks of incurring losses due to unfavorable prices at the time of trade execution. Because the `swap` function will call `ExactInputSingleParams` function with `amountOutMinimum` set to `0`:

```solidity
    function swap(bytes32 _inToken, bytes32 _outToken, uint256 _amount) external onlyOwner {
        uint256 swapFee = _amount * ISmartVaultManagerV3(manager).swapFeeRate() / ISmartVaultManagerV3(manager).HUNDRED_PC();
        address inToken = getSwapAddressFor(_inToken);
        uint256 minimumAmountOut = calculateMinimumAmountOut(_inToken, _outToken, _amount);
        ISwapRouter.ExactInputSingleParams memory params = ISwapRouter.ExactInputSingleParams({
                tokenIn: inToken,
                tokenOut: getSwapAddressFor(_outToken),
                fee: 3000,
                recipient: address(this),
                deadline: block.timestamp,
                amountIn: _amount - swapFee,
                amountOutMinimum: minimumAmountOut,
                sqrtPriceLimitX96: 0
            });
        inToken == ISmartVaultManagerV3(manager).weth() ?
            executeNativeSwapAndFee(params, swapFee) :
            executeERC20SwapAndFee(params, swapFee);
    }
```

## Impact

The risk associated with the absence of slippage and minimum return amount checks lies in potential price volatility during the swap. Trades can happen at a bad price and lead to receiving fewer tokens than at a fair market price. 

## Tools Used

Manual review.

## Recommendations

Ensure that users are allowed to specify their own slippage parameters which were calculated on their own e.g off-chain.

# Low Risk Findings

## <a id='L-01'></a>L-01. Unrestricted fee rates can cause the stability of the protocol            

### Relevant GitHub Links
	
https://github.com/Cyfrin/2023-12-the-standard/blob/c12272f2eec533019f2d255ab690f6892027f112/contracts/LiquidationPoolManager.sol#L28

https://github.com/Cyfrin/2023-12-the-standard/blob/c12272f2eec533019f2d255ab690f6892027f112/contracts/LiquidationPoolManager.sol#L84-L86

https://github.com/Cyfrin/2023-12-the-standard/blob/c12272f2eec533019f2d255ab690f6892027f112/contracts/SmartVaultManagerV5.sol#L103-L113

## Summary

The contracts `LiquidationPoolManager` and `SmartVaultManagerV5` exhibit a vulnerability where fee parameters (`poolFeePercentage`, `mintFeeRate`, `burnFeeRate`, and `swapFeeRate`) are susceptible to unbounded values. This oversight could lead to the imposition of exorbitant fees, potentially deterring user engagement, causing economic instability, and resulting in unpredictable protocol behavior.

## Vulnerability Details

Both contracts lack constraints or validation checks on key fee parameters. Specifically:

1. `LiquidationPoolManager`: The `poolFeePercentage` has no upper limit set, allowing the contract owner to define excessive fees for liquidation.
2. `SmartVaultManagerV5`: The `mintFeeRate`, `burnFeeRate`, and `swapFeeRate` are also not constrained, enabling the setting of unbounded fee percentages for various operations within the smart vault management.

## Impact

`LiquidationPoolManager.sol` contract: Unrestricted fee settings within the `LiquidationPoolManager` contract pose a significant threat to the stability and fairness of the protocol. High or unchecked fees might result in market distortions, unfair trading conditions, and potential economic instability. And absence of limitations on fee configurations could result in a loss of user confidence, leading to a perception of unpredictability and exploitation within the protocol's operations.

`SmartVaultManagerV5.sol` contract: Unbounded fee settings in SmartVaultManagerV5 present a vulnerability, potentially leading to market instability, unfair conditions, and adverse economic effects. Unchecked fee structures might erode user trust by creating an unpredictable and potentially exploitative environment within the protocol's ecosystem.

## Tools Used

Manual review.

## Recommendations

Add reasonable upper limits for fee parameters to prevent the imposition of excessively high fees.


