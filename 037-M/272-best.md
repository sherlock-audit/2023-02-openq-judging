0x52

medium

# setPayoutSchedule shouldn't be callable after a competition is closed

## Summary

Tiered bounties are only closed when the first winner claims from the contract. This means that by definition, that if a tiered bounty is closed then it has already made a payout. Funding is snapshot when a bounty is closed. If the payout schedule is adjusted after the contest has closed it can lead to the contract being unable to fund all payouts.

## Vulnerability Detail

https://github.com/sherlock-audit/2023-02-openq/blob/main/contracts/ClaimManager/Implementations/ClaimManagerV1.sol#L215-L228

When a tiered bounty is claimed for the first time it will close the bounty. _claimTieredPercentageBounty is the only function that can close a tiered bounty, which means if a tiered bounty is closed then it has already made a payout.

https://github.com/sherlock-audit/2023-02-openq/blob/main/contracts/Bounty/Implementations/TieredPercentageBountyV1.sol#L123-L136

When a tiered bounty is closed, the balances of each payout token is snapshot. If a tier that has already been paid out is changed then the funds available and the funds still owed winners will become out of sync. The result is that the contract may become insolvent. If paying out in an ERC20 token then any shortfall can be fixed by donating to the contract. If the token paid out is MATIC then the shortfall cannot be fixed because the receive function on every bounty type is disabled.

## Impact

Calling setPayoutSchedule after a tiered bounty is closed can lead to contract insolvency

## Code Snippet

https://github.com/sherlock-audit/2023-02-openq/blob/main/contracts/Bounty/Implementations/TieredPercentageBountyV1.sol#L141-L179

## Tool used

Manual Review

## Recommendation

Only allow the payout schedule to be set for percentage tiered bounties if they are still open