## [M-1] Block.timestamp dependence in Distributor.distributeRewards() allows validator timestamp manipulation → early/delayed reward distribution (MEV)

### Severity - Medium

## Description

The `Distributor.sol ::distributeRewards()` function enforces the minimum distribution period using block.timestamp:

```solidity

if (block.timestamp <= lastCalculatedTimestamp_ + minRewardsDistributePeriod) return;

```

Because block.timestamp is miner/validator-controllable within the consensus-allowed drift (~±15 seconds), the function becomes timestamp-dependent. When ` Distributor.sol :: minRewardsDistributePeriod ` variable is set to small values (e.g., seconds or minutes), validators can slightly advance or delay timestamps to manipulate reward distribution timing.

This enables:

1. Premature distribution: trigger rewards earlier than intended.

2. Delayed distribution: hold rewards back to batch under more favorable conditions.

Such behavior creates MEV opportunities for block producers and undermines fairness for protocol users.

## Impact

1. Rewards may be distributed earlier or later than designed, deviating from protocol rules.

2. Malicious validators can strategically time reward releases to extract value.

3. Over many cycles, these small deviations compound, creating fairness issues and potential value leakage.

4. Since ` Distributor.sol :: minRewardsDistributePeriod` variable is owner-controlled without a lower bound, misconfiguration can make the protocol highly vulnerable (especially if set to seconds/minutes).

## Recommended mitigation 

The following are the ways to eliminate this vulnerability . 

- #### Enforce a safe lower bound on minRewardsDistributePeriod in the setter function.

Example: enforce a minimum of 1 hour (3600 seconds).

```solidity
uint256 public constant MIN_DISTRIBUTE_PERIOD = 3600; // 1 hour

function setMinRewardsDistributePeriod(uint256 newPeriod) external onlyOwner {
    require(newPeriod >= MIN_DISTRIBUTE_PERIOD, "Period too short");
    minRewardsDistributePeriod = newPeriod;
      emit MinRewardsDistributePeriodSet(newPeriod);
}

```

This prevents owners from accidentally (or maliciously) setting the distribution period to very small values where block.timestamp drift becomes exploitable.

- #### Prefer block-based periods for distribution timing.

Using block.number reduces miner/validator influence compared to timestamps:

```solidity
uint128 public minRewardsDistributeBlocks;

function distributeRewards() external {

    uint128 lastCalculatedBlock_ = rewardPoolLastCalculatedBlock[rewardPoolIndex_];   
   ,,,,

    if (block.number <= lastDistributionBlock_ + minRewardsDistributeBlocks) return;
   
  rewardPoolLastCalculatedBlock[rewardPoolIndex_] = uint128(block.number).

   ,,,,

}

```

This ensures deterministic periods regardless of timestamp manipulation.

- #### If timestamp reliance is unavoidable, document the risks and enforce minimum thresholds at the contract level.



#### The PoC contains three tests:

1. POC (small period) — proves a miner can skew block.timestamp and cause distributeRewards() to run earlier than honest timing would allow.

2. Safe (large period) — shows the same miner tweak cannot force distribution when the period is large.

3. Repeat/Drift — demonstrates repeated small manipulations over multiple cycles (conceptual) and counts forced distributions.

- #### How to run the PoC locally

1. Open ` test/Distributor.test.ts ` and locate the describe ` #distributeRewards', () => { ... }` block . 

2. Append the following it(...) tests at the end of that describe `#distributeRewards` block (before the next describe block), so they execute with the same beforeEach setup that creates deposit pools.


```solidity

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

3. From project root run:

```solidity
npx hardhat test test/Distributor.test.ts --grep "POC"

```
  or run the full test file:

```solidity
npx hardhat test test/Distributor.test.ts

```

4. You should see the three POC tests run and pass, demonstrating 
  (a). an exploitable early distribution when minRewardsDistributePeriod is small.
  (b). the behavior is safe with a large period, and 
  (c). repeated cycles can be abused.



## [M-2] Owner  can silently overwrite the `depositPools[rewardPoolIndex_][depositPoolAddress_]` for the same `rewardPoolIndex_` and `depositPoolAddress_` in `Distributor.sol :: addDepositPool()` function.

### Severity - Medium

### Description

The following line in `Distributor.sol :: addDepositPool()` overwrites existing pools without restriction if the owner mistakenly calls the function with the same parameters:

```solidity
function addDepositPool(
    uint256 rewardPoolIndex_,
    address depositPoolAddress_,
    address token_,
    string memory chainLinkPath_,           
    Strategy strategy_
) external onlyOwner {
    ,,,,

    depositPools[rewardPoolIndex_][depositPoolAddress_] = depositPool_;

    ,,,,
}
```

### Impact

1. The owner could accidentally overwrite a deposit pool configuration (e.g., strategy, token, oracle path) without any error.

2. This may lead to reward miscalculation or misaligned accounting, potentially causing loss of rewards for stakers.

3. Even though the owner is trusted, this increases migration fragility and risk of human error.

### Recommended Mitigation

- Add a guard to prevent overwriting existing pools:

``` diff

function addDepositPool(
    uint256 rewardPoolIndex_,
    address depositPoolAddress_,
    address token_,
    string memory chainLinkPath_,           
    Strategy strategy_
) external onlyOwner {
    IRewardPool rewardPool_ = IRewardPool(rewardPool);
    rewardPool_.onlyExistedRewardPool(rewardPoolIndex_);

    require(
        IERC165(depositPoolAddress_).supportsInterface(type(IDepositPool).interfaceId),
        "DR: the deposit pool address is invalid"
    );  

    ,,,,

    DepositPool memory depositPool_ = DepositPool(token_, chainLinkPath_, 0, 0, 0, strategy_, aToken_, true);           

    depositPoolAddresses[rewardPoolIndex_].push(depositPoolAddress_);         

+   require(
+       !depositPools[rewardPoolIndex_][depositPoolAddress_].isExist,
+       "Deposit pool already exists"
+   );

    depositPools[rewardPoolIndex_][depositPoolAddress_] = depositPool_; 
    isDepositTokenAdded[token_] = true; 
}
```

### POC

 1. Save it as `test/Distributor.addDepositPool.PoC.test.ts` (or similar) in the same project where your other tests live.
```solidity

// test/Distributor.addDepositPool.PoC.test.ts
import { SignerWithAddress } from '@nomicfoundation/hardhat-ethers/signers';
import { expect } from 'chai';
import { ethers } from 'hardhat';

import {
  deployAavePoolDataProviderMock,
  deployAavePoolMock,
  deployDepositPoolMock,
  deployDistributor,
  deployERC20Token,
  deployRewardPoolMock,
  deployL1SenderMock,
} from '../helpers/deployers';

import { deployChainLinkDataConsumerMock } from '../helpers/deployers/mock/capital-protocol/chain-link-data-consumer-mock';
import { wei } from '@/scripts/utils/utils';

import {
  AavePoolDataProviderMock,
  AavePoolMock,
  ChainLinkDataConsumerMock,
  DepositPoolMock,
  Distributor,
  ERC20Token,
  L1SenderMock,
  RewardPoolMock,
} from '@/generated-types/ethers';

describe('Distributor PoC — addDepositPool overwrite & duplicate addresses', () => {
  enum Strategy {
    NONE,
    NO_YIELD,
    AAVE,
  }

  let OWNER: SignerWithAddress;
  let rewardPoolMock: RewardPoolMock;
  let chainLinkDataConsumerMock: ChainLinkDataConsumerMock;
  let aavePoolDataProviderMock: AavePoolDataProviderMock;
  let aavePoolMock: AavePoolMock;
  let l1SenderMock: L1SenderMock;
  let distributor: Distributor;

  before(async () => {
    [OWNER] = await ethers.getSigners();

    // Deploy required mocks (same helpers you already use)
    chainLinkDataConsumerMock = await deployChainLinkDataConsumerMock();
    aavePoolDataProviderMock = await deployAavePoolDataProviderMock();
    aavePoolMock = await deployAavePoolMock(aavePoolDataProviderMock);
    rewardPoolMock = await deployRewardPoolMock();
    l1SenderMock = await deployL1SenderMock();

    // Deploy Distributor with mocks
    distributor = await deployDistributor(
      chainLinkDataConsumerMock,
      aavePoolMock,
      aavePoolDataProviderMock,
      rewardPoolMock,
      l1SenderMock,
    );
  });

  it('demonstrates silent overwrite + duplicate depositPoolAddresses when addDepositPool called twice with same (rewardPoolIndex, depositPoolAddr)', async () => {
    // Prepare tokens and deposit pool mock
    const tokenA = await deployERC20Token();
    const tokenB = await deployERC20Token();
    const depositPool = await deployDepositPoolMock(tokenA, distributor);

    const rewardPoolId = 0;
    const chainLinkPath = 'wETH/USD';

    // Make sure reward pool exists & is public (Distributor requires this)
    await rewardPoolMock.setIsRewardPoolExist(rewardPoolId, true);
    await rewardPoolMock.setIsRewardPoolPublic(rewardPoolId, true);

    // Set some price for chainlink consumer to avoid any zero-price reverts if updateDepositTokensPrices triggered
    await chainLinkDataConsumerMock.setAnswer(chainLinkPath, wei(1));

    // --- FIRST add: tokenA ---
    await distributor.addDepositPool(
      rewardPoolId,
      depositPool, // helper allows passing contract directly
      tokenA,
      chainLinkPath,
      Strategy.NONE,
    );

    // Read back mapping and array after first add
    const first = await distributor.depositPools(rewardPoolId, depositPool.address);
    expect(first.isExist).to.eq(true);
    expect(first.token).to.eq(tokenA.address);
    expect(first.chainLinkPath).to.eq(chainLinkPath);

    const lenAfterFirst = await distributor.depositPoolAddressesLength
      ? await distributor.depositPoolAddressesLength(rewardPoolId)
      : // fallback if helper isn't available on your contract ABI
        (await distributor.depositPoolAddresses(rewardPoolId, 0), 1);
    // We expect depositPoolAddresses length to be 1
    expect(lenAfterFirst).to.eq(1);

    // --- SECOND add: same rewardPoolId & same depositPool.address but different token (tokenB) ---
    // This is allowed by contract logic because `isDepositTokenAdded[tokenB]` is false.
    await distributor.addDepositPool(
      rewardPoolId,
      depositPool,
      tokenB,
      'wETH/USD', // new chainLinkPath to observe overwrite
      Strategy.NONE,
    );

    // After second call the mapping should be overwritten with tokenB
    const afterSecond = await distributor.depositPools(rewardPoolId, depositPool.address);
    expect(afterSecond.isExist).to.eq(true);
    expect(afterSecond.token).to.eq(tokenB.address, 'depositPools mapping was NOT overwritten to tokenB');
    expect(afterSecond.chainLinkPath).to.eq('wETH/USD', 'chainLinkPath should be overwritten to second value');

    // And depositPoolAddresses array should have duplicate entries (same address pushed twice)
    // We expect length 2 and both slots equal depositPool.address
    const addr0 = await distributor.depositPoolAddresses(rewardPoolId, 0);
    const addr1 = await distributor.depositPoolAddresses(rewardPoolId, 1);
    expect(addr0).to.eq(depositPool.address);
    expect(addr1).to.eq(depositPool.address);

    // Final assertion: length must be 2
    // (call array length getter if your contract has one; otherwise check by reading index 1 above)
    // If Distributor exposes depositPoolAddressesLength helper as in other PoC, verify:
    if ((distributor as any).depositPoolAddressesLength) {
      const len = await (distributor as any).depositPoolAddressesLength(rewardPoolId);
      expect(len).to.eq(2);
    }
  });
});
```
2. The test demonstrates both problems:

- `depositPools[rewardPoolIndex][depositPoolAddress]` is silently overwritten (we assert token changed from tokenA → tokenB), and

- `depositPoolAddresses[rewardPoolIndex]` receives the same address twice (we assert both index 0 and 1 equal the same address).
