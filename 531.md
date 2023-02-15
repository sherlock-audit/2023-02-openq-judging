Jeiwan

medium

# Claiming ongoing bounty can be replayed, causing double-spending of rewards

## Summary
Due to a bug or a manipulation, the oracle may call the `ClaimManagerV1.claimBounty` function multiple times with the same parameters. Each such call can result it an unplanned payment to a bounty receiver.
## Vulnerability Detail
The [OngoingBountyV1](https://github.com/sherlock-audit/2023-02-openq/blob/main/contracts/Bounty/Implementations/OngoingBountyV1.sol#L11) contract is designed to multiple payments, to one or multiple contributors. The payments are triggered by a call to the [ClaimManagerV1.claimBounty](https://github.com/sherlock-audit/2023-02-openq/blob/main/contracts/ClaimManager/Implementations/ClaimManagerV1.sol#L31) function. It's only the oracle who can call the function, and the oracle is a trusted party that's programmed to react to off-chain events (e.g. GitHub commits or PR comments). Due to this fact, the oracle can be manipulated to repeat the same action multiple times and send an identical transaction multiple times.

When this happens with a `ClaimManagerV1.claimBounty` call for an `OngoingBountyV1` contract, the same bounty can be paid multiple times: claim IDs are not unique in `OngoingBountyV1`. A claim ID is a [hash of the claimant's ID and the address of the claimed token](https://github.com/sherlock-audit/2023-02-openq/blob/main/contracts/Bounty/Implementations/OngoingBountyV1.sol#L229-L234). During [claiming](https://github.com/sherlock-audit/2023-02-openq/blob/main/contracts/Bounty/Implementations/OngoingBountyV1.sol#L96), there's no check for whether a claim ID has already been used or not:
```solidity
bytes32 _claimId = generateClaimId(claimant, claimantAsset);

claimId[_claimId] = true;
claimIds.push(_claimId);

_transferToken(payoutTokenAddress, payoutVolume, _payoutAddress);
```
## Impact
The oracle can claim bounty multiple times for a contributor of an `OngoingBountyV1` bounty contract using the same parameters. In case the oracle is manipulated to trigger an identical transaction multiple times, funds may ne stolen from the bounty contract.

While the vulnerability may cause a loss of funds for the protocol, I report it as a medium funding because exploiting it requires manipulating the oracle, which is high difficulty, in my view.
## Code Snippet
[ClaimManagerV1.sol#L31](https://github.com/sherlock-audit/2023-02-openq/blob/main/contracts/ClaimManager/Implementations/ClaimManagerV1.sol#L31)
[OngoingBountyV1.sol#L96](https://github.com/sherlock-audit/2023-02-openq/blob/main/contracts/Bounty/Implementations/OngoingBountyV1.sol#L96)
## Tool used
Manual Review
## Recommendation
Consider making claim IDs unique in the `OngoingBountyV1` contract (e.g. adding the length of the [claimIds](https://github.com/sherlock-audit/2023-02-openq/blob/main/contracts/Bounty/Implementations/OngoingBountyV1.sol#L108) state variable, or adding a unique per-claimant index, to the `generateClaimId` function).