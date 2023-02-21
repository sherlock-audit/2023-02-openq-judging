bin2chen

medium

# ZERO_VOLUME_SENT Restriction failure

## Summary

"VolumeReceived" is the real deposit value. "volumeReceived" should be checked, not "_volume".

## Vulnerability Detail

For many reasons, we often limit the deposit amount to not 0.
Such as (occupied token address quantity, misoperation, etc.)

receiveFunds() code:
```solidity
    function receiveFunds(
        address _funder,
        address _tokenAddress,
        uint256 _volume,
        uint256 _expiration
    )
        external
        payable
        virtual
        onlyDepositManager
        nonReentrant
        returns (bytes32, uint256)
    {
        require(_volume != 0, Errors.ZERO_VOLUME_SENT);   //@audit <----------------limit !=0
        require(_expiration > 0, Errors.EXPIRATION_NOT_GREATER_THAN_ZERO);
        require(status == OpenQDefinitions.OPEN, Errors.CONTRACT_IS_CLOSED);

        bytes32 depositId = _generateDepositId();

        uint256 volumeReceived;
        if (_tokenAddress == address(0)) {
            volumeReceived = msg.value;    //@<------- if eth, don't use _volume ,but don't check msg.value !=0
        } else {
            volumeReceived = _receiveERC20(_tokenAddress, _funder, _volume); //@<------- if transfer need fees , volumeReceived maybe 0
        }

        volume[depositId] = volumeReceived;

```

The above code is limited to 0. There are two problems:
1. if _tokenAddress==address(0), it is not checked whether msg.value is equal to 0.
2. If handling fee is required for transfer, it can still be 0.

suggest: check volumeReceived before setting. It cannot be 0.


## Impact

ZERO_VOLUME_SENT Restriction failure

## Code Snippet

https://github.com/sherlock-audit/2023-02-openq/blob/main/contracts/Bounty/Implementations/BountyCore.sol#L34-L46

## Tool used

Manual Review

## Recommendation
```solidity
    function receiveFunds(
        address _funder,
        address _tokenAddress,
        uint256 _volume,
        uint256 _expiration
    )
        external
        payable
        virtual
        onlyDepositManager
        nonReentrant
        returns (bytes32, uint256)
    {
-       require(_volume != 0, Errors.ZERO_VOLUME_SENT);
        require(_expiration > 0, Errors.EXPIRATION_NOT_GREATER_THAN_ZERO);
        require(status == OpenQDefinitions.OPEN, Errors.CONTRACT_IS_CLOSED);

        bytes32 depositId = _generateDepositId();

        uint256 volumeReceived;
        if (_tokenAddress == address(0)) {
            volumeReceived = msg.value;
        } else {
            volumeReceived = _receiveERC20(_tokenAddress, _funder, _volume);
        }
+       require(volumeReceived != 0, Errors.ZERO_VOLUME_SENT);
```
