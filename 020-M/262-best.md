0x52

medium

# Adversary can break NFT distribution by depositing up to max then refunding all of them

## Summary

Bounties limit the number of NFT deposits to five. An adversary can block adding NFTs by repeatedly depositing and withdrawing an NFT.

## Vulnerability Detail

https://github.com/sherlock-audit/2023-02-openq/blob/main/contracts/Bounty/Implementations/BountyCore.sol#L64-L93

All bounties use BountyCore#refundDeposit to process refunds to user. This simply transfers the NFT back to the funder but leaves the nftDeposit. This uses up the deposit limit which is current set to 5. Since the deposit cap is used up by deposits that have been refunded the slots can't be used to distribute legitimate NFTs to the bounty claimant. 

## Impact

Adversary can block legitimate NFT distribution

## Code Snippet

https://github.com/sherlock-audit/2023-02-openq/blob/main/contracts/Bounty/Implementations/BountyCore.sol#L64-L93

## Tool used

Manual Review

## Recommendation

When an NFT deposit is refunded it should remove the depositID from nftDeposits