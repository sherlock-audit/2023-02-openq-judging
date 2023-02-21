carrot

medium

# `BountyCore.sol` `receiveFunds` function does not check zero msg.value

## Summary
The `receiveFunds` function in contract `BountyCore` checks if incoming ERC20 tokens have 0 volume, but does not do the same check on `msg.value`. This allows malicious users to make 0 value deposits for free and inflate the size of the deposit list, costing users more gas when looping through deposits (ex. when calling getLockedFunds)
## Vulnerability Detail
The `receiveFunds` function does a 0 volume check for ERC20 tokens
https://github.com/sherlock-audit/2023-02-openq/blob/main/contracts/Bounty/Implementations/BountyCore.sol#L21-L58
However, if `_tokenAddress` is set to 0, the contract does not check if the corresponding msg.value is greater than 0. This lets attackers spam empty deposits for free, which increases the size of the deposits array. All functions which loop the deposits array will end up costing large amounts of gas for users. 
## Impact
Large gas costs due to overloaded deposits array
## Code Snippet
Empty contributions can be sent with the following test snippet
```javascript
it('accepts empty contributions', async () => {
    // ARRANGE
    await openQProxy.mintBounty(
    Constants.bountyId,
    Constants.organization,
    atomicBountyInitOperation
    )
    const bountyAddress = await openQProxy.bountyIdToAddress(
    Constants.bountyId
    )

    // ACT
    await depositManager.fundBountyToken(
    bountyAddress,
    ethers.constants.AddressZero,
    100,
    1,
    0
    )
})
```
## Tool used
Hardhat
Manual Review

## Recommendation
Check for non-zero `msg.value` if token address is `address(0)`