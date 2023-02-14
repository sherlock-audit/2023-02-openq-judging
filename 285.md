unforgiven

high

# malicious winner in the TieredPercentageBounty can steal other winner funds by inflating contract token balance right before claiming and then refunding his deposit

## Summary
In TieredPercentageBounty code calculates prize amounts based on `fundingTotals[token]` which is contract token balance when bounty is closed. a malicious winner can inflate contract token balance before closing by depositing huge amount for short time and then call permissioned claim and code would close the bounty and transfer more amount to winner and the he would refund his deposit, by doing this malicious winner can withdraw other winner funds.

## Vulnerability Detail
This is `_claimTieredPercentageBounty()` code in ClaimManagerV1 contract:
```solidity
    function _claimTieredPercentageBounty(
        IBounty _bounty,
        address _closer,
        bytes calldata _closerData
    ) internal {
        (, , , , uint256 _tier) = abi.decode(
            _closerData,
            (address, string, address, string, uint256)
        );

        _eligibleToClaimTier(_bounty, _tier, _closer);

        if (_bounty.status() == 0) {
            _bounty.closeCompetition();
..............
        for (uint256 i = 0; i < _bounty.getTokenAddresses().length; i++) {
            uint256 volume = _bounty.claimTiered(
                _closer,
                _tier,
                _bounty.getTokenAddresses()[i]
            );
................
        }

        for (uint256 i = 0; i < _bounty.getNftDeposits().length; i++) {
            bytes32 _depositId = _bounty.nftDeposits(i);
            if (_bounty.tier(_depositId) == _tier) {
                _bounty.claimNft(_closer, _depositId);
..........
            }
        }

        _bounty.setTierClaimed(_tier);
    }
```
As you can see when bounty is open code calls `bounty.closeCompetition()` to close the bounty. and then code loops through deposit tokens in the bounty and calls `bounty.claimTiered(tierID, tokenAddress)` to transfer winner funds.
This is the `closeCompetition()` code in TieredPercentageBounty  contract which records contract's current token balances in the `fundingTotals[]` array.
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
This is `claimTiered()` code in TieredPercentageBountyV1 contract:
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
as you can see code calculates winner prize amount based on `fundingTotals[token]` and tier share of the bounty prize. a malicious winner can use this and inflate the value of the `fundingTotals[token]` by depositing before the time that bounty close and then call claim and receive more funds and then refund his tokens. these are the steps attacker would perform:
1. UserA creates Bounty1 from type TieredPercentageBounty and deposit 20K USDT into the Bounty1 for 3 month.
2. after 3 month UserA would set User1, User2 as tier winners in the Bounty1 and set tiers 1 and 2 to receive 50% of the prize.
3. User1 is malicious actor and would deposit 10K USDT to the Bounty1 with short expiration time and now the Bounty1 USDT balance would be 20K.
4. then User1 would call `ClaimManagerV1.permissionedClaimTieredBounty()` to withdraw his wining prize.
5. code would call `_claimTieredPercentageBounty()` to send winner prize and because bounty is still open code would first call `bounty.closeCompetition()` to close the bounty and function `closeCompetition()` would set the value of the `fundingTotals[USDT] = 20K`.
6. then execution would reach `claimTiered(User1, 1, USDT)` and the prize amount would calculate as `20K * 50% = 10K` and 10K USDT would be transferred to User1.
7. now User1 would call refund and would refund his 10K USDT funds. now User1 received all the 10K USDT and User2 didn't receive anything.

## Impact
Malicious winner can steal other winners funds.

## Code Snippet
https://github.com/sherlock-audit/2023-02-openq/blob/main/contracts/ClaimManager/Implementations/ClaimManagerV1.sol#L203-L272

https://github.com/sherlock-audit/2023-02-openq/blob/main/contracts/Bounty/Implementations/TieredPercentageBountyV1.sol#L104-L120

https://github.com/sherlock-audit/2023-02-openq/blob/main/contracts/Bounty/Implementations/TieredPercentageBountyV1.sol#L123-L136

## Tool used
Manual Review

## Recommendation
prevent funds from refunding for some time when contract is closed.