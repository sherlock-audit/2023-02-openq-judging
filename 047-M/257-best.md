0x52

high

# Refunding logic with multiple deposits is first mover take all

## Summary

There is no accounting to keep track of which deposits are being used to pay bounties. This means that after all deposits have expired, the first user to refund their deposit will get more funds.

## Vulnerability Detail

https://github.com/sherlock-audit/2023-02-openq/blob/main/contracts/DepositManager/Implementations/DepositManagerV1.sol#L171-L179

Available funds are simply calculated by taking the current balance of the bounty contract and subtracting the locked balance. This works well for single deposits but doesn't when there are multiple deposits from multiple funders. When this occurs it will cause a first mover take all situation.

Example:
User A and User B both makes 10,000 USDC deposits to an Ongoing bounty, bringing the total funds of the bounty to 20,000 USDC. After some time the bounty is closed. By the time it's closed, it has paid out 10,000 USDC in awards leaving it with 10,000 USDC. User A and B both wait until their deposits are expired. Now User A requests a refund from his deposit. This refunds all 10,000 USDC to User A leaving none for User B. As a result, User A has effectively forced User B to pay all the awards.

## Impact

Reward logic is first mover take all

## Code Snippet

https://github.com/sherlock-audit/2023-02-openq/blob/main/contracts/DepositManager/Implementations/DepositManagerV1.sol#L152-L195

## Tool used

Manual Review

## Recommendation

Bounties should track how much has been paid and how much has been funded. When an ongoing bounty is refunded it should even split remaining funds over depositors.