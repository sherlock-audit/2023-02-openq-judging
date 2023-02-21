unforgiven

high

# double reward payment in the OnGoingBounty if oracle calls the claim function twice because code don't check value of claimId[claimId]

## Summary
To transfer winner funds In Ongoing bounty, oracle calls claim function for the bounty and function `OnGoingBounty.claimOngoingPayout()` transfers funds and set the value of the `claimId[claimId]` to true. but there is no logic in the code to make sure `claimId[claimId]` is false and if off-chain oracle calls claim multiple for a user(user may initiate them off-chain) then that user would receive funds multiple time.

## Vulnerability Detail
This is function `_claimOngoingBounty()` code in ClaimManagerV1:
```solidity
    function _claimOngoingBounty(
        IOngoingBounty _bounty,
        address _closer,
        bytes calldata _closerData
    ) internal {
        _eligibleToClaimOngoingBounty(_bounty, _closer, _closerData);

        (address tokenAddress, uint256 volume) = _bounty.claimOngoingPayout(
            _closer,
            _closerData
        );
.............
    }
```
As you can see code calls `_eligibleToClaimOngoingBounty()` to runs all required statements to determine if the claimant can claim an ongoing bounty payout and there is no check for `claimId[claimId]` in the code and the code calls `bounty.claimOngoingPayout()`.
This is `_eligibleToClaimOngoingBounty()` code:
```solidity
    function _eligibleToClaimOngoingBounty(
        IOngoingBounty bounty,
        address _closer,
        bytes memory _closerData
    ) internal view {
        require(
            bounty.status() == OpenQDefinitions.OPEN,
            Errors.CONTRACT_IS_NOT_CLAIMABLE
        );

        (, string memory claimant, , string memory claimantAsset) = abi.decode(
            _closerData,
            (address, string, address, string)
        );

        bytes32 claimId = bounty.generateClaimId(claimant, claimantAsset);

        if (bounty.invoiceRequired()) {
            require(
                bounty.invoiceComplete(claimId),
                Errors.INVOICE_NOT_COMPLETE
            );
        }

        if (bounty.supportingDocumentsRequired()) {
            require(
                bounty.supportingDocumentsComplete(claimId),
                Errors.SUPPORTING_DOCS_NOT_COMPLETE
            );
        }

        if (bounty.kycRequired()) {
            require(hasKYC(_closer), Errors.ADDRESS_LACKS_KYC);
        }
    }
}
```
As you can see there is no check that `claimId[claimId]` is not set before and winner prize didn't claimed before.
this is `claimOngoingPayout()` code in OngoingBountyV1 contract:
```solidity
    function claimOngoingPayout(
        address _payoutAddress,
        bytes calldata _closerData
    ) external onlyClaimManager nonReentrant returns (address, uint256) {
        (, string memory claimant, , string memory claimantAsset) = abi.decode(
            _closerData,
            (address, string, address, string)
        );

        bytes32 _claimId = generateClaimId(claimant, claimantAsset);

        claimId[_claimId] = true;
        claimIds.push(_claimId);

        _transferToken(payoutTokenAddress, payoutVolume, _payoutAddress);
        return (payoutTokenAddress, payoutVolume);
    }
```
As you can see there is no check that ` claimId[_claimId] == False` here too.
So if oracle calls claim for a winner multiple times then the code would send the win multiple times. these are the steps that cause the issue:
1. User1 creates and ongoing bounty Bount1 and transfer 100K USDT with the prize of the 50K USDT.
2. User2 and User3 are the winner of bounty and User1 merge their PR.
3. off-chain oracle calls claim for User2 and code would send the funds to user1.
4. User2 initiate a claim function for himself in the off chain oracle program(or oracle send the transaction by mistake) and oracle calls the claim for User1 and because code don't check that claimId is not distributed before so it would send funds again.
5. now User3 can't claim his funds.

The issue won't happen in the other Bounty type because they check and make sure funds are not send to winner in the past but this check is not in the execution flow of the ongoing bounty claim.

## Impact
double payment and wrong funds distribution between users. attacker can still other winners funds.

## Code Snippet
https://github.com/sherlock-audit/2023-02-openq/blob/main/contracts/ClaimManager/Implementations/ClaimManagerV1.sol#L173-L197

https://github.com/sherlock-audit/2023-02-openq/blob/main/contracts/ClaimManager/Implementations/ClaimManagerV1.sol#L456-L490

https://github.com/sherlock-audit/2023-02-openq/blob/main/contracts/Bounty/Implementations/OngoingBountyV1.sol#L96-L112

## Tool used
Manual Review

## Recommendation
when claiming ongoing bounty make sure `claimId[claimId] == false`