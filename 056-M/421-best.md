yixxas

high

# `getLockedFunds()` is calculating locked funds wrongly if there are claims made

## Summary
`getLockedFunds()` compute locked funds based on what is deposited into the contract by tracking the `volume[]` array of each depositId. The `volume` is based on what is deposited at deposit time. It is not taking into consideration the amount that has already been claimed by other users.

## Vulnerability Detail

`getLockedFunds()` goes through all deposits and sums up the volume for all deposits that has not reached expiration as they are considered locked. However, it does not take into account claims that have been made and rewarded to users. This will lead to an overcalculation of what is locked if some claims has been made. The expiration is only meant to prevent depositors from refunding early. Users can still claim irrespective of expiration time.

```solidity
function getLockedFunds(address _depositId)
	public
	view
	virtual
	returns (uint256)
{
	uint256 lockedFunds;
	bytes32[] memory depList = this.getDeposits();
	for (uint256 i = 0; i < depList.length; i++) {
		if (
			block.timestamp <
			depositTime[depList[i]] + expiration[depList[i]] &&
			tokenAddress[depList[i]] == _depositId
		) {
			lockedFunds += volume[depList[i]];
		}
	}

	return lockedFunds;
}
```


## Impact
An overcalculation of lockedFunds will likely prevent depositors from refunding their deposits, as `availableFunds` can and will underflow.

> `uint256 availableFunds = bounty.getTokenBalance(depToken) - bounty.getLockedFunds(depToken);`

## Code Snippet
https://github.com/sherlock-audit/2023-02-openq/blob/main/contracts/Bounty/Implementations/BountyCore.sol#L333-L353
https://github.com/sherlock-audit/2023-02-openq/blob/main/contracts/DepositManager/Implementations/DepositManagerV1.sol#L152-L195

## Tool used

Manual Review

## Recommendation
Consider accounting for claim amount in calculation of locked funds.
