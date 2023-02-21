cccz

high

# refundDeposit can be called after the bounty is closed, which prevents the winner of the TieredPercentageBounty from claiming the prize

## Summary
If refundDeposit is called after TieredPercentageBounty is closed, it may make the contract balance less than fundingTotals, thus preventing the winner from claiming the prize
## Vulnerability Detail
When TieredPercentageBounty is closed, the balance of the contract is used as fundingTotals in the closeCompetition function.
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
In TieredPercentageBounty.claimTiered, the prize will be calculated based on fundingTotals.
```solidity
    function claimTiered(
        address _payoutAddress,
        uint256 _tier,
        address _tokenAddress
    ) external onlyClaimManager nonReentrant returns (uint256) {
        require(
            bountyType == OpenQDefinitions.TIERED_PERCENTAGE,
            Errors.NOT_A_TIERED_BOUNTY
        );
        require(!tierClaimed[_tier], Errors.TIER_ALREADY_CLAIMED);

        uint256 claimedBalance = (payoutSchedule[_tier] *
            fundingTotals[_tokenAddress]) / 100;

        _transferToken(_tokenAddress, claimedBalance, _payoutAddress);
        return claimedBalance;
    }
```
If refundDeposit is called after the closeCompetition, the contract balance will be less than fundingTotals and the winner will not be able to claim the prize because the contract balance is not enough.
```solidity
    function refundDeposit(
        bytes32 _depositId,
        address _funder,
        uint256 _volume
    ) external virtual onlyDepositManager nonReentrant {
        require(!refunded[_depositId], Errors.DEPOSIT_ALREADY_REFUNDED);
        require(funder[_depositId] == _funder, Errors.CALLER_NOT_FUNDER);
        require(
            block.timestamp >= depositTime[_depositId] + expiration[_depositId],
            Errors.PREMATURE_REFUND_REQUEST
        );

        refunded[_depositId] = true;

        if (tokenAddress[_depositId] == address(0)) {
            _transferProtocolToken(funder[_depositId], _volume);
        } else if (isNFT[_depositId]) {
            _transferNft(
                tokenAddress[_depositId],
                funder[_depositId],
                tokenId[_depositId]
            );
        } else {
            _transferERC20(
                tokenAddress[_depositId],
                funder[_depositId],
                _volume
            );
        }
    }
```
## Impact

## Code Snippet
https://github.com/sherlock-audit/2023-02-openq/blob/main/contracts/Bounty/Implementations/TieredPercentageBountyV1.sol#L123-L136
https://github.com/sherlock-audit/2023-02-openq/blob/main/contracts/Bounty/Implementations/TieredPercentageBountyV1.sol#L104-L120
https://github.com/sherlock-audit/2023-02-openq/blob/main/contracts/Bounty/Implementations/BountyCore.sol#L64-L93
## Tool used

Manual Review

## Recommendation
Consider allowing refundDeposit to be called only before the bounty is closed