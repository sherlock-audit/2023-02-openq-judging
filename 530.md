Jeiwan

medium

# Non-whitelisted tokens cannot be added if the limit of token addresses is filled with whitelisted ones

## Summary
Non-whitelisted tokens cannot be deposited to a bounty contract if too many whitelisted contracts were deposited.
## Vulnerability Detail
The [DepositManagerV1.fundBountyToken](https://github.com/sherlock-audit/2023-02-openq/blob/main/contracts/DepositManager/Implementations/DepositManagerV1.sol#L36) function allows depositing both whitelisted and non-whitelisted tokens by implementing the following check:
1. if a token is whitelisted, it [can be deposited without restrictions](https://github.com/sherlock-audit/2023-02-openq/blob/main/contracts/DepositManager/Implementations/DepositManagerV1.sol#L45);
1. if a token is not whitelisted, it [cannot be deposited if `openQTokenWhitelist.TOKEN_ADDRESS_LIMIT` tokens have already been deposited](https://github.com/sherlock-audit/2023-02-openq/blob/main/contracts/DepositManager/Implementations/DepositManagerV1.sol#L46-L49).

However, while the token addresses limit requirement is only applied to non-whitelisted tokens, whitelisted tokens also increase the counter of token addresses: both non-whitelisted and whitelisted token addresses are [added to the `tokenAddresses` set](https://github.com/sherlock-audit/2023-02-openq/blob/main/contracts/Bounty/Implementations/BountyCore.sol#L55).
## Impact
Bounty minters may not be able to deposit non-whitelisted tokens after they have deposited multiple whitelisted ones.
## Code Snippet
[DepositManagerV1.sol#L45-L50](https://github.com/sherlock-audit/2023-02-openq/blob/main/contracts/DepositManager/Implementations/DepositManagerV1.sol#L45-L50)
[BountyCore.sol#L326-L328](https://github.com/sherlock-audit/2023-02-openq/blob/main/contracts/Bounty/Implementations/BountyCore.sol#L326-L328)
[BountyCore.sol#L55](https://github.com/sherlock-audit/2023-02-openq/blob/main/contracts/Bounty/Implementations/BountyCore.sol#L55)
## Tool used
Manual Review
## Recommendation
Consider excluding whitelisted token addresses when checking the number of deposited tokens against the limit.