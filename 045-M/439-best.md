0xdeadbeef

medium

# Null characters are not checked in string values

## Summary

OpenQ consumes data from the on-chain contracts using even monitoring with subgraph. 
Strings with null characters that are emitted as part of an on-chain event will be recognized off-chain as if they were without null character. 

This can corrupt data off-chain and possibly lead to impersonation.
Some examples of strings that are emitted that include strings:
`TokenDepositReceived` - contains issue id and organization name (ORG name ZZ\x00Z will be the same as ZZZ)
`BountyClosed` - contains issue id and organization name (ORG name ZZ\x00Z will be the same as ZZZ), closer data.

## Vulnerability Detail

Events are emitted without validation that the string does not contain any null characters,
Example can be found:
https://github.com/sherlock-audit/2023-02-openq/blob/main/contracts/ClaimManager/Implementations/ClaimManagerV1.sol#L294

Due to a known limitation on subgraph:
https://github.com/graphprotocol/graph-ts/issues/264

Events that contain strings with null characters will be transformed to the same string without the null characterss 

## Impact

The OpenQ platform is heavily dependent on the on-chain events execute the business.

Events are emitted during critical operations such as funding, claiming, refunding, etc..

An attacker can create strings that will corrupt the data and possibly impersonate himself to a user/org
- ORG name ZZ\x00Z will be the same as ZZZ


## Code Snippet

In the description

## Tool used

Manual Review

## Recommendation

Consider adding a validator function that checks every index of the string to see if it is the ASCII range.
This will also help restrict to specific encoding.
