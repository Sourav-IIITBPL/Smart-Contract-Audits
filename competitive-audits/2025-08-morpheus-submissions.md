# S1 - Block.timestamp Dependence in `Distributor.distributeRewards()` Allows Validator Timestamp Manipulation → Early/Delayed Reward Distribution (MEV)

## Description

The `Distributor.sol::distributeRewards()` function enforces the minimum distribution period using `block.timestamp`:

```solidity
if (block.timestamp <= lastCalculatedTimestamp_ + minRewardsDistributePeriod) return;
```

Because `block.timestamp` is miner/validator-controllable within the consensus-allowed drift (approximately ±15 seconds), the function becomes **timestamp-dependent**. When `Distributor.sol::minRewardsDistributePeriod` is set to small values (e.g., seconds or minutes), validators can slightly advance or delay timestamps to manipulate reward distribution timing.

This enables:

- **Premature distribution**: triggering rewards earlier than intended.
- **Delayed distribution**: holding rewards back to batch under more favorable conditions.

Such behavior creates **MEV opportunities** for block producers and undermines fairness for protocol users.

---

## Impact

- Rewards may be distributed **earlier or later** than designed, deviating from protocol rules.
- Malicious validators can strategically time reward releases to **extract value**.
- Over many cycles, these small deviations **compound**, creating fairness issues and potential value leakage.
- Since `minRewardsDistributePeriod` is **owner-controlled without a lower bound**, misconfiguration can make the protocol highly vulnerable (especially if set to seconds/minutes).

---

## Recommended Mitigation

### 1. Enforce a Safe Lower Bound on `minRewardsDistributePeriod`

Enforce a minimum duration in the setter function to prevent unsafe configurations.

Example: enforce a minimum of **1 hour (3600 seconds)**.

```solidity
uint256 public constant MIN_DISTRIBUTE_PERIOD = 3600; // 1 hour

function setMinRewardsDistributePeriod(uint256 newPeriod) external onlyOwner {
    require(newPeriod >= MIN_DISTRIBUTE_PERIOD, "Period too short");
    minRewardsDistributePeriod = newPeriod;
    emit MinRewardsDistributePeriodSet(newPeriod);
}
```

This prevents owners from accidentally (or maliciously) setting the distribution period to very small values where `block.timestamp` drift becomes exploitable.

---

### 2. Prefer Block-Based Periods for Distribution Timing

Using `block.number` reduces miner/validator influence compared to timestamps.

```solidity
uint128 public minRewardsDistributeBlocks;

function distributeRewards() external {
    uint128 lastCalculatedBlock_ = rewardPoolLastCalculatedBlock[rewardPoolIndex_];

    if (block.number <= lastCalculatedBlock_ + minRewardsDistributeBlocks) return;

    rewardPoolLastCalculatedBlock[rewardPoolIndex_] = uint128(block.number);
    // ... distribution logic
}
```

This ensures deterministic periods regardless of timestamp manipulation.

---

### 3. If Timestamp Reliance Is Unavoidable

- Explicitly **document the risks** in protocol documentation.
- Enforce **minimum safe thresholds** at the contract level.

---

## Proof of Concept (PoC)

The PoC contains three tests:

1. **Small period** — proves a miner can skew `block.timestamp` and cause `distributeRewards()` to run earlier than honest timing would allow.
2. **Safe (large period)** — shows the same miner tweak cannot force distribution when the period is large.
3. **Repeat / Drift** — demonstrates repeated small manipulations over multiple cycles (conceptual) and counts forced distributions.

---

## How to Run the PoC Locally

1. Open `test/Distributor.test.ts` and locate the following block:

```ts
describe('#distributeRewards', () => {
```

2. Append the following `it(...)` tests **at the end of this describe block** (before the next `describe` block), so they execute with the same `beforeEach` setup that creates deposit pools.

---

### PoC 1 — Early Distribution via Timestamp Skew (Small Period)

```ts
it('POC: miner can force early distribution via timestamp skew (minPeriod small)', async () => {
  // Use the same deposit pools created by beforeEach
  // Set a small min period (attacker/miner can exploit ±15s tweak)
  await distributor.setMinRewardsDistributePeriod(60);
  // Put last calculated timestamp at a known baseline
  await distributor.setRewardPoolLastCalculatedTimestamp(publicRewardPoolId, 1000);

  // Make sure there is reward to distribute
  await imitateYield([wei(1), wei(1)], wei(100), [wei(5, 6), wei(5)]);

  // Honest time: just BEFORE boundary (1000 + 60 = 1060)
  await setNextTime(1055);
  // Early call should be skipped (since 1055 <= 1060)
  await distributor.distributeRewards(publicRewardPoolId);
  expect(await distributor.rewardPoolLastCalculatedTimestamp(publicRewardPoolId)).to.eq(1000);

  // Miner skews timestamp forward by a few seconds (plausible within block-producer window)
  await setNextTime(1065); // now > 1060 -> distribution allowed
  await distributor.distributeRewards(publicRewardPoolId);

  // lastCalculatedTimestamp should update to the block time we set
  expect(await distributor.rewardPoolLastCalculatedTimestamp(publicRewardPoolId)).to.eq(1065);

  // At least one deposit pool should have received distributed rewards
  const d0 = await distributor.distributedRewards(dp0Info.rewardPoolId, dp0Info.depositPool);
  const d1 = await distributor.distributedRewards(dp1Info.rewardPoolId, dp1Info.depositPool);
  const sum = d0.add(d1);
  expect(sum.isZero()).to.be.false;
});
```

---

### PoC 2 — Safe Configuration With Large Period

```ts
it('POC: miner cannot force distribution when minPeriod is large (safe config)', async () => {
  // Large period (e.g., 1 hour)
  await distributor.setMinRewardsDistributePeriod(3600);
  await distributor.setRewardPoolLastCalculatedTimestamp(publicRewardPoolId, 1000);

  await imitateYield([wei(1), wei(1)], wei(100), [wei(5, 6), wei(5)]);

  // Honest time: 20s before boundary (1000 + 3600 = 4600)
  await setNextTime(4600 - 20); // 4580
  await distributor.distributeRewards(publicRewardPoolId);
  expect(await distributor.rewardPoolLastCalculatedTimestamp(publicRewardPoolId)).to.eq(1000);

  // Miner can at best tweak by ~15s; still insufficient (4580 + 15 = 4595)
  await setNextTime(4580 + 15);
  await distributor.distributeRewards(publicRewardPoolId);
  expect(await distributor.rewardPoolLastCalculatedTimestamp(publicRewardPoolId)).to.eq(1000);
});
```

---

### PoC 3 — Repeated Small Manipulations Across Multiple Cycles

```ts
it('POC: repeated small manipulations — miner can force distributions across multiple cycles when period is small', async () => {
  // Small period so miner has a chance each cycle
  await distributor.setMinRewardsDistributePeriod(60);

  // Start baseline
  const start = 10_000;
  await distributor.setRewardPoolLastCalculatedTimestamp(publicRewardPoolId, start);
  await distributor.setMinRewardsDistributePeriod(60);

  let forcedDistributions = 0;

  // Run a few cycles (conceptual demonstration)
  for (let i = 0; i < 4; i++) {
    // Make sure there is reward this cycle
    await imitateYield([wei(1), wei(1)], wei(50), [wei(2, 6), wei(2)]);

    // Read last timestamp (as number)
    const lastBn = await distributor.rewardPoolLastCalculatedTimestamp(publicRewardPoolId);
    const last = lastBn.toNumber();

    // Honest time progression: set to last + 55 (5s before boundary)
    await setNextTime(last + 55);
    // Without miner skew, distribution will be skipped
    await distributor.distributeRewards(publicRewardPoolId);

    // Miner attempts to push block timestamp a bit forward (within realistic tweak)
    // If pushing to last + 61 ( > last + 60 ) works, the miner forced distribution
    await setNextTime(last + 61);
    await distributor.distributeRewards(publicRewardPoolId);

    const newLast = (await distributor.rewardPoolLastCalculatedTimestamp(publicRewardPoolId)).toNumber();
    if (newLast > last) forcedDistributions += 1;
  }

  // We expect at least one forced distribution in the loop for plausibility
  // (on a real chain, whether each cycle succeeds depends on exact timings; this demonstrates possibility)
  expect(forcedDistributions).to.be.gte(1);
});
```

---

## Running the Tests

From the project root:

```bash
npx hardhat test test/Distributor.test.ts --grep "POC"
```

Or run the full test file:

```bash
npx hardhat test test/Distributor.test.ts
```

You should observe all three PoC tests passing, demonstrating:

- (a) Exploitable early distribution when `minRewardsDistributePeriod` is small.
- (b) Safety when a large distribution period is enforced.
- (c) The feasibility of abuse across repeated cycles.


# S2 - Deposit Pools Can Be Silently Overwritten

## Description

The following assignment allows an existing deposit pool to be overwritten **without any restriction or validation**:

```solidity
depositPools[rewardPoolIndex_][depositPoolAddress_] = depositPool_;
```

If a deposit pool already exists for the given `rewardPoolIndex_` and `depositPoolAddress_`, the new configuration silently replaces the old one.

---

## Impact

- The owner may **accidentally overwrite** an existing deposit pool configuration (e.g., strategy address, underlying token, oracle path, or accounting parameters) without any error or warning.
- Silent overwrites can lead to **reward miscalculation**, broken accounting assumptions, or incorrect reward routing.
- Stakers associated with the overwritten deposit pool may experience **loss or misallocation of rewards**.
- Even though the owner is assumed to be trusted, this behavior significantly **increases migration and upgrade fragility**, especially during protocol reconfiguration or pool expansion.

---

## Recommendation

Introduce a guard that prevents overwriting an already-initialized deposit pool.

```solidity
require(
    !depositPools[rewardPoolIndex_][depositPoolAddress_].isActive,
    "Deposit pool already exists"
);
```

This ensures:
- Deposit pools are **append-only** by default.
- Any attempt to redefine an existing pool fails loudly rather than silently mutating state.

If pool updates are required as part of protocol governance, they should be performed via a **dedicated update function** with explicit intent, events, and migration safeguards.

---

## Proof of Concept (Conceptual)

1. A deposit pool is created for a given `rewardPoolIndex_` and `depositPoolAddress_`.
2. The owner later calls the same creation logic again with a different `depositPool_` configuration.
3. The previous pool configuration is overwritten without reverting.
4. Existing stakers now interact with a pool whose parameters no longer match the assumptions under which they deposited.

This behavior is silent and undetectable on-chain unless additional off-chain monitoring is in place.

---

## Severity Consideration

While this issue does not directly enable an external attacker, it:
- Breaks **defensive configuration assumptions**.
- Introduces **high operational risk** during upgrades or migrations.
- Can cause irreversible accounting inconsistencies affecting user rewards.

As such, it is appropriate to treat this as a **Medium-severity configuration and safety issue**.

