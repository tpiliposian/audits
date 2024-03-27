# [H-01] Attacker can bypass transfer restrictions in GameItems contract

## Lines of code

https://github.com/code-423n4/2024-02-ai-arena/blob/cd1a0e6d1b40168657d1aaee8223dc050e15f8cc/src/GameItems.sol#L289-L303

https://github.com/code-423n4/2024-02-ai-arena/blob/cd1a0e6d1b40168657d1aaee8223dc050e15f8cc/src/GameItems.sol#L204

## Vulnerability details

### Impact

In the `GameItems.sol` contract there is a potential security issue that could allow users to bypass transfer restrictions set by the contract. Specifically, users could exploit the `ERC1155`'s `safeBatchTransferFrom` function to transfer tokens in batches, potentially bypassing individual transfer restrictions imposed by the contract.

### Proof of Concept

The `GameItems.sol` contract implements transfer restrictions using a boolean flag to determine if a game item is transferable.
```solidity
    /// @param transferable Boolean of whether or not the game item can be transferred
```
However, the `ERC1155` standard's `safeBatchTransferFrom` function does not perform individual transferability checks for each token in the batch, allowing users to transfer tokens in batches regardless of their transferability status.
```solidity
     * @dev See {IERC1155-safeBatchTransferFrom}.
     */
    function safeBatchTransferFrom(
        address from,
        address to,
        uint256[] memory ids,
        uint256[] memory values,
        bytes memory data
    ) public virtual {
        address sender = _msgSender();
        if (from != sender && !isApprovedForAll(from, sender)) {
            revert ERC1155MissingApprovalForAll(sender, from);
        }
        _safeBatchTransferFrom(from, to, ids, values, data);
    }
```

### Tools Used

Manual review.

### Recommended Mitigation Steps

Consider overriding the `safeBatchTransferFrom` function to perform transferability checks for each token being transferred in the batch or disallow batch transferring.

# [M-01] Predictable On-Chain Randomness and Manipulation through fighters.length

## Lines of code

https://github.com/code-423n4/2024-02-ai-arena/blob/f2952187a8afc44ee6adc28769657717b498b7d4/src/FighterFarm.sol#L484-L531

https://github.com/code-423n4/2024-02-ai-arena/blob/f2952187a8afc44ee6adc28769657717b498b7d4/src/FighterFarm.sol#L307-L331

## Vulnerability details

### Impact

The vulnerability in the `FighterFarm` smart contract stems from its reliance on on-chain randomness, which can be predictable, and the susceptibility to manipulation through the `fighters.length` property. This combination allows users to potentially manipulate the creation process of fighters, leading to unfair advantages in the associated game or ecosystem.

### Proof of Concept

`mintFromMergingPool` function is calling `_createNewFighter` with these parameters:
```solidity
    function mintFromMergingPool(
        address to, 
        string calldata modelHash, 
        string calldata modelType, 
        uint256[2] calldata customAttributes
    ) 
        public 
    {
        require(msg.sender == _mergingPoolAddress);
        _createNewFighter(
            to, 
@>          uint256(keccak256(abi.encode(msg.sender, fighters.length))), 
            modelHash, 
            modelType,
            0,
            0,
            customAttributes
        );
    }
```
So the `dna` of the fighter is being calculated throughout the on-chain randomness, moreover using the `fighters.length`, which any user can change (increase) with creating new fighter:
```solidity
    function _createNewFighter(
        address to, 
        uint256 dna, 
        string memory modelHash,
        string memory modelType, 
        uint8 fighterType,
        uint8 iconsType,
        uint256[2] memory customAttributes
    ) 
        private 
    {  
        require(balanceOf(to) < MAX_FIGHTERS_ALLOWED);
        uint256 element; 
        uint256 weight;
        uint256 newDna;
        if (customAttributes[0] == 100) {
            (element, weight, newDna) = _createFighterBase(dna, fighterType);
        }
        else {
            element = customAttributes[0];
            weight = customAttributes[1];
            newDna = dna;
        }
        uint256 newId = fighters.length;

        bool dendroidBool = fighterType == 1;
        FighterOps.FighterPhysicalAttributes memory attrs = _aiArenaHelperInstance.createPhysicalAttributes(
            newDna,
            generation[fighterType],
            iconsType,
            dendroidBool
        );
@>      fighters.push(
            FighterOps.Fighter(
                weight,
                element,
                attrs,
                newId,
                modelHash,
                modelType,
                generation[fighterType],
                iconsType,
                dendroidBool
            )
        );
        _safeMint(to, newId);
        FighterOps.fighterCreatedEmitter(newId, weight, element, generation[fighterType]);
    }
```
By artificially increasing the length of the `fighters` array, an attacker can influence the randomization process used to generate attributes for newly created fighters. This manipulation allows the attacker to bias the creation of fighters in their favor, granting them unfair advantages within the game.

### Tools Used

Manual review.

### Recommended Mitigation Steps

Use off-chain randomness, e.g. Chainlink VRF.
