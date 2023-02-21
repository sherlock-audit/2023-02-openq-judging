yixxas

medium

# `invoiceCompleteClaimIds[]` and `supportingDocumentsCompleteClaimIds[]` can contain claimIds that are incomplete

## Summary
`invoiceCompleteClaimIds[]` and `supportingDocumentsCompleteClaimIds[]` are used to hold the claimIds that has been marked as complete.

However, it is possible for these arrays to contain claimIds that are incomplete and there is no way to remove them.

## Vulnerability Detail
`setInvoiceComplete()` and `setSupportingDocumentsComplete()` can be used to set a `claimId` to either true or false. The true/false value is stored in `invoiceComplete[]` and `supportingDocumentsComplete[]` respectively. 

However, regardless of whether it is being set to true or false, the `claimId` is being pushed into the `invoiceCompleteClaimIds[]` and `supportingDocumentsCompleteClaimIds[]` array. 

This means that these 2 arrays can not only hold repeated claimIds, it can also hold claimIds that is set to false.

## Impact
While these 2 arrays are only used as view functions, their contained values are not reliable and 3rd party apps that can potentially cause issues for 3rd party apps that interacts with OpenQ.

## Code Snippet
https://github.com/sherlock-audit/2023-02-openq/blob/main/contracts/Bounty/Implementations/OngoingBountyV1.sol#L176-L183
https://github.com/sherlock-audit/2023-02-openq/blob/main/contracts/Bounty/Implementations/OngoingBountyV1.sol#L188-L198

## Tool used

Manual Review

## Recommendation
If there is ever a need to set a previously completed invoice or document to be false, possibly due to it being set by mistake in the first place, we should have a way to prune the value from `supportingDocumentsComplete[]` and `invoiceComplete[]` arrays.
