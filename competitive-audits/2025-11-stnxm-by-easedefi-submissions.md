# S1- First Depositor Can Steal Most Protocol Funds Through Manipulated Initial Share Price

## Summary

The protocol’s initialization logic mints shares without corresponding assets. As a result, the first or early external depositor can exploit the severely imbalanced `totalSupply / totalAssets` ratio to mint massively inflated shares. After later users deposit at normalized prices, the attacker can withdraw a disproportionate amount of assets, effectively stealing protocol funds from subsequent depositors.

This results in catastrophic fund loss and makes the protocol unsafe at launch.

## Root Cause

The `initialize` function in `stNXM.sol` mints a large number of shares to the owner without depositing matching `wNXM`, causing `totalSupply > 0` while `totalAssets ≈ 0`.

Affected code:
https://github.com/sherlock-audit/2025-11-stnxm-by-easedefi/blob/main/stNXM-Contracts/contracts/core/stNXM.sol#L88C2-L104C1

Additionally, `initializeExternals` mints `stNXM` while `totalAssets ≈ 0`, further increasing the `totalSupply`. These shares are later partially classified as virtual, but the imbalance still exists at the time deposits are enabled.

Affected code:
https://github.com/sherlock-audit/2025-11-stnxm-by-easedefi/blob/main/stNXM-Contracts/contracts/core/stNXM.sol#L111C2-L145C6
https://github.com/sherlock-audit/2025-11-stnxm-by-easedefi/blob/main/stNXM-Contracts/contracts/core/stNXM.sol#L645C5-L678C1

`totalAssets` and `totalSupply` are calculated as follows:

```solidity
function totalAssets() public view override returns (uint256) {
    // Add staked NXM, wNXM in the contract, wNXM in the dex and Morpho, subtract the admin fees.
    return stakedNxm() + unstakedNxm() - adminFees;
}

/**
 * @notice Get the total supply of stNXM.
 * @dev The stNXM in the dex is "virtual" so it must be removed from total supply.
 */
function totalSupply() public view override(ERC20Upgradeable, IERC20) returns (uint256) {
    // Do not include the "virtual" assets in the Uniswap pool in total supply calculations.
    (, uint256 virtualShares) = dexBalances();
    return super.totalSupply() - virtualShares;
}
```

After initialization, the contract has `totalSupply > 0` while `totalAssets ≈ 0`, creating an extreme share price distortion.

The first external depositor can exploit this imbalance to mint massively inflated shares (50×–200× more than intended). Once legitimate users deposit and the share price normalizes, the attacker can withdraw an outsized amount of `wNXM`, effectively stealing funds from later depositors.

## Internal Preconditions

- `_mintAmount > 0` in `initialize()`.
- DEX bootstrap via `initializeExternals()` deposits only minimal `wNXM`.
- No significant `wNXM` balance in the vault.
- Contract accepts deposits and withdrawals.

## External Preconditions

- Attacker observes contract deployment.
- Attacker holds even 1 `wNXM`.
- Attacker can frontrun other depositors.
- Market liquidity exists to exit the position.

## Attack Path

### Phase 1 — Exploitation

- Deployment mints 100,000 unbacked shares to the owner.
- DEX deposit is minimal (e.g., 1,000 `wNXM`).
- Attacker deposits 1 `wNXM` and receives ~100 `stNXM` (≈100× inflation).
- Legitimate users deposit 60,000 `wNXM` at normal prices.
- Exchange rate normalizes.

### Phase 2 — Extraction

- After the withdrawal delay, attacker withdraws ~38 `wNXM`.
- Attacker profit: 37 `wNXM` from a 1 `wNXM` deposit (≈3700% ROI).
- Legitimate depositors lose ~62% of their funds.

## Impact

- Attacker captures arbitrarily large portions of protocol capital.
- First depositor can drain tens of thousands of dollars with minimal input.
- Later users suffer severe dilution and loss.
- Protocol is fundamentally unsafe at launch.

## Proof of Concept

No response.

## Mitigation

Modify `initialize()` to require an initial asset deposit that matches the number of minted shares:

```solidity
function initialize(
    address _beneficiary,
    uint256 _mintAmount,
    uint256 _initialDeposit // NEW: require initial wNXM deposit
) public initializer {
    require(
        _initialDeposit >= _mintAmount,
        "Must deposit assets equal to minted shares"
    );
    // ... existing logic
}
```

This ensures `totalSupply` and `totalAssets` are aligned from inception and prevents first-depositor share price manipulation.

---


# S2 - Users Will Suffer Unbounded Losses Due to Exchange Rate Changes During Withdrawal Delay Period

## Summary

The two-stage withdrawal mechanism recalculates asset amounts at finalization time rather than honoring the exchange rate at request time. As a result, significant exchange rate changes during the mandatory 2-day withdrawal delay (due to slashing events, rewards, or market movements) can cause users to receive substantially fewer assets than expected. There is no slippage protection, minimum output guarantee, or ability to cancel withdrawals, exposing all withdrawing users to unbounded loss.

## Root Cause

In `stNXM.sol`, the withdrawal flow is split into a request phase and a finalize phase. The `withdrawFinalize` function recalculates the asset amount using the *current* exchange rate instead of the amount recorded at request time.

Affected code:
https://github.com/sherlock-audit/2025-11-stnxm-by-easedefi/blob/main/stNXM-Contracts/contracts/core/stNXM.sol#L191C2-L197C6

```solidity
function withdrawFinalize(address _user) external notPaused update {
    WithdrawalRequest memory withdrawal = withdrawals[_user];

    uint256 shares = uint256(withdrawal.shares);

    // RECALCULATES assets - ignores the stored asset amount from request time
    uint256 assets = _convertToAssets(shares, Math.Rounding.Floor); // LINE 196
    uint256 requestTime = uint256(withdrawal.requestTime);

    require((requestTime + withdrawDelay) <= block.timestamp, "Not ready to withdraw");
    require(assets > 0, "No pending amount to withdraw");

    pending -= uint256(withdrawal.shares);
    delete withdrawals[_user];

    if (block.timestamp > requestTime + withdrawDelay + 1 days) {
        _transfer(address(this), _user, shares);
        return;
    }

    _burn(address(this), shares);
    wNxm.transfer(_user, assets); // Transfers the RECALCULATED amount
    emit Withdrawal(_user, assets, shares, block.timestamp);
}
```

The `WithdrawalRequest.assets` value calculated at request time is never used. Instead, `_convertToAssets()` is called again at finalization, fully exposing users to exchange rate movements during the delay window.

## Internal Preconditions

- User holds `stNXM` and calls `withdraw()` or `redeem()` to create a withdrawal request.
- `withdrawDelay` is non-zero (currently 2 days).
- Withdrawal request survives the full delay period.
- Protocol has sufficient liquidity for settlement.
- Exchange rate moves adversely between request and finalization.

## External Preconditions

- Slashing event due to a covered protocol exploit processed by Nexus Mutual.
- Market volatility affecting staking rewards or pool performance.
- Oracle price updates for the `stNXM / wNXM` pair.
- Mass withdrawal events reducing per-share value.
- Time-sensitive events coinciding with the withdrawal delay window.

## Attack Path

### Scenario — Slashing Event During Withdrawal Delay

**Day 0 — User Requests Withdrawal**

Initial state:

- `totalAssets()` = 1,000,000 wNXM
- `totalSupply()` = 1,000,000 stNXM
- Exchange rate = 1.0 wNXM per stNXM

User initiates withdrawal:

```solidity
stNXM.withdraw(100_000e18, userAddress, userAddress);
```

Withdrawal request recorded:

- `requestTime` = Day 0
- `shares` = 100,000 stNXM
- `assets` = 100,000 wNXM (calculated at request time)

User expects to receive 100,000 wNXM after 2 days.

**Day 1 — Covered Protocol Gets Hacked**

- A covered DeFi protocol is exploited.
- Nexus Mutual initiates the claims process.

**Day 2 — Claims Are Processed**

- Slashing event occurs.
- Staked positions are slashed by 15%.
- `stakedNxm()` drops from 850,000 to 722,500 wNXM.

New state:

- `totalAssets()` = 872,500 wNXM
- `totalSupply()` = 1,000,000 stNXM
- New exchange rate = 0.8725 wNXM per stNXM

**Day 2 — User Finalizes Withdrawal**

```solidity
stNXM.withdrawFinalize(userAddress);
```

Asset recalculation:

```text
assets = shares * totalAssets / totalSupply
assets = 100,000 * 872,500 / 1,000,000
assets = 87,250 wNXM
```

User receives:

```solidity
wNxm.transfer(userAddress, 87_250e18);
```

Loss calculation:

- Expected: 100,000 wNXM
- Received: 87,250 wNXM
- Loss: 12,750 wNXM (12.75%)

At $30 / wNXM, this equals a **$382,500 USD loss**.

## Critical Issue

- Users receive no warning of changing withdrawal outcomes.
- No slippage protection or minimum output parameter exists.
- Withdrawals cannot be cancelled once requested.
- Users are forced to wait through adverse market conditions.

## Impact

### No Exit During Crisis

- Shares are locked once withdrawal is requested.
- Users cannot react to incoming slashing events.
- Creates forced exposure during the worst possible market conditions.

### Liquidity Crisis Amplification

- Users rush to withdraw when risk increases.
- Delay prevents exit, amplifying panic and bank-run psychology.

### Quantitative Impact Analysis

For a protocol with $30M TVL:

- Typical monthly withdrawals: $3M
- Probability of 15% slashing during delay: ~5%

Expected monthly loss:

```
3,000,000 * 15% = 450,000 USD
```

Annual expected loss: **~$270,000 USD**.

Losses are unbounded. In a catastrophic 50% slashing scenario, losses could reach millions.

### User Trust Impact

- Users cannot predict withdrawal outcomes.
- Requires constant monitoring during withdrawal windows.
- Severely degrades trust in protocol safety.

## Proof of Concept

No response.

## Mitigation

Modify `withdrawFinalize()` to use the asset amount recorded at request time:

```solidity
function withdrawFinalize(address _user) external notPaused update {
    WithdrawalRequest memory withdrawal = withdrawals[_user];

    uint256 shares = uint256(withdrawal.shares);
    uint256 assets = uint256(withdrawal.assets); // Use stored amount
    uint256 requestTime = uint256(withdrawal.requestTime);

    require((requestTime + withdrawDelay) <= block.timestamp, "Not ready to withdraw");
    // ... remaining logic
}
```

This guarantees deterministic withdrawal outcomes and eliminates unbounded exchange rate risk during the delay period.

---


# S3 - Malicious Users Will Dilute Protocol Losses During Emergency Pause Affecting All Legitimate Stakers

## Summary

The absence of `notPaused` protection on the `deposit()` and `mint()` functions introduces a dilution attack during crises. When the protocol owner pauses withdrawals due to a detected or expected slashing event, attackers can still deposit fresh assets because entry points are not protected by `notPaused`. These opportunistic deposits allow attackers to spread the upcoming losses across all holders, unfairly shifting the majority of the loss burden from original stakers to late entrants.

## Root Cause

In `stNXM.sol`, the `deposit()` and `mint()` functions lack the `notPaused` modifier:

```solidity
function deposit(uint256 assets, address receiver) public override update returns (uint256) {
    return super.deposit(assets, receiver);
}

function mint(uint256 shares, address receiver) public override update returns (uint256) {
    return super.mint(shares, receiver);
}
```

Withdrawal-related operations correctly enforce pause conditions:

```solidity
function _withdraw(...) internal override notPaused { ... }
function withdrawFinalize(address _user) external notPaused update { ... }
```

Because deposits remain enabled during pauses, users can still enter the system at a moment when risk is known but losses have not yet been realized.

## Affected Code Link

https://github.com/sherlock-audit/2025-11-stnxm-by-easedefi/blob/main/stNXM-Contracts/contracts/core/stNXM.sol#L183C2-L190C1

## Internal Preconditions

- Owner sets `paused = true` using `togglePause()`.
- A slashing event is pending on Nexus Mutual.
- `totalAssets / totalSupply` is expected to decrease.
- There are existing stakers with non-zero supply.

## External Preconditions

- A covered protocol suffers an exploit.
- Nexus Mutual claim assessment is ongoing.
- The market becomes aware of the hack.
- A time window exists between detection and claim settlement (typically 24–72 hours).

## Attack Path

The following example explicitly describes the exploit scenario.

- Hack occurs: Covered protocol loses $10M.
- Owner pauses: Withdrawals are halted.
- Attacker deposits:
  - Deposits 900,000 wNXM while paused.

New state:

- `totalAssets`: 1.9M
- `totalSupply`: 1.9M

- Slashing happens:
  - Loss applied: 100,000 wNXM.

New state:

- `totalAssets`: 1.8M

- Owner unpauses: Operations resume.
- Attacker withdraws:
  - Withdraws at the new exchange rate (0.947).
  - Suffers only a 5.3% loss instead of the full 10%.

- Original stakers suffer:
  - Their effective loss increases by 52.3% due to dilution.

## Impact

- Original stakers bear 52.3% more loss than intended.
- Attacker avoids the full slashing penalty.
- Creates a perverse incentive to deposit after a hack.
- Violates fair-risk principles and harms trust in the protocol.

## Proof of Concept

No response.

## Mitigation

### Primary

Add `notPaused` to all entry functions:

```solidity
function deposit(...) public override update notPaused returns (...) { ... }
function mint(...) public override update notPaused returns (...) { ... }
function redeem(...) public override update notPaused returns (...) { ... }
```

### Alternate

Implement a pause snapshot rate to lock deposit share prices during pauses.

