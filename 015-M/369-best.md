yixxas

medium

# If a refund is made but contract does not have enough tokens, refunder is shortchanged

## Summary
Refunds that are made on a deposit can only be done once on a particular deposit id. If contract is currently not able to refund the correct amount, calling `refundDeposit()` will jeopardise refunder as they will claim less than what they should be able to.

## Vulnerability Detail
Deposits can be refunded once expiration is reached. Refunds are made based on a particular `_depositId`. Each `_depositId` can only be refunded once.

```solidity
function refundDeposit(address _bountyAddress, bytes32 _depositId)
	external
	onlyProxy
{
	IBounty bounty = IBounty(payable(_bountyAddress));

	require(
		bounty.funder(_depositId) == msg.sender,
		Errors.CALLER_NOT_FUNDER
	);

	require(
		block.timestamp >=
			bounty.depositTime(_depositId) + bounty.expiration(_depositId),
		Errors.PREMATURE_REFUND_REQUEST
	);

	address depToken = bounty.tokenAddress(_depositId);

	uint256 availableFunds = bounty.getTokenBalance(depToken) -
		bounty.getLockedFunds(depToken);

	uint256 volume;
	if (bounty.volume(_depositId) <= availableFunds) {
		volume = bounty.volume(_depositId);
	} else {
		volume = availableFunds;
	}

	bounty.refundDeposit(_depositId, msg.sender, volume);
	...

}
```

We see that if the volume attached to the `_depositId` is more than what is available in the contract, volume is then set to what is available, and the refund amount will be made based on that. This means that refunds in such cases will be partial, or even 0 and this `depositId` can no longer be refunded again. 

If at some point in time later the bounty contract has enough funds again, it will be too late for refunder as they will not be able to refund the remaining amount on the same `_depositId`.

## Impact
Refunder gets a refund of less than what they should get, and cannot claim the remaining balance.

## Code Snippet
https://github.com/sherlock-audit/2023-02-openq/blob/main/contracts/DepositManager/Implementations/DepositManagerV1.sol#L174-L179

## Tool used

Manual Review

## Recommendation
Consider allowing refunds to be made only in full. 

