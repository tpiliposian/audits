# MorpheusAI - Findings Report

# Table of contents
- ## [Contest Summary](#contest-summary)
- ## [Results Summary](#results-summary)

- ## Medium Risk Findings
    - ### [M-01. Relying on block.timestamp for setting a swap deadline](#M-01)



# <a id='contest-summary'></a>Contest Summary

### Sponsor: MorpheusAI

### Dates: Jan 30th, 2024 - Feb 3rd, 2024

[See more contest details here](https://www.codehawks.com/contests/clrzgrole0007xtsq0gfdw8if)

# <a id='results-summary'></a>Results Summary

### Number of findings:
   - High: 0
   - Medium: 1
   - Low: 0



		
# Medium Risk Findings

## <a id='M-01'></a>M-01. Relying on block.timestamp for setting a swap deadline            

### Relevant GitHub Links
	
https://github.com/Cyfrin/2024-01-Morpheus/blob/76898177fbedcbbf4b78b513d9fa151bbf3388de/contracts/L2TokenReceiver.sol#L65

https://github.com/Cyfrin/2024-01-Morpheus/blob/76898177fbedcbbf4b78b513d9fa151bbf3388de/contracts/L2TokenReceiver.sol#L112

## Summary

The `L2TokenReceiver.sol` contract implements the deadline of the `swap` and `increaseLiquidityCurrentRange` wrong.

## Vulnerability Details

Proposers can anticipate proposing single or consecutive blocks in advance. In this situation, a malicious validator can delay the transaction, strategically executing it at a more advantageous `block number`.

```solidity
    function swap(uint256 amountIn_, uint256 amountOutMinimum_) external onlyOwner returns (uint256) {
        SwapParams memory params_ = params;

        ISwapRouter.ExactInputSingleParams memory swapParams_ = ISwapRouter.ExactInputSingleParams({
            tokenIn: params_.tokenIn,
            tokenOut: params_.tokenOut,
            fee: params_.fee,
            recipient: address(this),
            deadline: block.timestamp,
            amountIn: amountIn_,
            amountOutMinimum: amountOutMinimum_,
            sqrtPriceLimitX96: params_.sqrtPriceLimitX96
        });

        uint256 amountOut_ = ISwapRouter(router).exactInputSingle(swapParams_);

        emit TokensSwapped(params_.tokenIn, params_.tokenOut, amountIn_, amountOut_, amountOutMinimum_);

        return amountOut_;
    }

    function increaseLiquidityCurrentRange(
        uint256 tokenId_,
        uint256 depositTokenAmountAdd_,
        uint256 rewardTokenAmountAdd_,
        uint256 depositTokenAmountMin_,
        uint256 rewardTokenAmountMin_
    ) external onlyOwner returns (uint128 liquidity_, uint256 amount0_, uint256 amount1_) {
        uint256 amountAdd0_;
        uint256 amountAdd1_;
        uint256 amountMin0_;
        uint256 amountMin1_;

        (, , address token0_, , , , , , , , , ) = INonfungiblePositionManager(nonfungiblePositionManager).positions(
            tokenId_
        );
        if (token0_ == params.tokenIn) {
            amountAdd0_ = depositTokenAmountAdd_;
            amountAdd1_ = rewardTokenAmountAdd_;
            amountMin0_ = depositTokenAmountMin_;
            amountMin1_ = rewardTokenAmountMin_;
        } else {
            amountAdd0_ = rewardTokenAmountAdd_;
            amountAdd1_ = depositTokenAmountAdd_;
            amountMin0_ = rewardTokenAmountMin_;
            amountMin1_ = depositTokenAmountMin_;
        }

        INonfungiblePositionManager.IncreaseLiquidityParams memory params_ = INonfungiblePositionManager
            .IncreaseLiquidityParams({
                tokenId: tokenId_,
                amount0Desired: amountAdd0_,
                amount1Desired: amountAdd1_,
                amount0Min: amountMin0_,
                amount1Min: amountMin1_,
                deadline: block.timestamp
            });

        (liquidity_, amount0_, amount1_) = INonfungiblePositionManager(nonfungiblePositionManager).increaseLiquidity(
            params_
        );

        emit LiquidityIncreased(tokenId_, amount0_, amount1_, liquidity_, amountMin0_, amountMin1_);
    }
```

## Impact

This provides no safeguard, as the `block.timestamp` will reflect the value of the block in which the transaction is included. Consequently, malicious validators can indefinitely withhold the transaction.

## Tools Used

Manual review.

## Recommendations

Consider enabling users to specify a `deadline` parameter for their transactions




