## Griefing of the ICLFactory(factory).createPool() by creating the pool first

High

### Description

When calling `finalizeFundraising()` function, it does necessary checks and calculations after which it is turn to `MintParams` and call `POSITION_MANAGER.mint(params)`.

https://github.com/velodrome-finance/slipstream/blob/7b50de4648ec340891a8d5c1366c83983308d3b9/contracts/periphery/interfaces/INonfungiblePositionManager.sol#L91-L104

```solidity
        INonfungiblePositionManager.MintParams
            memory params = INonfungiblePositionManager.MintParams(
                token0,
                token1,
                TICKING_SPACE,
                initialTick,
                upperTick,
                amountToken0ForLP,
                amountToken1ForLP,
                0,
                0,
                address(this),
                block.timestamp,
                sqrtPriceX96
            );

        token.renounceOwnership();
        IERC20(token0).approve(address(POSITION_MANAGER), amountToken0ForLP);
        IERC20(token1).approve(address(POSITION_MANAGER), amountToken1ForLP);
        emit DebugLog("Minted additional tokens for LP");

        (uint256 tokenId, , , ) = POSITION_MANAGER.mint(params);  //@audit-issue pool creation griefing
        emit LPTokenMinted(tokenId);
```

When calling `POSITION_MANAGER.mint()` the function calculates `sqrtPriceX96` and passes it as a parameter. This is being done because there is no pool created before, so the `mint` needs to first create a pool:

https://github.com/velodrome-finance/slipstream/blob/7b50de4648ec340891a8d5c1366c83983308d3b9/contracts/periphery/NonfungiblePositionManager.sol#L143-L201

https://explorer.mode.network/address/0x991d5546C4B442B4c5fdc4c8B8b8d131DEB24702?tab=contract

```solidity
    /// @inheritdoc INonfungiblePositionManager
    function mint(MintParams calldata params)
        external
        payable
        override
        checkDeadline(params.deadline)
        returns (uint256 tokenId, uint128 liquidity, uint256 amount0, uint256 amount1)
    {
        if (params.sqrtPriceX96 != 0) {
            ICLFactory(factory).createPool({
                tokenA: params.token0,
                tokenB: params.token1,
                tickSpacing: params.tickSpacing,
                sqrtPriceX96: params.sqrtPriceX96
            });
        }
.....
```

In our case the call is always being done via non-zero `sqrtPriceX96` so it will always call `ICLFactory(factory).createPool()`. The issue is that an attacker can create a pool beforehand by directly calling `createPool`. They even dont need to have a tokens when creating a new pool. So the Daao's call to the `mint` will revert:

```solidity
    /// @notice Creates a pool for the given two tokens and tick spacing
    /// @param tokenA One of the two tokens in the desired pool
    /// @param tokenB The other of the two tokens in the desired pool
    /// @param tickSpacing The desired tick spacing for the pool
    /// @param sqrtPriceX96 The initial sqrt price of the pool, as a Q64.96
>>    /// @dev tokenA and tokenB may be passed in either order: token0/token1 or token1/token0. The call will
>>    /// revert if the pool already exists, the tick spacing is invalid, or the token arguments are invalid
    /// @return pool The address of the newly created pool
    function createPool(address tokenA, address tokenB, int24 tickSpacing, uint160 sqrtPriceX96)
        external
        returns (address pool);
```

### Recommendation

The `CLFactory` has `getPool()` function, which returns the pool address for a given pair of tokens and a tick spacing, or address 0 if it does not exist. So first call `getPool()` function, if there is already a pool created, pass the `sqrtPriceX96` parameter as a 0, otherwise call it as it is being called now.
