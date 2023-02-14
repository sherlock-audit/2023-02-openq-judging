unforgiven

high

# attacker can cause calls to claim function of the all Bounty types (except Ongoing) to revert because code would try to transfer refunded NFT deposits

## Summary
function claimNft() in BountyCore would try to transfer deposited NFT without checking the refund status of the deposit. attacker can use this and cause any bounty of  all types (except Ongoing) to be unclaimable by making claim calls to revert because in claim code would loop through deposited NFT tokens and try to transfer them to winner and if one of deposited NFTs were refunded then the whole claim transaction would revert always.

## Vulnerability Detail
Function `claimBounty()` in ClaimManagerV1 is used for claiming winner prize from Bounty contract. it calls appropriate claim method based on bounty type. the specific type bounty claim function would loop through bounty deposited NFT tokens and call `claimNft()` function of the bounty contract for that deposited NFT token. 
This is `_claimAtomicBounty()` code which is called by `claimBounty()`: (other claim functions for bounty types are similar)
```solidity
    function _claimAtomicBounty(
        IAtomicBounty _bounty,
        address _closer,
        bytes calldata _closerData
    ) internal {
        _eligibleToClaimAtomicBounty(_bounty, _closer);

        for (uint256 i = 0; i < _bounty.getTokenAddresses().length; i++) {
            uint256 volume = _bounty.claimBalance(
                _closer,
                _bounty.getTokenAddresses()[i]
            );
            ................
        }

        for (uint256 i = 0; i < _bounty.getNftDeposits().length; i++) {
            _bounty.claimNft(_closer, _bounty.nftDeposits(i));
            ................
        }
    }
```
As you can see it loops through bounty deposited NFT tokens and calls `bounty.claimNft()`. This is `claimNft()` code in BountyCoreV1 contract:
```solidity
    function claimNft(address _payoutAddress, bytes32 _depositId)
        external
        virtual
        onlyClaimManager
        nonReentrant
    {
        _transferNft(
            tokenAddress[_depositId],
            _payoutAddress,
            tokenId[_depositId]
        );
    }
    
        function _transferNft(
        address _tokenAddress,
        address _payoutAddress,
        uint256 _tokenId
    ) internal virtual {
        IERC721Upgradeable nft = IERC721Upgradeable(_tokenAddress);
        nft.safeTransferFrom(address(this), _payoutAddress, _tokenId);
    }
```
As you can see it calls `_transferNft()` without checking that the deposit is not refunded and `_transferNft()` don't check that contract own the NFT token. so if one of the deposited NFT tokens were refunded then call to `claimNft()` for that refunded NFT deposit would revert and it would cause the whole claim to revert. as all Bounty types(except Ongoing) supports NFT deposit and claims, and the claim functions in the ClaimManagerV1 loops through the deposited NFTs and try to transfer them to winner so this issue exists for all the bounty types(except Ongoing) to exploit this attacker need to perform this:
1. NFT1 token is in the whitelist.
2. User1 creates a Bounty1 (atomicBounty) and deposits (id=10 NFT1) for 6 month.
3. after 3 month attacker deposits (id=20 NFT1) token for 1 day to Bounty1. code would add (id=20 NFT1) to Bounty1 NFT token deposits.
4. after 1 day attacker would refund his deposit and code would set the status of the deposit to refunded and would transfer the (id=20 NFT1) to attacker.
5. after 2 month(5 month from bounty start time) User1 would merge a PR and the winner of the bounty would be announced.
6. off-chain oracle would call claim function for the Bounty1 to transfer winner funds and claim function would try to loop through deposited NFT tokens and when it reaches the NFT1 it would try to transfer (id=20 NFT1) token and the transfer call would revert because contract don't own that token and whole oracle's transaction would revert.
7. even so winner performed his job he couldn't receive his prize.

the issue exists for other type of the bounty (in tiered bounty attacker need to deposit NFT for a specific tier and then refund and claims for that tier would revert)

## Impact
attacker can cause winners to not receive their prizes as claim function would always revert. attacker can perform this to every bounties if the type isn't Ongoing. this would cause fund lose for winner.

## Code Snippet
https://github.com/sherlock-audit/2023-02-openq/blob/main/contracts/ClaimManager/Implementations/ClaimManagerV1.sol#L150-L151

https://github.com/sherlock-audit/2023-02-openq/blob/main/contracts/ClaimManager/Implementations/ClaimManagerV1.sol#L251-L254

https://github.com/sherlock-audit/2023-02-openq/blob/main/contracts/ClaimManager/Implementations/ClaimManagerV1.sol#L320-L323

https://github.com/sherlock-audit/2023-02-openq/blob/main/contracts/Bounty/Implementations/BountyCore.sol#L125-L136

https://github.com/sherlock-audit/2023-02-openq/blob/main/contracts/Bounty/Implementations/BountyCore.sol#L257-L264

## Tool used
Manual Review

## Recommendation
check that NFT deposit is not refunded before calling transfer.
or check that contract owns the NFT before trying to transfer.