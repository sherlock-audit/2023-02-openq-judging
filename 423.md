yixxas

high

# An attacker can use a malicious token contract to force prevent atomic bounty and tiered percentage bounty claims

## Summary
A whitelist is used for when a bounty accepts NFT. However, any ERC20 tokens can be used to deposit into bounty. While it does use a whitelist, it is used to enforce a limit on number of non-whitelisted tokens that can be deposited, rather than the kind of tokens that can be deposited.

A malicious token contract can be crafted by an attacker to force claims made in atomic bounty and tier percentage bounty to revert.

## Vulnerability Detail

Below is a code snippet of how ERC20 tokens are transferred to claimant when a claim is made in `_claimAtomicBounty()`. It loops through the token addresses, which is added when a new ERC20 token is deposited into the contract and then calls `claimBalance()`, which transfers the token to claimant.

```solidity
for (uint256 i = 0; i < _bounty.getTokenAddresses().length; i++) {
	uint256 volume = _bounty.claimBalance(
		_closer,
		_bounty.getTokenAddresses()[i]
	);
}
```

The issue here is that a single revert on one of the tokens will cause all claims to be reverted. NFT transfers will be reverted too as they are all done in one functon `_claimAtomicBounty()`.

A malicious attacker can craft a ERC20 token contract that always reverts on all `transfer` except for the first one, such that it can be first used to be funded into the contract.

The same issue is found in `_claimTieredPercentageBounty()`.

## Impact

## Code Snippet
https://github.com/sherlock-audit/2023-02-openq/blob/main/contracts/ClaimManager/Implementations/ClaimManagerV1.sol#L123-L166
https://github.com/sherlock-audit/2023-02-openq/blob/main/contracts/ClaimManager/Implementations/ClaimManagerV1.sol#L203-L272

## Tool used

Manual Review

## Recommendation
Consider using whitelist to control the kind of ERC20 tokens we are accepting.

