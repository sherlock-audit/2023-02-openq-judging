0x52

high

# Adversary can brick bounty payouts by calling fundBountyToken but funding it with an ERC721 token instead

## Summary

recieveFunds is only meant to be called with an ERC20 token but _receiveERC20 is generic enough to work with an ERC721 token. An adversary could call this with an ERC721 token to add it as a bounty reward. The problem is that the payout functions would completely break when trying to send it as a payout. The result is that afterwards the bounty would be completely broken.

## Vulnerability Detail

https://github.com/sherlock-audit/2023-02-openq/blob/main/contracts/Bounty/Implementations/BountyCore.sol#L197-L209

BountyCore#_receiveERC20 makes two calls to the underlying token contract. The first is the transferFrom method which exists functions identical to the ERC20 variant using token id instead of amount.

 https://github.com/sherlock-audit/2023-02-openq/blob/main/contracts/Bounty/Implementations/BountyCore.sol#L291-L299

The other call that's made is balanceOf which is also present in ERC721

https://github.com/sherlock-audit/2023-02-openq/blob/main/contracts/Bounty/Implementations/BountyCore.sol#L221-L228

On the flipside, when withdrawing and ERC20 it uses the transfer method which doesn't exist in ERC721. The result is that ERC721 tokens can be deposited as ERC20 tokens but can't be withdrawn. Since the contract will revert when trying to payout the ER721 token all payouts from the bounty will no longer work.

Submitting as high risk because when combined with refund locking methods it will result in all deposited tokens being stuck forever.

## Impact

Bounty will be permanently unclaimable

## Code Snippet

## Tool used

Manual Review

## Recommendation

Split the whitelist into an NFT whitelist and an ERC20 whitelist, to prevent a whitelisted NFT being deposited as an ERC20 token.