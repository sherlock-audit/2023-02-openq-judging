clems4ever

high

# Token might become unrefundable because of arithmetic overflow in `getLockedFunds`

## Summary

To compute the volume that is to be refunded, the calculation depends on the balance available for the given token of the deposit and the amount locked. However, as malicious user, I can make a deposit that can make the refund function revert for all deposits of a given token by exploiting an arithmetic overflow on the expiration.

## Vulnerability Detail

Let say a legit user makes a funding deposit of 1000 USDC using the `fundBountyToken` function. Now a malicious user makes a funding deposit of 1 USDC with an expiration of 2^256-1.
This will be deposited as expected and the expiration will be recorded here 

```solidity
function receiveFunds(
        address _funder,
        address _tokenAddress,
        uint256 _volume,
        uint256 _expiration
    )
        external
        payable
        virtual
        onlyDepositManager
        nonReentrant
        returns (bytes32, uint256)
    {
        require(_volume != 0, Errors.ZERO_VOLUME_SENT);
        require(_expiration > 0, Errors.EXPIRATION_NOT_GREATER_THAN_ZERO);
        require(status == OpenQDefinitions.OPEN, Errors.CONTRACT_IS_CLOSED);

        bytes32 depositId = _generateDepositId();

        uint256 volumeReceived;
        if (_tokenAddress == address(0)) {
            volumeReceived = msg.value;
        } else {
            volumeReceived = _receiveERC20(_tokenAddress, _funder, _volume);
        }

        funder[depositId] = _funder;
        tokenAddress[depositId] = _tokenAddress;
        volume[depositId] = volumeReceived;
        depositTime[depositId] = block.timestamp;
        expiration[depositId] = _expiration; <=============================================
        isNFT[depositId] = false;

        deposits.push(depositId);
        tokenAddresses.add(_tokenAddress);

        return (depositId, volumeReceived);
    }
```

However, later on, when the first user tries to get a refund after the lock expiration of his 1000 USDC deposit, he would call `refundDeposit` which would call `getLockedFunds` but unfortunately this function will revert because of arithmetic overflow at the highlighted line because the function iterates on the malicious deposit to compute the overall locked funds for this token...

```solidity
function getLockedFunds(address _depositId)
        public
        view
        virtual
        returns (uint256)
    {
        uint256 lockedFunds;
        bytes32[] memory depList = this.getDeposits();
        for (uint256 i = 0; i < depList.length; i++) {
            if (
                block.timestamp <
                depositTime[depList[i]] + expiration[depList[i]] && <===========================================
                tokenAddress[depList[i]] == _depositId
            ) {
                lockedFunds += volume[depList[i]];
            }
        }

        return lockedFunds;
    }
```

This works both for ERC-20 and NFTs in the same collection because the contract still tries to compute the volume even if it does not make sense for NFTs.

## Impact

All deposits of the token which was supposed to be refundable after their expiration cannot be refunded anymore.

## Code Snippet

https://github.com/-clems4ev3r:sherlock-audit/2023-02-openq/blob/main/contracts/Bounty/Implementations/BountyCore.sol#L344

## Tool used

Manual Review

## Recommendation

Ideally, I would not make the calculation for a refund depend on other deposits but if you think it's unavoidable at least I would add a check at deposit time to make sure `depositTime + expiration` would not overflow.

Also filter out NFT deposits in `getLockedFunds` to reduce the attack surface.