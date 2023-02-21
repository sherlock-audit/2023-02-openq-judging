tsvetanovv

medium

# A transfer of an NFT need approve

## Summary
A transfer of an NFT clears its approvals. This is required by the standard.
`_receiveNft()` function in [BountyCore.sol](https://github.com/sherlock-audit/2023-02-openq/blob/main/contracts/Bounty/Implementations/BountyCore.sol#L244) no implementation for approve. 

```solidity
244: function _receiveNft(
        address _tokenAddress,
        address _sender,
        uint256 _tokenId
    ) internal virtual {
        IERC721Upgradeable nft = IERC721Upgradeable(_tokenAddress);
        nft.safeTransferFrom(_sender, address(this), _tokenId);
    }
```


## Vulnerability Detail
[OngoingBountyV1.sol](https://github.com/sherlock-audit/2023-02-openq/blob/main/contracts/Bounty/Implementations/OngoingBountyV1.sol#L133) and [TieredBountyCore.sol](https://github.com/sherlock-audit/2023-02-openq/blob/main/contracts/Bounty/Implementations/TieredBountyCore.sol#L18) 
call `_receiveNft()` function in [BountyCore.sol](https://github.com/sherlock-audit/2023-02-openq/blob/main/contracts/Bounty/Implementations/BountyCore.sol#L244) but nowhere is there approve.

The OpenZeppelin documentation describes `approve(address to, uint256 tokenId)` as:
“Gives permission to transfer tokenId token to another account. The approval is cleared when the token is transferred.”
`approve()` grants permission to transfer a token. You need to implement approve pattern in these contracts.


## Impact
See Summary and Vulnerability Detail

## Code Snippet
[OngoingBountyV1.sol](https://github.com/sherlock-audit/2023-02-openq/blob/main/contracts/Bounty/Implementations/OngoingBountyV1.sol#L133)
```solidity
function receiveNft(
        address _sender,
        address _tokenAddress,
        uint256 _tokenId,
        uint256 _expiration,
        bytes calldata
    ) external onlyDepositManager nonReentrant returns (bytes32) {
        require(
            nftDeposits.length < nftDepositLimit,
            Errors.NFT_DEPOSIT_LIMIT_REACHED
        );
        require(_expiration > 0, Errors.EXPIRATION_NOT_GREATER_THAN_ZERO);
        _receiveNft(_tokenAddress, _sender, _tokenId);

        bytes32 depositId = _generateDepositId();

        funder[depositId] = _sender;
        tokenAddress[depositId] = _tokenAddress;
        depositTime[depositId] = block.timestamp;
        tokenId[depositId] = _tokenId;
        expiration[depositId] = _expiration;
        isNFT[depositId] = true;

        deposits.push(depositId);
        nftDeposits.push(depositId);

        return depositId;
    }
```
[TieredBountyCore.sol](https://github.com/sherlock-audit/2023-02-openq/blob/main/contracts/Bounty/Implementations/TieredBountyCore.sol#L18)
```solidity
function receiveNft(
        address _sender,
        address _tokenAddress,
        uint256 _tokenId,
        uint256 _expiration,
        bytes calldata _data
    ) external override onlyDepositManager nonReentrant returns (bytes32) {
        require(
            nftDeposits.length < nftDepositLimit,
            Errors.NFT_DEPOSIT_LIMIT_REACHED
        );
        require(_expiration > 0, Errors.EXPIRATION_NOT_GREATER_THAN_ZERO);
        _receiveNft(_tokenAddress, _sender, _tokenId);

        bytes32 depositId = _generateDepositId();

        funder[depositId] = _sender;
        tokenAddress[depositId] = _tokenAddress;
        depositTime[depositId] = block.timestamp;
        tokenId[depositId] = _tokenId;
        expiration[depositId] = _expiration;
        isNFT[depositId] = true;

        uint256 _tier = abi.decode(_data, (uint256));
        tier[depositId] = _tier;

        deposits.push(depositId);
        nftDeposits.push(depositId);

        return depositId;
    }
```


## Tool used

Manual Review

## Recommendation
Implement approve pattern in these contracts.