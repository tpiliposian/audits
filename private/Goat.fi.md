# Preliminary Security Review Report

### **[Goat.fi](https://www.goat.fi/#/)** (DeFi Yield Optimizer)
Led by [Tigran Piliposyan](https://twitter.com/tpiliposian) at 14.02.2024

# About **Tigran Piliposyan**

Tigran Piliposyan ([tpiliposian](https://twitter.com/tpiliposian)) is a security researcher specializing in smart contract audits and security consulting.

Currently, he holds the role of Security Researcher at [Hexens](https://hexens.io/) and a Core Contributor at [Security Wiki](https://wiki.r.security/). 

Previously, he held roles in the financial sphere for over a decade, including a tenure at the Central Bank of Armenia, where he led the Risk Management Division: [LinkedIn](https://www.linkedin.com/in/tpiliposyan/).

Feel free to connect with him at:
- X/Twitter - [@tpiliposian](https://twitter.com/tpiliposian) 
- Telegram - [@tpiliposian](https://t.me/tpiliposian)

# Disclaimer

A smart contract security review is a comprehensive attempt to uncover vulnerabilities, but it cannot establish their complete absence. The process is bounded by time, resources, and expertise, aiming to identify as many potential issues as possible. However, it does not assure absolute security for the protocol.

# Severity classification

| Severity Level | Impact: High | Impact: Medium | Impact: Low |
| - | - | - | - |
| Likelihood: High| High | High |  Medium |
| Likelihood: Medium | High | Medium |  Low |
| Likelihood: Low | Medium | Low |  Low |

## Impact

- High - Funds are directly at risk or a severe disruption of the protocol's core functionality.
- Medium - Funds are indirectly at risk or some disruption of the protocol's functionality.
- Low - Funds are not at risk.

## Likelihood

- High - Highly likely to occur.
- Medium - Might occur under specific conditions.
- Low - Unlikely to occur.

# About Goat Protocol

The Goat Protocol is a decentralized yield optimizer. It allows users, DAOs and other protocols earn the yield on their digital assets by auto compounding the rewards into more of what they've deposited.

# Summary & Scope

The [Goat.fi](https://www.goat.fi/#/) repository was audited at commit [724d610f4f5d7bb9abf0b965a66cbf0ec809953b](https://github.com/goatfi/contracts/commit/724d610f4f5d7bb9abf0b965a66cbf0ec809953b#diff-2e49d11dda360152884d6f7a372562327ce7f06c8ec2ef52cf6b9f8dc9794400).

The following contracts were in scope:
1. `src/infra/GoatRewardPool.sol` (~200 nSLOC)

After the completion of the fixes, the [Y](Y.com) commit was reviewed.

# Summary of Findings

| Severity | Issues Found |
| - | - |
| High | 1 |
| Medium | 2 |
| Low | 0 |
| Informational | 2 |

### Summary

| Identifier | Title | Severity | Status |
| - | - | - | - |
| [H-01] | Unfair reward distribution due to wrong calculations | High |  Fixed/Acknowledged |
| [M-01] | Stake with permit can be blocked | Medium |  Fixed/Acknowledged |
| [M-02] | Inconsistent decimal place handling in reward calculations | Medium |  Fixed/Acknowledged |
| [I-01] | Use Ownable2Step instead of Ownable | Informational |  Fixed/Acknowledged |
| [I-02] | Unnecessary initialization of constant in constructor | Informational |  Fixed/Acknowledged |

# Findings

## [H-01] Unfair reward distribution due to wrong calculations

### Description

Path: `GoatRewardPool.sol:_rewardPerToken()` https://github.com/goatfi/contracts/blob/724d610f4f5d7bb9abf0b965a66cbf0ec809953b/src/infra/GoatRewardPool.sol#L379-L390

The current implementation of the `_rewardPerToken` function in the `GoatRewardPool.sol` contract leads to an inequitable distribution of rewards among users who stake their tokens at different times. This results in users who stake earlier receiving lower rewards compared to users who stake later, i.e. after every new `stake` the reward per token for previous stakers decreases, because the `_rewardPerToken` function:

```solidity
    function _rewardPerToken(address _reward) private view returns (uint256 rewardPerToken) {
        RewardInfo storage rewardData = _getRewardInfo(_reward);
        if (totalSupply() == 0) {
            rewardPerToken = rewardData.rewardPerTokenStored;
        } else {
            rewardPerToken = rewardData.rewardPerTokenStored + Math.mulDiv(
                (_lastTimeRewardApplicable(rewardData.periodFinish) - rewardData.lastUpdateTime),
                rewardData.rate * 1e30,
                totalSupply()
            );
        }
    }
```
calculates the reward amount per token using the `totalSupply()` of staked tokens. This means that the reward distribution is influenced by the total number of tokens staked in the pool, so when a user stakes tokens, it increases the `totalSupply()`, consequently reducing the reward per token for existing stakers.

### Proof of Concept

In the following testing scenario where a user calls `getReward()` without any prior staking activity, they receive more rewards compared to scenarios where someone stakes tokens before the user calls `getReward()`:

In the `GoatFarm.rewards.t.sol` add `import "forge-std/console.sol";` and these two test functions and run the tests:

```solidity
    function test_without_anybodystakedbefore_getreward() public {
        _notifyRewards(FARM_REWARD);
        vm.warp(block.timestamp + DURATION);

        uint256 previousRewardTokenBalance = rewardToken.balanceOf(USER);

        vm.prank(USER);
        farm.getReward();

        assertEq(previousRewardTokenBalance, 0);
        console.log(rewardToken.balanceOf(USER));
        // assertEq(rewardToken.balanceOf(USER), FARM_REWARD);
    }

    function test_stakedbefore_getreward() public {
        _notifyRewards(FARM_REWARD);
        vm.warp(block.timestamp + DURATION/2);

        uint256 previousRewardTokenBalance = rewardToken.balanceOf(USER);

        _userDepositWETH(makeAddr("bob"), 100 ether);
        vm.warp(block.timestamp + DURATION/2);
        vm.prank(USER);
        farm.getReward();

        assertEq(previousRewardTokenBalance, 0);
        console.log(rewardToken.balanceOf(USER));
        // assertEq(rewardToken.balanceOf(USER), FARM_REWARD);
    }
```

The result:

```
[PASS] test_stakedbefore_getreward() (gas: 337398)
Logs:
  545454545454545454540

[PASS] test_without_anybodystakedbefore_getreward() (gas: 223286)
Logs:
  1000000000000000000000
```

### Recommendation

Ensure a fair reward distribution system, and revise the calculation methodology in the `_rewardPerToken` function. Instead of using the `totalSupply()` of staked tokens, the function should calculate rewards based on the individual user's share of the total staked tokens. 

## [M-01] Stake with permit can be blocked

### Description

Path: `GoatRewardPool.sol:stakeWithPermit()` https://github.com/goatfi/contracts/blob/724d610f4f5d7bb9abf0b965a66cbf0ec809953b/src/infra/GoatRewardPool.sol#L124-L136

The `stakeWithPermit` function internally calls `permit()` function from `IERC20Permit(address(stakedToken))`. However, this flow exposes `stakeWithPermit` to a griefing attack, where an attacker can forcibly block the victim's transaction.

```solidity
    function stakeWithPermit(
        address _user,
        uint256 _amount,
        uint256 _deadline,
        uint8 _v,
        bytes32 _r,
        bytes32 _s
    ) external update(_user) {
        IERC20Permit(address(stakedToken)).permit(
            _user, address(this), _amount, _deadline, _v, _r, _s
        );
        _stake(_user, _amount);
    }
```

Attack scenario: the attacker front-runs the victim's transaction, extracts parameters from the mempool, and places a transaction that directly calls `IERC20Permit(address(stakedToken).permit()` with parameters given by the victim. Consequently, the victim's transaction reverts since parameters have already been used for `permit()` in the attacker's transaction.

### Recommendation

You can check how `The Graph` solved this issue by adding their own `_permit` like this: https://github.com/edgeandnode/billing-contracts/blob/4aa2c70702da61c67b9d58cf066773a3b1dde539/contracts/BillingConnector.sol#L268-L299

Consider adding `try/cath`, so if the `permit()` is already called, just check the allowance of `msg.sender` and skip the call to `pemit()`:

An example from `The Grapgh`:

```solidity
        IERC20WithPermit token = IERC20WithPermit(address(graphToken));
        // Try permit() before allowance check to advance nonce if possible
        try token.permit(_owner, _spender, _value, _deadline, _v, _r, _s) {
            return;
        } catch Error(string memory reason) {
            // Check for existing allowance before reverting
            if (token.allowance(_owner, _spender) >= _value) {
                return;
            }

            revert(reason);
        }
```

## [M-02] Inconsistent decimal place handling in reward calculations

### Description

https://github.com/goatfi/contracts/blob/724d610f4f5d7bb9abf0b965a66cbf0ec809953b/src/infra/GoatRewardPool.sol#L396-L403

The comment and calculations in the `_earned` function of the `GoatRewardPool.sol` contract appear to be inconsistent with each other regarding the decimal precision of the reward tokens. While the comment states that `rewardPerTokenStored` is in 18 decimals, the calculations use a fixed divisor of `1e30`, which suggests a different decimal precision. This inconsistency may lead to incorrect reward calculations.

```solidity
    /// @param rewardPerTokenStored Stored reward value per staked token in 18 decimals
    /// @param userRewardPerTokenPaid Stored reward value per staked token in 18 decimals at the
    /// last time a user was paid the reward
```

```solidity
    function _rewardPerToken(address _reward) private view returns (uint256 rewardPerToken) {
        RewardInfo storage rewardData = _getRewardInfo(_reward);
        if (totalSupply() == 0) {
            rewardPerToken = rewardData.rewardPerTokenStored;
        } else {
            rewardPerToken = rewardData.rewardPerTokenStored + Math.mulDiv(
                (_lastTimeRewardApplicable(rewardData.periodFinish) - rewardData.lastUpdateTime),
                rewardData.rate * 1e30,
                totalSupply()
            );
        }
    }

    /// @dev Calculate the reward amount earned by the user
    /// @param _user Address of the user
    /// @param _reward Address of the reward
    /// @return earnedAmount Amount of reward earned by the user
    function _earned(address _user, address _reward) private view returns (uint256 earnedAmount) {
        RewardInfo storage rewardData = _getRewardInfo(_reward);
        earnedAmount = rewardData.earned[_user] + Math.mulDiv(
            balanceOf(_user), 
            (_rewardPerToken(_reward) - rewardData.userRewardPerTokenPaid[_user]),
            1e30
        );
    }
```

### Recommendation

Ensure that comments accurately reflect the decimal precision of the reward tokens.


## [I-01] Use Ownable2Step instead of Ownable

### Description

https://github.com/goatfi/contracts/blob/724d610f4f5d7bb9abf0b965a66cbf0ec809953b/src/infra/GoatRewardPool.sol#L6

The `GoatRewardPool.sol` contract inherits from OpenZeppelin's `Ownable.sol` contract, which implements a single-step ownership transfer pattern. So if the owner intends to transfer ownership to a new address but provides an incorrect address, there's no built-in mechanism to rectify this mistake within the contract. As a result, ownership of the contract may become effectively lost, and functionalities that are restricted to the owner may become inaccessible.

### Recommendation

Use OpenZeppelin's `Ownable2Step.sol` instead of `Ownable.sol`.

## [I-02] Unnecessary initialization of constant in constructor

### Description

https://github.com/goatfi/contracts/blob/724d610f4f5d7bb9abf0b965a66cbf0ec809953b/src/infra/GoatRewardPool.sol#L106

In the constructor of the `GoatRewardPool.sol` contract, the variable `rewardMax` is set to a value of 10. However, this value is not changed anywhere else in the contract. Initializing a constant value in the constructor is unnecessary and can be optimized.

```solidity
    constructor(address _stakedToken) ERC20("Staked GOA", "stGOA") Ownable(msg.sender) { // @audit-info 2 step
        stakedToken = IERC20(_stakedToken);
        rewardMax = 10;
    }
```

### Recommendation

Consider declaring `rewardMax` as an internal constant directly in the contract, rather than initializing it in the constructor. 
