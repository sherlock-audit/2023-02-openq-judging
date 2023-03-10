clems4ever

medium

# Refund expiration can be bypassed

## Summary

There is no mechanism to prevent a user from setting a very short expiration meaning that anyone could invest in a github issue and can get a refund whenever they want. This could particularly be bad if the refund is made right before the development is done and shipped (like using a front running on a reward claiming transaction for instance). That would have incentivized the dev to work on the PR but he or she would eventually get paid much less, if anything, than what they initially expected.

## Vulnerability Detail

The expiration cannot be 0 but can be 1 meaning that the funds would be locked for a really short time and then the funders can get a refund at any time. This is because there is no minimum expiration value for a refund besides the checks that are highlighted in the snippets below:

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
        require(_expiration > 0, Errors.EXPIRATION_NOT_GREATER_THAN_ZERO); <===================================
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
The next snippet is actually the same for all bounties. I don't repeat it to keep the issue small.
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
        require(_expiration > 0, Errors.EXPIRATION_NOT_GREATER_THAN_ZERO); <====================================
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
        expiration[depositId] = _expiration;
        isNFT[depositId] = false;

        deposits.push(depositId);
        tokenAddresses.add(_tokenAddress);

        return (depositId, volumeReceived);
    }
```

## Impact

A funder can invest in a PR without actually locking the funds more than one second, meaning that a developer cannot know what would be the payout before it actually happens (because funders can just get a refund right before the dev is paid using front-running). This is, in my opinion, disincentivizing developers to take on an issue because, even when knowing how much time it would take him or her to develop a feature, he or she would not know how much they can certainly gain given this amount of time invested.

## Code Snippet

https://github.com/sherlock-audit/2023-02-openq/blob/main/contracts/Bounty/Implementations/BountyCore.sol#L35
https://github.com/sherlock-audit/2023-02-openq/blob/main/contracts/Bounty/Implementations/AtomicBountyV1.sol#L136

## Tool used

Manual Review

## Recommendation

Why not allow the issuer to set a minimum expiration period for all deposits of a given bounty?