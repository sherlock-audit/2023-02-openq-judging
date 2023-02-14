yixxas

high

# Claimed amount of tokens will be computed incorrectly if rebasing token is used

## Summary
Protocol wants to support all kinds of tokens, including rebasing tokens as noted in Sherlock's On-chain context guidelines. If rebasing tokens are used as payoutTokens for TieredPercentageBounty, users may either underclaim or overclaim dependant on whether token rebased up or down. One of the more popular rebasing token is stETH, with a 5 billion market cap currently.

## Vulnerability Detail
The first time a claim is made in a bounty, `closeCompetition()` is called to prevent any further funding of the bounty contract. `closeCompetition()` keeps track of the total balances of each token in the `fundingTotals[]` array at this point in time. However, claims are not all made in the same timestamp. A rebasing token can make subsequent claims that are made to be computed wrongly.

```solidity
function closeCompetition() external onlyClaimManager {
	require(
		status == OpenQDefinitions.OPEN,
		Errors.CONTRACT_ALREADY_CLOSED
	);

	status = OpenQDefinitions.CLOSED;
	bountyClosedTime = block.timestamp;

	for (uint256 i = 0; i < getTokenAddresses().length; i++) {
		address _tokenAddress = getTokenAddresses()[i];
		fundingTotals[_tokenAddress] = getTokenBalance(_tokenAddress);
	}
}
```

`claimedBalance()` is calculated in this way. It uses the snapshot of the balance of the token and take a percentage based on `payoutSchedule`. Because rebasing token balances can adjust up or down, but `fundingTotals[]` remain constant, the amount claimed by users will be more than expected if token rebases down, and less than expected if token rebases up.

```solidity
uint256 claimedBalance = (payoutSchedule[_tier] *
            fundingTotals[_tokenAddress]) / 100;
```


## Impact
User will either overclaim, which results in some other user not being able to claim their rightful amount, or underclaim tokens themselves.

## Code Snippet
https://github.com/sherlock-audit/2023-02-openq/blob/main/contracts/Bounty/Implementations/TieredPercentageBountyV1.sol#L116
https://github.com/sherlock-audit/2023-02-openq/blob/main/contracts/Bounty/Implementations/TieredPercentageBountyV1.sol#L134

## Tool used

Manual Review

## Recommendation
In order to add support for rebasing tokens, consider checking the new balance right before a claim is made

