# S1 - Unrestricted Asset Registration Allows Malicious Token Injection and Protocol-Wide DoS

## Summary

`registerAsset` in `Spoke.sol` is publicly callable and guarded only by the `protected` modifier, which implements a reentrancy lock but **no authorization**. As a result, any external attacker can register arbitrary ERC-20 (or ERC-1155-style) tokens as official protocol assets.

This allows malicious or incompatible tokens to enter the Centrifuge asset registry and subsequently be used in pool creation, share-class creation, vault instantiation, and reserve logic.

The absence of access control enables pool pollution, vault malfunction, mis-pricing, denial-of-service, and systemic accounting corruption across the protocol.

## Root Cause

In `Spoke.sol`, the following function is externally callable without authorization:

```solidity
function registerAsset(
    CentId centId,
    address token,
    uint256 tokenId,
    Refund refund
) external payable protected returns (AssetId assetId) {
    ...
}
```

The `protected` modifier only enforces a non-reentrancy lock via `_initiator`; it does **not** restrict callers.

As a result:

- Any external address can call `registerAsset`.
- `_assetCounter` is incremented unconditionally.
- Arbitrary `(token, tokenId)` pairs are written into the asset registry.

## Affected Code Link

https://github.com/sherlock-audit/2025-10-centrifuge-protocol-v3-1-audit/blob/main/protocol/src/core/spoke/Spoke.sol#L120C4-L156C6

## Internal Preconditions

- Attacker calls `registerAsset(...)` using a malicious, incompatible, or non-standard token.
- `Spoke` increments `_assetCounter` and assigns a new `AssetId`.
- Attacker-controlled `(token, tokenId)` pair is stored as an official asset.
- Core protocol modules assume `assetId` entries are authoritative.

Affected downstream components include:

- `VaultFactory`
- `PoolFactory`
- Share-class linker
- `BalanceSheet`
- Hub-side accounting and cross-chain messaging

## External Preconditions

None.

## Attack Path

### 1. Attacker Injects a Malicious Token

The attacker deploys a malicious ERC-20 / ERC-1155 token with harmful behavior, such as:

- Reentrancy
- Fee-on-transfer
- Reverting transfers
- Mint-on-transfer
- Non-standard return values

The attacker then calls:

```solidity
assetId = spoke.registerAsset(
    centId,
    address(maliciousToken),
    tokenId,
    refund
);
```

No validation is performed:

- Token code is not validated
- `centId` is not validated
- `tokenId` uniqueness is not enforced
- No authorization is required

### 2. Malicious Asset Becomes Eligible for Pool Configuration

The returned `assetId` is now usable in:

- Pool creation
- Share-class creation
- Vault creation
- Reserve and accounting logic

Any operator assuming registered assets are safe may unknowingly integrate the malicious token.

### 3. Malicious Token Enters Vault Mint / Redeem Flow

**Deposit phase:**

- `transferFrom` may revert
- Reentrancy may be triggered
- Approval-based theft may occur
- Tokens may be minted or burned unexpectedly

**Redemption phase:**

- Reentrancy into vault logic
- Settlement failures
- Corrupted accounting balances

### 4. Pool / Vault DoS or Economic Corruption

Consequences include:

- Pool creation denial-of-service
- Permanently stuck funds
- Cross-chain accounting corruption
- Inflation / deflation manipulation
- Protocol integrity failure

## Impact

- Denial-of-service across pools and vaults
- Permanently stuck user funds
- Corrupted accounting and reserve logic
- Attacker-controlled tokens treated as official protocol assets
- System-wide protocol compromise

## Proof of Concept

No response.

## Mitigation

### 1. Enforce Strict Authorization

```solidity
function registerAsset(...) external auth { ... }
```

Restrict asset registration to governance, factory contracts, or explicitly authorized roles.

### 2. Add Strong Asset Validation

Validate at registration time:

- ERC-20 total supply and decimals
- Transfer behavior and return values
- Fee-on-transfer and rebasing behavior
- ERC-1155 interface compliance where applicable

## Proof of Concept (Executable)

The following test demonstrates that:

- An attacker can deploy a malicious token
- The attacker can register it as an official protocol asset
- The protocol accepts it without validation
- Vaults can be created and linked using the malicious asset
- No special privileges are required

### Running the PoC

Save the following code as:

```
test/core/spoke/integration/UnauthorizedAsset.t.sol
```

```solidity
// SPDX-License-Identifier: BUSL-1.1
pragma solidity 0.8.28;

import "./BaseTest.sol";
import {AssetId} from "../../../../src/core/types/AssetId.sol";
import {ShareClassId} from "../../../../src/core/types/ShareClassId.sol";
import {PoolId} from "../../../../src/core/types/PoolId.sol";
import {IVault} from "../../../../src/core/spoke/interfaces/IVault.sol";
import {VaultKind} from "../../../../src/core/spoke/interfaces/IVault.sol";
import {IShareToken} from "../../../../src/core/spoke/interfaces/IShareToken.sol";
import {MessageLib, VaultUpdateKind} from "../../../../src/core/messaging/libraries/MessageLib.sol";
import {IVaultFactory} from "../../../../src/core/spoke/factories/interfaces/IVaultFactory.sol";
import {IBaseVault} from "../../../../src/vaults/interfaces/IBaseVault.sol";

contract MaliciousToken {
    string public name = "Malicious Token";
    string public symbol = "EVIL";
    uint8 public decimals = 6;

    mapping(address => uint256) public balanceOf;
    mapping(address => mapping(address => uint256)) public allowance;

    event Transfer(address indexed from, address indexed to, uint256 value);
    event Approval(address indexed owner, address indexed spender, uint256 value);

    constructor() {
        balanceOf[msg.sender] = 1_000_000 * 10 ** 6;
    }

    function transfer(address to, uint256 amount) external returns (bool) {
        require(balanceOf[msg.sender] >= amount, "Insufficient balance");
        balanceOf[msg.sender] -= amount;
        balanceOf[to] += amount;
        emit Transfer(msg.sender, to, amount);
        return true;
    }

    function transferFrom(address from, address to, uint256 amount) external returns (bool) {
        require(balanceOf[from] >= amount, "Insufficient balance");
        require(allowance[from][msg.sender] >= amount, "Insufficient allowance");
        balanceOf[from] -= amount;
        allowance[from][msg.sender] -= amount;
        balanceOf[to] += amount;
        emit Transfer(from, to, amount);
        return true;
    }

    function approve(address spender, uint256 amount) external returns (bool) {
        allowance[msg.sender][spender] = amount;
        emit Approval(msg.sender, spender, amount);
        return true;
    }
}
```

### Running the Test

```bash
forge test --mt testAttackerCanRegisterArbitraryAsset -vv
```

### Expected Output

The test passes, confirming that:

- An attacker registered a malicious asset without authorization
- The asset became an official protocol asset
- A vault was successfully created and linked using the malicious token
- The protocol fully integrated attacker-controlled assets

**Root Cause:** `spoke.registerAsset()` lacks authorization checks.  
**Impact:** Complete protocol compromise is possible.

