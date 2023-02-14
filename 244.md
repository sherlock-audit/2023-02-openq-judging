clems4ever

medium

# Resizing the payout schedule with less items might revert

## Summary

According to some comments in `setPayoutScheduleFixed`, reducing the number of items in the schedule is a supported use case. However in that case, the function will revert because we are iterating over as many items as there was in the previous version of the three arrays making the function revert since the new arrays have less items.

## Vulnerability Detail

Let say they were 4 items in the arrays `tierWinners`, `invoiceComplete` and `supportingDocumentsComplete` and we are resizing the schedule to 3 items. Then the following function would revert because we use the length of the previous arrays instead of the new ones in the for loops.

```solidity
function setPayoutScheduleFixed(
        uint256[] calldata _payoutSchedule,
        address _payoutTokenAddress
    ) external onlyOpenQ {
        require(
            bountyType == OpenQDefinitions.TIERED_FIXED,
            Errors.NOT_A_FIXED_TIERED_BOUNTY
        );
        payoutSchedule = _payoutSchedule;
        payoutTokenAddress = _payoutTokenAddress;

        // Resize metadata arrays and copy current members to new array
        // NOTE: If resizing to fewer tiers than previously, the final indexes will be removed
        string[] memory newTierWinners = new string[](payoutSchedule.length);
        bool[] memory newInvoiceComplete = new bool[](payoutSchedule.length);
        bool[] memory newSupportingDocumentsCompleted = new bool[](
            payoutSchedule.length
        );

        for (uint256 i = 0; i < tierWinners.length; i++) { <=====================================================
            newTierWinners[i] = tierWinners[i];
        }
        tierWinners = newTierWinners;

        for (uint256 i = 0; i < invoiceComplete.length; i++) { <=====================================================
            newInvoiceComplete[i] = invoiceComplete[i];
        }
        invoiceComplete = newInvoiceComplete;

        for (uint256 i = 0; i < supportingDocumentsComplete.length; i++) { <=====================================================
            newSupportingDocumentsCompleted[i] = supportingDocumentsComplete[i];
        }
        supportingDocumentsComplete = newSupportingDocumentsCompleted;
    }
```

The same issue exists on TieredPercentageBounty too.

## Impact

Unable to resize the payout schedule to less items than the previous state.

## Code Snippet

https://github.com/sherlock-audit/2023-02-openq/blob/main/contracts/Bounty/Implementations/TieredFixedBountyV1.sol#L157

## Tool used

Manual Review

## Recommendation

```solidity
        for (uint256 i = 0; i < newTierWinners.length; i++) {
            newTierWinners[i] = tierWinners[i];
        }
        tierWinners = newTierWinners;

        for (uint256 i = 0; i < newInvoiceComplete.length; i++) {
            newInvoiceComplete[i] = invoiceComplete[i];
        }
        invoiceComplete = newInvoiceComplete;

        for (uint256 i = 0; i < newSupportingDocumentsCompleted.length; i++) {
            newSupportingDocumentsCompleted[i] = supportingDocumentsComplete[i];
        }
        supportingDocumentsComplete = newSupportingDocumentsCompleted;
```
Note this won't work if increasing the number of items compared to previous state must also be supported. In that case you must use the length of the smallest of the two arrays in each for loop.