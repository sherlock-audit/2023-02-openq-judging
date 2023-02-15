0x52

high

# Adversary can break any bounty they wish by depositing an NFT then refunding it

## Summary

Refunding an NFT doesn't remove the nftDeposit so the contract will try to payout an NFT it doesn't have if the NFT has been refunded.

## Vulnerability Detail

https://github.com/sherlock-audit/2023-02-openq/blob/main/contracts/Bounty/Implementations/BountyCore.sol#L64-L93

All bounties use BountyCore#refundDeposit to process refunds to user. This simply transfers the NFT back to the funder but leaves the nftDeposit. 

https://github.com/sherlock-audit/2023-02-openq/blob/main/contracts/ClaimManager/Implementations/ClaimManagerV1.sol#L150-L165

When any bounty is claimed it loops through nftDeposits and attempts to transfer an NFT for each one. The problem is that if an NFT has been refunded the deposit receipt will still exist but the contract won't have the NFT. The result is that all payouts will be permanently broken. 

Submitting as high risk because when combined with refund locking methods it will result in all deposited tokens being stuck forever.

## Impact

Adversary can permanently break payouts on any bounty

## Code Snippet

https://github.com/sherlock-audit/2023-02-openq/blob/main/contracts/ClaimManager/Implementations/ClaimManagerV1.sol#L31-L67

## Tool used

Manual Review

## Recommendation

Remove the deposit receipt from nftDeposits when refunding NFT deposits