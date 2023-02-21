nicobevi

medium

# [Medium] - Funds locked if bounty is funded with both ether and erc20 tokens at the same time

## Summary
When a bounty is created. Next step is to fund such bounty with the rewards by the creator. To do that, a call to `fundBountyToken()` in [DepositManagerV1.sol](https://github.com/sherlock-audit/2023-02-openq/blob/main/contracts/DepositManager/Implementations/DepositManagerV1.sol#L36-L74) must be done. This function allows both kind of funds, native tokens (eth) sending value to the transaction, or any allowed token through `_tokenAddress` (previously whitelisted by the protocol).
Internally, the function `receiveFunds()` is called for that particular bounty.

## Vulnerability Detail
If for some reason the bounty creator calls this method sending both ether and a valid `_tokenAddress` then the ether sent will be lost.
In the `recieveFunds()` function body we find this condition:
```solidity
        uint256 volumeReceived;
        if (_tokenAddress == address(0)) {
            volumeReceived = msg.value;
        } else {
            volumeReceived = _receiveERC20(_tokenAddress, _funder, _volume);
        }
```

In our case, `_tokenAddress` is a valid token so the if condition is not fullfilled and the volume received will be only the token amount. Locking the ether since the deposit will be recorded just for the tokens sent.

There's a function `getLockedFunds()` that looks like a way to get such locked ether. But because these funds are not being recorded as part of a deposit, this function won't work.

## Impact
Ether locked on the bounty contract.

## Code Snippet
[DepositManagerV1.sol#L36-L74](https://github.com/sherlock-audit/2023-02-openq/blob/main/contracts/DepositManager/Implementations/DepositManagerV1.sol#L36-L74)

[BountyCore.sol#L21-L58](https://github.com/sherlock-audit/2023-02-openq/blob/main/contracts/Bounty/Implementations/BountyCore.sol#L21-L58)


## Tool used

Manual Review

## Recommendation
The recommendation is to validate that, in case that a `_tokenAddress` is sent, then there is no ether sent too.

```diff
        uint256 volumeReceived;
        if (_tokenAddress == address(0)) {
            volumeReceived = msg.value;
        } else {
+            if (msg.value != 0) {
+                  revert EtherSent();
+            }
            volumeReceived = _receiveERC20(_tokenAddress, _funder, _volume);
        }
```