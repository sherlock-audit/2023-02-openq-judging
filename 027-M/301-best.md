yixxas

medium

# `solvent()` cannot be used on tiered bounty

## Summary
As documented, `solvent()` should be usable to determine if a bounty has enough funds to cover payouts. However, the current implementation does not allow it to be used for tiered bounty.

> /// @notice Determines whether or not an ongoing bounty or tiered bounty have enough funds to cover payouts

## Vulnerability Detail

`solvent()` calls `bounty.payoutVolume()`. `payoutVolume` is however a variable only implemented in OngoingBountyStorage.sol. Only ongoing bounty will be able to utilise this function, contrary to what is documented. 

```solidity
function solvent(string calldata _bountyId) external view returns (bool) {
	IBounty bounty = getBounty(_bountyId);

	uint256 balance = bounty.getTokenBalance(bounty.payoutTokenAddress());
	return balance >= bounty.payoutVolume();
}
```

## Impact
`solvent()` cannot be used on tiered bounty

## Code Snippet
https://github.com/sherlock-audit/2023-02-openq/blob/main/contracts/OpenQ/Implementations/OpenQV1.sol#L408-L413

## Tool used

Manual Review

## Recommendation
Consider implementing payoutVolume for tiered bounty if we want to check its solvency with `solvent()`.
