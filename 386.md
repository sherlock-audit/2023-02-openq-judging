HonorLt

medium

# Closed bounty is considered claimable

## Summary
When the bounty type is not found, it is considered claimable if the status is closed.

## Vulnerability Detail
When deciding if the bounty is claimable if the type is not among the known definitions, it returns true if the status is 1:
```solidity
    function bountyIsClaimable(address _bountyAddress)
        public
        view
        returns (bool)
    {
        IBounty bounty = IBounty(payable(_bountyAddress));

        uint256 status = bounty.status();
        uint256 _bountyType = bounty.bountyType();

        if (
            _bountyType == OpenQDefinitions.ATOMIC ||
            _bountyType == OpenQDefinitions.ONGOING ||
            _bountyType == OpenQDefinitions.TIERED_PERCENTAGE ||
            _bountyType == OpenQDefinitions.TIERED_FIXED
        ) {
            return status == 0;
        } else {
            return status == 1;
        }
    }
```

1 means the bounty is closed, thus a bounty with an unknown type is considered claimable if it is closed. I believe this is a wrong assumption. Also, the name of the function is a bit misleading because it actually indicates if the bounty is open, while the reader might think it indicates if you can claim the rewards.

## Impact
The claim manager will trick the outside readers and if a new bounty type is introduced in the future, this function will return an inverted result.

## Code Snippet

https://github.com/sherlock-audit/2023-02-openq/blob/main/contracts/ClaimManager/Implementations/ClaimManagerV1.sol#L355-L364

## Tool used

Manual Review

## Recommendation
It should throw an error (revert with `UNKNOWN_BOUNTY TYPE`) in the else block.
