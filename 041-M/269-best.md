cccz

medium

# _claimOngoingBounty does not verify that _closer is associated with the claimant

## Summary
_claimOngoingBounty does not verify that _closer is associated with the claimant 
## Vulnerability Detail
OngoingBountyV1.claimOngoingPayout will use the claimant to calculate the _claimId and set claimId[_claimId] = true to indicate that the claimant has already claimed the prize.
```solidity
    function claimOngoingPayout(
        address _payoutAddress,
        bytes calldata _closerData
    ) external onlyClaimManager nonReentrant returns (address, uint256) {
        (, string memory claimant, , string memory claimantAsset) = abi.decode(
            _closerData,
            (address, string, address, string)
        );

        bytes32 _claimId = generateClaimId(claimant, claimantAsset);

        claimId[_claimId] = true;
        claimIds.push(_claimId);

        _transferToken(payoutTokenAddress, payoutVolume, _payoutAddress);
        return (payoutTokenAddress, payoutVolume);
    }
```
But before ClaimManagerV1._claimOngoingBounty calls OngoingBountyV1.claimOngoingPayout, it does not verify that _closer is associated with the claimant, which allows closer to provide any claimant to claim the prize.

Considering that _claimOngoingBounty needs to be called by oracle, it should be a mid-risk.

## Impact
It allows closer to provide any claimant to claim the prize in Ongoing Bounty.
## Code Snippet
https://github.com/sherlock-audit/2023-02-openq/blob/main/contracts/Bounty/Implementations/OngoingBountyV1.sol#L96-L112
## Tool used

Manual Review

## Recommendation
Consider verifying that _closer is associated with the claimant before calling OngoingBountyV1.claimOngoingPayout