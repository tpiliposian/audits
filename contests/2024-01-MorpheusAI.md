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

## Impact

This provides no safeguard, as the `block.timestamp` will reflect the value of the block in which the transaction is included. Consequently, malicious validators can indefinitely withhold the transaction.

## Tools Used

Manual review.

## Recommendations

Consider enabling users to specify a `deadline` parameter for their transactions




