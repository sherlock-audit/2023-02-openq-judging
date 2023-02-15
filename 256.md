0x52

medium

# When ongoing bounty is closed all deposits are still locked until expiration

## Summary

Ongoing Bounties can be closed but all deposits will still be stuck until they hit their expiration time. When an ongoing bounty is closed it is no longer possible to be claimed but deposits will still be stuck until expiration.

## Vulnerability Detail

https://github.com/sherlock-audit/2023-02-openq/blob/main/contracts/OpenQ/Implementations/OpenQV1.sol#L328-L351

OpenQV1#closeOngoing allows the creator of the bounty to close their ongoing bounty.

https://github.com/sherlock-audit/2023-02-openq/blob/main/contracts/ClaimManager/Implementations/ClaimManagerV1.sol#L456-L465

Once an ongoing bounty has been closed, it can no longer be claimed. Since it can't be claimed anymore the bounty won't pay out any more funds. Deposits are still stuck until they expire. This needlessly locks the funds that can never be used. 

## Impact

Funds can be needlessly locked after ongoing bounties have been closed

## Code Snippet

https://github.com/sherlock-audit/2023-02-openq/blob/main/contracts/DepositManager/Implementations/DepositManagerV1.sol#L152-L195

## Tool used

Manual Review

## Recommendation

Bypass the expiration check in DepositManagerV1#refundDeposit for ongoing bounties if the bounty is closed.