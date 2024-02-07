# Steadefi - Findings Report

# Table of contents
- ## [Contest Summary](#contest-summary)
- ## [Results Summary](#results-summary)

- ## Medium Risk Findings
    - ### [M-01. Lack of Slippage and Execution Fee Checks](#M-01)
    - ### [M-02. Attacker can frontrun receiving native tokens](#M-02)
    - ### [M-03. Invalid checks can lead to protocol disruptions](#M-03)
- ## Low Risk Findings
    - ### [L-01. Redundant import of Ownable contract](#L-01)
    - ### [L-02. Constant variables should be marked as private](#L-02)


# <a id='contest-summary'></a>Contest Summary

### Sponsor: Steadefi

### Dates: Oct 26th, 2023 - Nov 6th, 2023

[See more contest details here](https://www.codehawks.com/contests/clo38mm260001la08daw5cbuf)

# <a id='results-summary'></a>Results Summary

### Number of findings:
   - High: 0
   - Medium: 3
   - Low: 2



		
# Medium Risk Findings

## <a id='M-01'></a>M-01. Lack of Slippage and Execution Fee Checks            

### Relevant GitHub Links
	
https://github.com/Cyfrin/2023-10-SteadeFi/blob/1841efff70defe305d53a850c49acffda788a401/contracts/strategy/gmx/GMXVault.sol#L652-L669

## Summary

The `GMXVault.sol` contract contains functions to update minimum slippage and minimum execution fees. However, these functions lack explicit checks to ensure that the provided values conform to the specified ranges.

## Vulnerability Details

The functions `updateMinSlippage` and `updateMinExecutionFee` do not include checks to verify whether the input values for `minSlippage` and `minExecutionFee` adhere to the expected ranges of `1e4` and `1e18`, respectively.
For example, mistakenly setting slippage or execution fees to zero could bypass important checks, for slippage potentially impacts the functioning of the contract and leads to no slippage protection. While it's primarily a concern of the owner's actions, there is a risk of unintended behavior if these values are not appropriately configured.

## Impact

Setting slippage or execution fees to zero means the protocol will operate without any slippage protection. This could lead to undesirable consequences, such as higher trading risks, especially in volatile markets. 

## Tools Used

Manual code review.

## Recommendations

Add checks within the `updateMinSlippage` and `updateMinExecutionFee` functions. These checks should ensure that the provided values are within acceptable bounds, preventing the inadvertent setting of these parameters to zero.

For example. 

```solidity
function updateMinSlippage(uint256 minSlippage) external onlyOwner {
    require(minSlippage >= 1e4, "Slippage value must be greater than or equal to 1e4");
    _store.minSlippage = minSlippage;
    emit MinSlippageUpdated(minSlippage);
}

function updateMinExecutionFee(uint256 minExecutionFee) external onlyOwner {
    require(minExecutionFee >= 1e18, "Execution fee value must be greater than or equal to 1e18");
    _store.minExecutionFee = minExecutionFee;
    emit MinExecutionFeeUpdated(minExecutionFee);
}
```
## <a id='M-02'></a>M-02. Attacker can frontrun receiving native tokens            

### Relevant GitHub Links
	
https://github.com/Cyfrin/2023-10-SteadeFi/blob/1841efff70defe305d53a850c49acffda788a401/contracts/strategy/gmx/GMXVault.sol#L697-L702

## Summary

The vulnerability is related to the `GMXVaul.sol` contract's `receive()` function. It lacks proper safeguards, potentially enabling frontrunning attacks and allowing an attacker to manipulate the refund mechanism to gain undeserved native tokens.

```solidity
  receive() external payable {
    if (msg.sender == _store.depositVault || msg.sender == _store.withdrawalVault) {
      (bool success, ) = _store.refundee.call{value: address(this).balance}("");
      require(success, "Transfer failed.");
    }
  }
```

## Vulnerability Details

The vulnerability arises from the contract's `receive()` function, which refunds native tokens to a specific address under certain conditions. This function doesn't include proper safeguards to prevent potential frontrunning attacks, allowing an attacker to manipulate the refund mechanism to gain undeserved native tokens.

An attacker can frontrun and before `receive()` function call e.g. `GMXDeposit.sol` contract's `deposit()` function and become a `refundee`: `self.refundee = payable(msg.sender)` and gain all native tokens from fallback function.

```solidity
  function deposit(
    GMXTypes.Store storage self,
    GMXTypes.DepositParams memory dp,
    bool isNative
  ) external {
    // Sweep any tokenA/B in vault to the temporary trove for future compouding and to prevent
    // it from being considered as part of depositor's assets
    if (self.tokenA.balanceOf(address(this)) > 0) {
      self.tokenA.safeTransfer(self.trove, self.tokenA.balanceOf(address(this)));
    }
    if (self.tokenB.balanceOf(address(this)) > 0) {
      self.tokenB.safeTransfer(self.trove, self.tokenB.balanceOf(address(this)));
    }

    self.refundee = payable(msg.sender);

    GMXTypes.HealthParams memory _hp;

    _hp.equityBefore = GMXReader.equityValue(self);
    _hp.lpAmtBefore = GMXReader.lpAmt(self);
    _hp.debtRatioBefore = GMXReader.debtRatio(self);
    _hp.deltaBefore = GMXReader.delta(self);

    // Transfer assets from user to vault
    if (isNative) {
      GMXChecks.beforeNativeDepositChecks(self, dp);

      self.WNT.deposit{ value: dp.amt }();
    } else {
      IERC20(dp.token).safeTransferFrom(msg.sender, address(this), dp.amt);
    }

    GMXTypes.DepositCache memory _dc;

    _dc.user = payable(msg.sender);

    if (dp.token == address(self.lpToken)) {
      // If LP token deposited
      _dc.depositValue = self.gmxOracle.getLpTokenValue(
        address(self.lpToken),
        address(self.tokenA),
        address(self.tokenA),
        address(self.tokenB),
        false,
        false
      )
      * dp.amt
      / SAFE_MULTIPLIER;
    } else {
      // If tokenA or tokenB deposited
      _dc.depositValue = GMXReader.convertToUsdValue(
        self,
        address(dp.token),
        dp.amt
      );
    }
    _dc.depositParams = dp;
    _dc.healthParams = _hp;

    self.depositCache = _dc;

    GMXChecks.beforeDepositChecks(self, _dc.depositValue);

    self.status = GMXTypes.Status.Deposit;

    self.vault.mintFee();

    // Borrow assets and create deposit in GMX
    (
      uint256 _borrowTokenAAmt,
      uint256 _borrowTokenBAmt
    ) = GMXManager.calcBorrow(self, _dc.depositValue);

    _dc.borrowParams.borrowTokenAAmt = _borrowTokenAAmt;
    _dc.borrowParams.borrowTokenBAmt = _borrowTokenBAmt;

    GMXManager.borrow(self, _borrowTokenAAmt, _borrowTokenBAmt);

    GMXTypes.AddLiquidityParams memory _alp;

    _alp.tokenAAmt = self.tokenA.balanceOf(address(this));
    _alp.tokenBAmt = self.tokenB.balanceOf(address(this));
    _alp.minMarketTokenAmt = GMXManager.calcMinMarketSlippageAmt(
      self,
      _dc.depositValue,
      dp.slippage
    );
    _alp.executionFee = dp.executionFee;

    _dc.depositKey = GMXManager.addLiquidity(
      self,
      _alp
    );

    self.depositCache = _dc;

    emit DepositCreated(
      _dc.user,
      _dc.depositParams.token,
      _dc.depositParams.amt
    );
  }
```

## Impact

An attacker could, in a frontrunning manner, deposit a small amount and become eligible for a refund. This could lead to an unintended transfer of native tokens to the attacker, effectively draining the contract's native token balance. 

## Tools Used

Manual review.

## Recommendations

To mitigate this vulnerability, it is advised to implement additional checks and safeguards in the receive function. These checks should ensure that only valid refund requests are processed and that they are protected against manipulation or unauthorized access. Additionally, consider alternative design patterns to handle refunds securely.
## <a id='M-03'></a>M-03. Invalid checks can lead to protocol disruptions            

### Relevant GitHub Links
	
https://github.com/Cyfrin/2023-10-SteadeFi/blob/1841efff70defe305d53a850c49acffda788a401/contracts/oracles/ChainlinkARBOracle.sol#L249-L268

## Summary

The `addTokenMaxDelay` and `addTokenMaxDeviation` functions in the `ChainlinkARBOracle.sol` contract have redundant checks for the `maxDelay` and `maxDeviation` `uint256` parameters. These functions are designed to be called by the owner to configure certain settings, but the `maxDelay` and `maxDeviation` parameters are both checked to be equal or greater than 0, which is not only redundant because they are `uint256` which can't be smaller than 0, but also the fact that the parameters can be set to 0 could lead to certain important functions not working.

```solidity
  function addTokenMaxDelay(address token, uint256 maxDelay) external onlyOwner {
    if (token == address(0)) revert Errors.ZeroAddressNotAllowed();
    if (feeds[token] == address(0)) revert Errors.NoTokenPriceFeedAvailable();
    if (maxDelay < 0) revert Errors.TokenPriceFeedMaxDelayMustBeGreaterOrEqualToZero();

    maxDelays[token] = maxDelay;
  }

  /**
    * @notice Add Chainlink max deviation for token
    * @param token Token address
    * @param maxDeviation  Max deviation allowed in seconds
  */
  function addTokenMaxDeviation(address token, uint256 maxDeviation) external onlyOwner {
    if (token == address(0)) revert Errors.ZeroAddressNotAllowed();
    if (feeds[token] == address(0)) revert Errors.NoTokenPriceFeedAvailable();
    if (maxDeviation < 0) revert Errors.TokenPriceFeedMaxDeviationMustBeGreaterOrEqualToZero();

    maxDeviations[token] = maxDeviation;
  }
```

## Vulnerability Details

The `maxDelay` and `maxDeviation` parameters are checked for values less than 0. However, since these parameters are of type `uint256`, they cannot be negative by definition. While this check doesn't directly pose a security risk, it can lead to issues if the owner mistakenly assigns a value of 0 to these parameters. In such cases, important functions will not work as intended, affecting the operational stability of the contract. It is crucial to remove this redundant check to prevent inadvertent problems caused by zero values.

In case `maxDelay` is set to 0, the `_chainlinkIsFrozen` check in the `consult` function will always revert:

```solidity
  function consult(address token) public view whenNotPaused returns (int256, uint8) {
    address _feed = feeds[token];

    if (_feed == address(0)) revert Errors.NoTokenPriceFeedAvailable();

    ChainlinkResponse memory chainlinkResponse = _getChainlinkResponse(_feed);
    ChainlinkResponse memory prevChainlinkResponse = _getPrevChainlinkResponse(_feed, chainlinkResponse.roundId);

    if (_chainlinkIsFrozen(chainlinkResponse, token)) revert Errors.FrozenTokenPriceFeed();
    if (_chainlinkIsBroken(chainlinkResponse, prevChainlinkResponse, token)) revert Errors.BrokenTokenPriceFeed();

    return (chainlinkResponse.answer, chainlinkResponse.decimals);
  }
```

These functions are using Chainlink's Oracle and therefore are affected:

1. `GMXDeposit.sol` `deposit()`
2. `GMXCompound.sol` `compound()`
3. `GMXManager.sol` `calcBorrow()`

etc.

## Impact

The impact of this vulnerability is primarily operational and could potentially lead to significant disruptions within the contract. If the owner inadvertently assigns a value of 0 to the `maxDelay` or `maxDeviation` parameters, it will render crucial functions inoperative. For instance, functions relying on these parameters to check token pricing will inadvertently revert which will cause protocol disruptions.

## Tools Used

Manual review.

## Recommendations

Change checks to be strictly greater than 0.

For example:

```diff
-- if (maxDelay < 0) revert Errors.TokenPriceFeedMaxDelayMustBeGreaterOrEqualToZero();
++ if (maxDelay == 0) revert Errors.TokenPriceFeedMaxDelayMustBeGreaterThanZero();
```

```diff
-- if (maxDeviation < 0) revert Errors.TokenPriceFeedMaxDelayMustBeGreaterOrEqualToZero();
++ if (maxDeviation == 0) revert Errors.TokenPriceFeedMaxDelayMustBeGreaterThanZero();
```

# Low Risk Findings

## <a id='L-01'></a>L-01. Redundant import of Ownable contract            

### Relevant GitHub Links
	
https://github.com/Cyfrin/2023-10-SteadeFi/blob/1841efff70defe305d53a850c49acffda788a401/contracts/oracles/ChainlinkARBOracle.sol#L4

https://github.com/Cyfrin/2023-10-SteadeFi/blob/1841efff70defe305d53a850c49acffda788a401/contracts/strategy/gmx/GMXVault.sol#L6

## Summary

The mentioned contracts import both `Ownable` and `Ownable2Step` from the OpenZeppelin contracts. However, a redundancy is present, as `Ownable2Step` itself imports `Ownable`. This redundancy in imports is not only unnecessary but also results in higher gas costs during deployment and usage.

## Vulnerability Details

The issue arises from an import redundancy in the contract. Specifically, contracts import both `Ownable` and `Ownable2Step` contracts. However, `Ownable2Step` internally imports `Ownable`, making the direct import of `Ownable` redundant.

## Impact

It leads to increased gas costs when deploying and interacting with the contract. 

## Tools Used

Manual review.

## Recommendations

To optimize the contract and reduce gas costs, remove the redundant import of `Ownable.sol`.
## <a id='L-02'></a>L-02. Constant variables should be marked as private            

### Relevant GitHub Links
	
https://github.com/Cyfrin/2023-10-SteadeFi/blob/1841efff70defe305d53a850c49acffda788a401/contracts/strategy/gmx/GMXEmergency.sol#L21-L22

https://github.com/Cyfrin/2023-10-SteadeFi/blob/1841efff70defe305d53a850c49acffda788a401/contracts/strategy/gmx/GMXChecks.sol#L22-L23

https://github.com/Cyfrin/2023-10-SteadeFi/blob/1841efff70defe305d53a850c49acffda788a401/contracts/strategy/gmx/GMXCompound.sol#L22

https://github.com/Cyfrin/2023-10-SteadeFi/blob/1841efff70defe305d53a850c49acffda788a401/contracts/strategy/gmx/GMXDeposit.sol#L27

https://github.com/Cyfrin/2023-10-SteadeFi/blob/1841efff70defe305d53a850c49acffda788a401/contracts/strategy/gmx/GMXEmergency.sol#L21-L22

https://github.com/Cyfrin/2023-10-SteadeFi/blob/1841efff70defe305d53a850c49acffda788a401/contracts/strategy/gmx/GMXManager.sol#L23

https://github.com/Cyfrin/2023-10-SteadeFi/blob/1841efff70defe305d53a850c49acffda788a401/contracts/strategy/gmx/GMXReader.sol#L19

https://github.com/Cyfrin/2023-10-SteadeFi/blob/1841efff70defe305d53a850c49acffda788a401/contracts/strategy/gmx/GMXWithdraw.sol#L22

https://github.com/Cyfrin/2023-10-SteadeFi/blob/1841efff70defe305d53a850c49acffda788a401/contracts/strategy/gmx/GMXWorker.sol#L13

https://github.com/Cyfrin/2023-10-SteadeFi/blob/1841efff70defe305d53a850c49acffda788a401/contracts/oracles/ChainlinkARBOracle.sol#L26-L27

## Summary

Setting `constants` to `private` will save deployment gas. This is because the compiler won't have to create non-payable getter functions for deployment calldata, won't need to store the bytes of the values outside of where it's used, and won't add another entry to the method ID table. The values can still be read from the verified contract source code if necessary.

## Vulnerability Details

In the mentioned links there are constant variables, that are `public`.

## Impact

Deployment gas waste.

## Tools Used

Manual review.

## Recommendations

Change constants from public to private:

e.g:

```diff
-- uint256 public constant SAFE_MULTIPLIER = 1e18;
++ uint256 private constant SAFE_MULTIPLIER = 1e18;
```


