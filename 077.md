carrot

high

# Refunds can be bricked by triggering OOG (out of gas) in DepositManager

## Summary
The `DepositManager` contract is in charge of refunding tokens from the individual bounties. This function ends up running a for loop over an unbounded array. This array can be made to be sufficiently large to exceed the block gas limit and cause out-of-gas errors and stop the processing of any refunds.
## Vulnerability Detail
The function `refundDeposit()` in `DepositManager.sol` is responsible for handling refunds, through the following snippet of code,
https://github.com/sherlock-audit/2023-02-openq/blob/main/contracts/DepositManager/Implementations/DepositManagerV1.sol#L152-L181
We are here interested in the line 
https://github.com/sherlock-audit/2023-02-openq/blob/main/contracts/DepositManager/Implementations/DepositManagerV1.sol#L171-L172
which calculates available funds. If we check the function `getLockedFunds()`, we see it run a for loop
https://github.com/sherlock-audit/2023-02-openq/blob/main/contracts/Bounty/Implementations/BountyCore.sol#L333-L352
This loop is running over the list of ALL deposits. The deposit list is unbounded, since there are no checks for such limits in the `receiveFunds()` function. This can result in a very long list, causing out-of-gas errors when making refund calls.
## Impact
Inability to withdraw funds. Can be forever locked into the contract.
## Code Snippet
https://github.com/sherlock-audit/2023-02-openq/blob/main/contracts/Bounty/Implementations/BountyCore.sol#L333-L352
## Tool used
Manual Review

## Recommendation
Put a bound on the function `receiveFunds` to limit the number of deposits allowed.