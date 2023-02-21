ltyu

high

# Partial refunds can cause the rest of the deposit to be lost

## Summary
Partial funds that are refunded will cause the rest of the funds to be lost.

## Vulnerability Detail
In `refundDeposit()` of DepositManagerV1.sol, deposit refunds are calculated using the unlocked/available funds. If the deposit amount is greater than the unlocked funds, whatever amount is unlocked will be sent to the depositor.

Afterwards, the deposit is marked as `refunded`. This is problematic because this amount can be less than what was deposited, and since the deposit is considered `refunded`, the rest of the funds will be irretrievable by the depositor.

## Impact
Loss of depositor funds if attempting to refund

## Code Snippet
the `volume` to be sent is calculated here
https://github.com/sherlock-audit/2023-02-openq/blob/main/contracts/DepositManager/Implementations/DepositManagerV1.sol#L152-L181

The deposit is marked as `refunded`. A revert will occur when attempting to refund again.
https://github.com/sherlock-audit/2023-02-openq/blob/main/contracts/Bounty/Implementations/BountyCore.sol#L64-L76
## Tool used

Manual Review

## Recommendation
- Consider only refunding full amounts
- If a partial refund is needed, consider tracking the amount left to refund and subtracting from it after each partial refund.
