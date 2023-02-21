rvierdiiev

medium

# Protocol doesn't provide gaps for upgrades. As result memory of proxy can be corrupted after upgrade.

## Summary
Protocol doesn't provide gaps for upgrades. As result memory of proxy can be corrupted after upgrade.
## Vulnerability Detail
Protocol deploys a lot of contracts under the proxy, because it's possible that some contract will be upgraded.
The problem is that some contract that are going to be upgradeable extends another contracts that do not have storage gap for future variable's update. As result in case if smth will be added to that parent contracts, memory of proxy can be corrupted.

Let's check first example.
There are few contracts, such as OpenQV1, ClaimManagerStorageV1 that extend Oraclize. Oraclize contract doesn't have any storage gap, so in case if new variable will be added it will corrupt proxy memory.

Another example.
BountyCore is extended by 3 contracts: `TieredBountyCore`, `OngoingBountyV1`, `AtomicBountyV1` and each of them has own storage contract. But BountyStorageCore doesn't have gap. So in case if BountyCore storage contract will be introduced with new variable, it will overwrite memory for the proxy.
`
## Impact
In case of upgrade with new variable in parent contract, memory of proxy can be corrupted.
## Code Snippet
https://github.com/sherlock-audit/2023-02-openq/blob/main/contracts/ClaimManager/Storage/ClaimManagerStorage.sol#L21
https://github.com/sherlock-audit/2023-02-openq/blob/main/contracts/Oracle/Oraclize.sol#L11
https://github.com/sherlock-audit/2023-02-openq/blob/main/contracts/Bounty/Storage/BountyStorageCore.sol#L24
https://github.com/sherlock-audit/2023-02-openq/blob/main/contracts/Bounty/Storage/AtomicBountyStorage.sol#L10
## Tool used

Manual Review

## Recommendation
Add gap into all storage contracts that are extended by child contracts.