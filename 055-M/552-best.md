HonorLt

medium

# Incorrect new expiration when extending expired deposit

## Summary
`extendDeposit` assigns the wrong new expiration time when the deposit has expired.

## Vulnerability Detail
`extendDeposit`  calculates the new expiration differently depending if the deposit has expired or not:
```solidity
    function extendDeposit(
        bytes32 _depositId,
        uint256 _seconds,
        address _funder
    ) external virtual onlyDepositManager nonReentrant returns (uint256) {
        ...
        if (
            block.timestamp > depositTime[_depositId] + expiration[_depositId]
        ) {
            expiration[_depositId] =
                block.timestamp -
                depositTime[_depositId] +
                _seconds;
        } else {
            expiration[_depositId] = expiration[_depositId] + _seconds;
        }

        return expiration[_depositId];
    }
```
As you can see, when it has expired, the new expiration time is calculated this way:
` block.timestamp - depositTime[_depositId] + _seconds;`
This basically translates to the time elapsed since the deposit plus new seconds.

I believe this calculation is wrong. It operates on the elapsed time with no anchor to any specific timestamp. To illustrate why I think this is wrong, let's see this example:
1) I deposited now with an expiration of _5_ seconds. The initial deposit expiration is _now + 5s_, where now is a UNIX timestamp. _1676471088_ at the time of writing, so my expiration is _1676471093_.
2) After _10_ seconds (UNIX timestamp _1676471098_), I decided to extend my deposit by _30_ seconds. The current algorithm will calculate this new expiration timestamp:
_1676471998_ - _1676471088_ + _30_ = _940_
Clearly, this is a wrong timestamp. It has already passed.

## Impact

Now it is practically impossible to extend an expired deposit.

## Code Snippet

https://github.com/sherlock-audit/2023-02-openq/blob/main/contracts/Bounty/Implementations/BountyCore.sol#L108-L114

## Tool used

Manual Review

## Recommendation
I think a simple solution would be to continue from the current timestamp when the deposit is expired: 
`expiration[_depositId] = block.timestamp + _seconds`.
Another more precise option would be allowing users to specify not the seconds but the end timestamp.
