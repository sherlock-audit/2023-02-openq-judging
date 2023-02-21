yixxas

medium

# Funding of bounty tokens for `onGoingBounty` and `TieredFixedBounty` should be limited to only `payoutTokenAddress`

## Summary
Both `onGoingBounty` and `TieredFixedBounty` only allows the claiming of one type of ERC20 token, as dictated by `payoutTokenAddress`. It is also the only token that claimant can claim. However, funders can still fund the contract with other tokens with `fundBountyToken()`, resulting in such tokens being left in contract. 

## Vulnerability Detail

In both `claimOngoingPayout()` and `claimTieredFixed()`, both claimable tokens are only limited to `payoutTokenAddress`. This is decided at the specific bounty contract level.

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

```solidity
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
```

However, when tokens are funded, `receiveFunds()` is used, and this is implemented in the bountyCore contract inherited by all bounties. Hence, it is possible for funders to fund tokens that are not being used by onGoingBounty and TieredFixedBounty, resulting in such tokens being lost.

## Impact
Tokens that are not used by the bounties can still be funded into the contract, resulting in lost assets.

## Code Snippet
https://github.com/sherlock-audit/2023-02-openq/blob/main/contracts/Bounty/Implementations/OngoingBountyV1.sol#L96-L112
https://github.com/sherlock-audit/2023-02-openq/blob/main/contracts/Bounty/Implementations/TieredFixedBountyV1.sol#L91-L107

## Tool used

Manual Review

## Recommendation
Consider preventing funders to fund tokens to the ongoing bounty and tiered fixed bounty if token being funded is not `payoutTokenAddress`.
