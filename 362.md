0x52

high

# Adversary can lock every deposit forever by making a deposit with _expiration = type(uint256).max

## Summary

DepositMangerV1 allows the caller to specify _expiration which specifies how long the deposit is locked. An adversary can specify a deposit with _expiration = type(uint256).max which will cause an overflow in the BountyCore#getLockedFunds subcall and permanently break refunds.

## Vulnerability Detail

https://github.com/sherlock-audit/2023-02-openq/blob/main/contracts/DepositManager/Implementations/DepositManagerV1.sol#L54-L56

DepositManagerV1#fundBountyToken allows the depositor to specify an _expiration which is passed directly to BountyCore#receiveFunds.

https://github.com/sherlock-audit/2023-02-openq/blob/main/contracts/Bounty/Implementations/BountyCore.sol#L47-L52

BountyCore stores the _expiration in the expiration mapping.

https://github.com/sherlock-audit/2023-02-openq/blob/main/contracts/Bounty/Implementations/BountyCore.sol#L339-L349

When requesting a refund, getLockedFunds returns the amount of funds currently locked. The line to focus on is `depositTime[depList[i]] + expiration[depList[i]]`

An adversary can cause `getLockedFunds` to always revert by making a deposit in which `depositTime[depList[i]] + expiration[depList[i]] > type(uint256).max` causing an overflow. To exploit this the user would make a deposit with `_expiration = type(uint256).max` which will cause a guaranteed overflow. This causes DepositMangerV1#refundDeposit to always revert breaking all refunds.

Submitting as high risk because when combined with payout breaking methods it will result in all deposited tokens being stuck forever.

## Impact

Adversary can permanently break refunds

## Code Snippet

https://github.com/sherlock-audit/2023-02-openq/blob/main/contracts/DepositManager/Implementations/DepositManagerV1.sol#L152-L195

## Tool used

Manual Review

## Recommendation

Add the following check to DepositMangerV1#fundBountyToken:

    +   require(_expiration <= type(uint128).max)