bin2chen

medium

# not support token: revert on Zero Value Transfers.

## Summary
Some token will revert when transfer 0,  claimBounty()  will fail

## Vulnerability Detail

Openq is committed to supporting more token, as written in the document:

```solidity
DEPLOYMENT: polygon mainnet
LANGUAGE: solidity 0.8.17
ERC20: any
ERC721: any
ERC777: any
FEE-ON-TRANSFER: any
REBASING TOKENS: any
ADMIN: restricted for upgrades, trusted for other admin functionality described below
EXTERNAL-ADMINS: oracle, claimManager, depositManager, openQ, issuer/minter of a bounty
```
But the current implementation does not support: Revert on Zero Value Transfers.

https://github.com/d-xo/weird-erc20/#revert-on-zero-value-transfers

The current transfer token code is implemented as follows:

```solidity
    function _transferERC20(
        address _tokenAddress,
        address _payoutAddress,
        uint256 _volume
    ) internal virtual {
        IERC20Upgradeable token = IERC20Upgradeable(_tokenAddress);
        token.safeTransfer(_payoutAddress, _volume);  //@audit <--------some token like: LEND ,when _volume ==0 will revert
    }
```

When will there be a transfer quantity equal to 0?

The most common situation is:
1. When the deposit is refunded, the token address still exists in the "tokenAddresses" array, so _bounty.claimBalance () will still be executed, but the balance is 0 at this time.
2. Type is TIERED_PERCENTAGE, and the total bonus is too small. After rounding, the bonus due is 0.

All of the above have been attacked by malicious users, such as depositing a token with a quantity of 1, or refunding it immediately after depositing it.

If this type of token in whitelist token, malicious user attacks will cause users to fail in claims forever.


Suggestion when _volume ==0, don't perform transfer.

## Impact

If this kind of token be add to whitelist , it will can attack, resulting in failure to claim bounty

## Code Snippet
https://github.com/sherlock-audit/2023-02-openq/blob/main/contracts/Bounty/Implementations/BountyCore.sol#L181-L191

https://github.com/sherlock-audit/2023-02-openq/blob/main/contracts/Bounty/Implementations/BountyCore.sol#L221-L228

## Tool used

Manual Review

## Recommendation

```solidity
    function _transferERC20(
        address _tokenAddress,
        address _payoutAddress,
        uint256 _volume
    ) internal virtual {
+      if (_volume==0) return;
        IERC20Upgradeable token = IERC20Upgradeable(_tokenAddress);
        token.safeTransfer(_payoutAddress, _volume);  //@audit <--------some token like: LEND ,when _volume ==0 will revert
    }
```
