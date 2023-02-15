ak1

medium

# `_claimTieredPercentageBounty` and `_claimTieredFixedBounty` should update the `setTierClaimed` inside the `claimTiered` `claimTieredFixed`

## Summary

In the current code flow, the tierClaimed[_tier] is updated after completing the transactions.

Refer the function [_claimTieredPercentageBounty ](https://github.com/sherlock-audit/2023-02-openq/blob/main/contracts/ClaimManager/Implementations/ClaimManagerV1.sol#L203-L272)and [_claimTieredFixedBounty](https://github.com/sherlock-audit/2023-02-openq/blob/main/contracts/ClaimManager/Implementations/ClaimManagerV1.sol#L278-L341)

when we look at the flow .`_claimTieredFixedBounty` -> `claimTieredFixed`

    function claimTieredFixed(address _payoutAddress, uint256 _tier)
        external
        onlyClaimManager
        nonReentrant
        returns (uint256)
    {
        require(
            bountyType == OpenQDefinitions.TIERED_FIXED,
            Errors.NOT_A_TIERED_FIXED_BOUNTY
        );
        require(!tierClaimed[_tier], Errors.TIER_ALREADY_CLAIMED);


        uint256 claimedBalance = payoutSchedule[_tier];


        _transferToken(payoutTokenAddress, claimedBalance, _payoutAddress);
        return claimedBalance;
    }

The check `require(!tierClaimed[_tier], Errors.TIER_ALREADY_CLAIMED);` is done inside the `claimTieredFixed` but the status is updated inside the `_claimTieredFixedBounty`.

The same is done to `_claimTieredPercentageBounty` 

## Vulnerability Detail
Refer the summary section
## Impact

I do see the `nonReentrant` in the `claimTieredFixed`, but still would like to flag to the team about this logic.
It is giving control to external user who can do anything.

## Code Snippet

https://github.com/sherlock-audit/2023-02-openq/blob/main/contracts/ClaimManager/Implementations/ClaimManagerV1.sol#L278-L341

https://github.com/sherlock-audit/2023-02-openq/blob/main/contracts/Bounty/Implementations/TieredFixedBountyV1.sol#L91-L107

## Tool used

Manual Review

## Recommendation

we suggest to update the `tierClaimed[_tier]` inside the inside the `claimTieredFixed` call itself. the same goes to percentage based tier claim.


    function claimTieredFixed(address _payoutAddress, uint256 _tier)
        external
        onlyClaimManager
        nonReentrant
        returns (uint256)
    {
        require(
            bountyType == OpenQDefinitions.TIERED_FIXED,
            Errors.NOT_A_TIERED_FIXED_BOUNTY
        );
        require(!tierClaimed[_tier], Errors.TIER_ALREADY_CLAIMED);

        tierClaimed[_tier] = true; -----------------------> audit update.        

        uint256 claimedBalance = payoutSchedule[_tier];


        _transferToken(payoutTokenAddress, claimedBalance, _payoutAddress);
        return claimedBalance;
    }
