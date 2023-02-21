rvierdiiev

high

# OpenQV1.closeOngoing allows issuer to avoid payment

## Summary
OpenQV1.closeOngoing allows issuer to avoid payment
## Vulnerability Detail
It's possible to claim ongoing bounty only when bounty [is not closed](https://github.com/sherlock-audit/2023-02-openq/blob/main/contracts/ClaimManager/Implementations/ClaimManagerV1.sol#L461-L464). Once bounty is closed, then noone can claim paymnet, but all depositors are able to [refund their deposits](https://github.com/sherlock-audit/2023-02-openq/blob/main/contracts/DepositManager/Implementations/DepositManagerV1.sol#L152-L195).

`OpenQV1.closeOngoing` allows issuer to [close OngoingBountyV1 bounty](https://github.com/sherlock-audit/2023-02-openq/blob/main/contracts/OpenQ/Implementations/OpenQV1.sol#L328-L351) at any time he wishes, which allows him to get job done for free.
1.Issuer creates OngoingBountyV1 and deposits big amount of funds.
2.User do needed job.
3.Before claiming issuer calls OpenQV1.closeOngoing and as result claimer can't claim.
4.Issuer received job done for free.

As the systen is designed to allow any people to work together, i believe that this is high severity issue as it gives advantages to 1 side and makes it possible to not pay(which means steal of claimers funds).
## Impact
Issuer do not pay claimers.
## Code Snippet
Provided above.
## Tool used

Manual Review

## Recommendation
Think about some another states of bounty, not only open closed. And think about transitions from one state to another. So for example in case if state will be `in progress`, then it's not possible to close bounty.