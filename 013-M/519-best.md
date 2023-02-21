Jeiwan

high

# Bounty minter can rugpull contest winners by changing winners and payout amounts after contest results were announced

## Summary
Bounty minters can change contest winners at any time by calling the `setTierWinner` function. A malicious bounty minter can unset contest winners after they were announces and before they claimed their rewards. Or a malicious bounty minter can remove some contest winners from the list after one of them has claimed a reward.

Bounty minters can also change payout amounts at any time by calling the `setPayoutSchedule` and `setPayoutScheduleFixed` functions. A malicious bounty minter can change payout amounts after contest results were announced and before winners claimed their rewards.
## Vulnerability Detail
The [OpenQV1.setTierWinner](https://github.com/sherlock-audit/2023-02-openq/blob/main/contracts/OpenQ/Implementations/OpenQV1.sol#L96) and [TieredBountyCore.setTierWinner](https://github.com/sherlock-audit/2023-02-openq/blob/main/contracts/Bounty/Implementations/TieredBountyCore.sol#L59) functions are not restricted by contest stages. A bounty minter can change contest winners at any time: before contest results were announces, after they were announces, or after one of the winners have claimed their reward.

Also, the [OpenQV1.setPayoutSchedule](https://github.com/sherlock-audit/2023-02-openq/blob/main/contracts/OpenQ/Implementations/OpenQV1.sol#L281), [TieredPercentageBountyV1.setPayoutSchedule](https://github.com/sherlock-audit/2023-02-openq/blob/main/contracts/Bounty/Implementations/TieredPercentageBountyV1.sol#L141) and [OpenQV1.setPayoutScheduleFixed](https://github.com/sherlock-audit/2023-02-openq/blob/main/contracts/OpenQ/Implementations/OpenQV1.sol#L305), [TieredFixedBountyV1.setPayoutScheduleFixed](https://github.com/sherlock-audit/2023-02-openq/blob/main/contracts/Bounty/Implementations/TieredFixedBountyV1.sol#L138) functions are not restricted by contest stages as well. A malicious bounty minter can change payout amounts at any stage of a contest, e.g. after announcing winners and before winners have claimed their rewards.

In the current implementation, there's no protection for contest winners: a malicious bounty minter can announce a contest, receive value from contest participants and winners, and leave them without a reward, or reduce their reward. While bounty minters are privileged users (they create and set rules of bounties), they shouldn't be allowed to rugpull other people by using the OpenQ protocol.
## Impact
Contest winners will not receive rewards, or will receive reduced rewards, if a malicious bounty minter changes the list of winners or payout amounts after the winners have contributed.
## Code Snippet
[OpenQV1.setTierWinner](https://github.com/sherlock-audit/2023-02-openq/blob/main/contracts/OpenQ/Implementations/OpenQV1.sol#L96)
[TieredBountyCore.setTierWinner](https://github.com/sherlock-audit/2023-02-openq/blob/main/contracts/Bounty/Implementations/TieredBountyCore.sol#L59)
## Tool used
Manual Review
## Recommendation
Taking into account that calling `setTierWinner` and setting tier winners is done only after (and, ideally, right after) contest results were announced, consider disallowing changing contest winners and payout schedules after winners have been announced.