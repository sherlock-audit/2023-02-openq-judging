Jeiwan

medium

# Oracle is not initialized in `OpenQV1`, making the contract partially non-operational

## Summary
Upon deployment and initialization, the oracle is not set in the `OpenQV1` contract. This makes the functions that can be triggered by the oracle non-operational, until the oracle is set.
## Vulnerability Detail
The `OpenQV1` contract implements multiple functions that can be called by the oracle, namely:
- [associateExternalIdToAddress](https://github.com/sherlock-audit/2023-02-openq/blob/main/contracts/OpenQ/Implementations/OpenQV1.sol#L464)–can be called only by the oracle;
- [setInvoiceComplete](https://github.com/sherlock-audit/2023-02-openq/blob/main/contracts/OpenQ/Implementations/OpenQV1.sol#L207)–can be called by a bounty issuer or the oracle;
- [setSupportingDocumentsComplete](https://github.com/sherlock-audit/2023-02-openq/blob/main/contracts/OpenQ/Implementations/OpenQV1.sol#L231)–can be called by a bounty issuer or the oracle.

However, the oracle is not set during contract initialization:
```solidity
function initialize() external initializer onlyProxy {
    __Ownable_init();
    __UUPSUpgradeable_init();
    __ReentrancyGuard_init(); // @audit-issue missing __Oraclize_init
}
```
Thus, the `_oracle` state variable will be set to the zero address.
## Impact
`OpenQV1` uses the oracle for automation. Since the oracle is not set during deployment and initialization, the following functions will be impacted:
- Assigning external IDs to users via the [associateExternalIdToAddress](https://github.com/sherlock-audit/2023-02-openq/blob/main/contracts/OpenQ/Implementations/OpenQV1.sol#L464) function won't be possible since the function can only be called by an oracle;
- Automated changing of invoice status via the [setInvoiceComplete](https://github.com/sherlock-audit/2023-02-openq/blob/main/contracts/OpenQ/Implementations/OpenQV1.sol#L207) function won't be possible.
- Automated changing of supporting documents status via the [setSupportingDocumentsComplete](https://github.com/sherlock-audit/2023-02-openq/blob/main/contracts/OpenQ/Implementations/OpenQV1.sol#L231) function won't be possible.
## Code Snippet
[OpenQV1.sol#L15-L17](https://github.com/sherlock-audit/2023-02-openq/blob/main/contracts/OpenQ/Implementations/OpenQV1.sol#L15-L17)
## Tool used
Manual Review
## Recommendation
Consider initializing an oracle address via the `__Oraclize_init` function in the `initialize` function of the `OpenQV1` contract.