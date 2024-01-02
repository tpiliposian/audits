## [H-01] Potential Bid Manipulation in AuctionDemo Contract

### Lines of Code

https://github.com/code-423n4/2023-10-nextgen/blob/8b518196629faa37eae39736837b24926fd3c07c/smart-contracts/AuctionDemo.sol#L124-L143

### Vulnerability details

In the provided contract, there is a potential attack vector where a bidder can place a high bid to discourage others and then cancel their bid later to win the auction at a lower cost.

### Impact

The impact of this finding is that the `AuctionDemo.sol` contract could be susceptible to bid manipulation, potentially allowing a bidder to win an auction with a lower bid than initially placed. This issue could undermine the fairness and integrity of the auction system.

The initial lower bidder could win the auction with an exceptionally lower bid, which may not represent the true value of the item being auctioned. This could lead to unfair outcomes and manipulation of the auction results.

### Proof of Concept

https://github.com/code-423n4/2023-10-nextgen/blob/8b518196629faa37eae39736837b24926fd3c07c/smart-contracts/AuctionDemo.sol#L124-L143

Here is the vulnerable code segment in the AuctionDemo contract:

```solidity
    function cancelBid(uint256 _tokenid, uint256 index) public {
        require(block.timestamp <= minter.getAuctionEndTime(_tokenid), "Auction ended");
        require(auctionInfoData[_tokenid][index].bidder == msg.sender && auctionInfoData[_tokenid][index].status == true);
        auctionInfoData[_tokenid][index].status = false;
        (bool success, ) = payable(auctionInfoData[_tokenid][index].bidder).call{value: auctionInfoData[_tokenid][index].bid}("");
        emit CancelBid(msg.sender, _tokenid, index, success, auctionInfoData[_tokenid][index].bid);
    }
```

Attack Scenario:

1. An attacker initially places a lower bid.
2. An attacker then places a very high bid, so that nobody will agree to pay more, and effectively discourage others.
3. Other potential bidders are discouraged by the high bid.
4. An attacker cancels their bid before the auction ends and wins with the initially placed lower bid.

### Tools Used

Manual review.

### Recommended Mitigation Steps

Consider implementing one or more of the following mitigation steps:

Implement a bid lock-in period during which bidders cannot cancel their bids. This will prevent last-minute bid cancellations and potential manipulation.
Enforce a minimum bid increment, which requires each new bid to be a certain percentage higher than the previous one. This discourages artificially low initial bids.
Introduce a fee for canceling bids, especially if the cancellation occurs after a certain point in the auction. This discourages malicious behavior and ensures commitment from bidders.
