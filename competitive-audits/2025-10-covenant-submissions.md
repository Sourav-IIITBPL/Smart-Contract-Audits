
# S1 - Missing Validation in ValidationLogic.sol Allows Creation of Self-Referential Markets

Severity
Medium

## Description
A market may be created with baseToken equal to quoteToken or with identical edge prices where edgeSqrtPriceX96_A equals edgeSqrtPriceX96_B. The function ValidationLogic.checkMarketParams, which is called during Covenant.createMarket, does not validate these conditions. As a result, multiple malformed markets can be created, each of which is permanently unusable.

The current implementation only validates that the LEX implementation and curator are whitelisted and that the market does not already exist.

Code excerpt

```solidity
function checkMarketParams(
    MarketParams calldata marketParams,
    MarketParams storage storageMarketParams,
    mapping(address LEXimplementation => bool) storage validLEX,
    mapping(address Oracle => bool) storage validCurator
) internal view {
    if (!validLEX[address(marketParams.lex)])
        revert Errors.E_LEXimplementationNotAuthorized();

    if (!validCurator[address(marketParams.curator)])
        revert Errors.E_CuratorNotAuthorized();

    if (address(storageMarketParams.baseToken) != address(0))
        revert Errors.E_MarketAlreadyExists();
}
```

## Root Cause
Missing sanity checks at market creation and lack of defense-in-depth in the math layer. ValidationLogic.checkMarketParams does not ensure that baseToken and quoteToken are distinct, nor does it ensure that edge prices are strictly ordered. LatentMath.computeLiquidity assumes a non-zero price delta and uses it directly as a divisor, which causes reverts when the delta is zero.

## Attack Scenario
An attacker constructs MarketParams such that:
The baseToken and quoteToken are the same address
Or the edge prices are equal such that edgeSqrtPriceX96_A equals edgeSqrtPriceX96_B

The attacker uses a whitelisted lex and curator and calls Covenant.createMarket with these parameters. ValidationLogic.checkMarketParams succeeds, allowing the market to be created. Subsequent operations on the market revert, rendering it permanently unusable.

## Impact
Availability
The affected market becomes unusable. Minting, swapping, updating, or redeeming positions will consistently revert.

Scope
The impact is local to the malformed markets. However, an attacker may create multiple such markets if the whitelist allows it, amplifying the availability impact across the protocol.

Funds
There is no direct loss of user funds because transactions revert. However, user experience, integrations, and off-chain tooling may break, and protocol operators must intervene to remediate affected markets.

Likelihood
The issue is realistic if market creation is permissionless or broadly permissioned. The likelihood increases with wider LEX and curator whitelists.

## Recommended Mitigations

Validation at Market Creation
Add explicit checks during market creation to prevent degenerate markets.

```solidity
require(marketParams.baseToken != marketParams.quoteToken, "COV: base == quote");
require(
    marketParams.edgeSqrtPriceX96_B > marketParams.edgeSqrtPriceX96_A,
    "COV: invalid edge prices"
);
```

Defense in Depth in LatentMath
Add validation inside the math library to prevent unsafe assumptions if invalid parameters reach this layer.

```solidity
require(sqrtRatioX96_B > sqrtRatioX96_A, "LM: invalid edge prices");
```

## Proof of Concept

```solidity
function test_submissionValidity() public {
    MarketParams memory badMarketParams = MarketParams({
        baseToken: _mockBaseAsset,
        quoteToken: _mockBaseAsset,
        curator: _mockOracle,
        lex: address(lex)
    });

    MarketId badMarketId =
        covenant.createMarket(badMarketParams, hex"");

    MarketParams memory fetched =
        covenant.getIdToMarketParams(badMarketId);

    assertEq(address(fetched.baseToken), _mockBaseAsset);
    assertEq(address(fetched.quoteToken), _mockBaseAsset);

    MintParams memory mp = MintParams({
        marketId: badMarketId,
        marketParams: badMarketParams,
        baseAmountIn: 1e18,
        to: address(this),
        minATokenAmountOut: 0,
        minZTokenAmountOut: 0,
        data: hex"",
        msgValue: 0
    });

    MockERC20(_mockBaseAsset)
        .approve(address(covenant), 1e18);

    vm.expectRevert();
    covenant.mint(mp);
}
```

## Conclusion
The absence of basic market parameter validation allows the creation of self-referential or degenerate markets that permanently revert on use. While no direct fund loss occurs, the issue causes denial of service at the market level and degrades protocol reliability. Adding simple validation checks at creation time and reinforcing assumptions in the math layer fully mitigates this class of issues.


# S2 - Unrestricted `updateState` Function Enables Fee Accrual Griefing and Base Supply Manipulation

## Description

`Covenant.updateState` is an external, publicly callable function that forwards execution to the private `_updateState` function. Internally, `_updateState` calls:

```solidity
ILiquidExchangeModel(marketParams.lex).updateState{value: msgValue}(
    marketId,
    marketParams,
    marketState[marketId].baseSupply,
    data
);
```

The returned `protocolFees` are then applied as follows:

```solidity
marketState[marketId].protocolFeeGrowth += protocolFees;
marketState[marketId].baseSupply -= protocolFees;
```

Because `updateState` is **publicly callable**, has **no rate limiting**, no permissioning, and does not enforce that `msgValue` meaningfully covers oracle or update costs, **any external actor can repeatedly trigger state updates**.

Repeated calls cause the Liquid Exchange Model (LEX) accrual logic to run multiple times, incrementally increasing `protocolFeeGrowth` and decreasing `baseSupply`, even in the absence of genuine market activity.

This enables griefing and market-state manipulation.

## Root Cause

The issue arises from the combination of:

- Public accessibility of `updateState`.
- `_updateState` unconditionally applying returned `protocolFees` to storage.
- Absence of rate limiting, debouncing, or caller restrictions.
- LEX `updateState` logic that accrues fees based on time, oracle reads, and internal math.

An attacker can force accrual logic to execute more frequently than intended and under selectively favorable oracle states.

## Attack Scenario

1. Attacker selects a target `marketId`.
2. Attacker repeatedly calls:

```solidity
covenant.updateState(marketId, marketParams, data, 0);
```

3. Execution flow:

- `updateState` → `_updateState`.
- `_checkPayment(msgValue)` passes with `msgValue = 0`.
- `ValidationLogic.checkUpdateParams` only verifies parameter consistency.
- `ILiquidExchangeModel.updateState` executes accrual logic.
- Non-zero `protocolFees` may be returned.
- `protocolFeeGrowth` is incremented.
- `baseSupply` is decremented.

4. The attacker repeats this process across blocks or batches calls.

## Effects Over Repeated Calls

- `baseSupply` steadily decreases, reducing liquidity available for swaps, mints, or redeems.
- `protocolFeeGrowth` increases, distorting internal accounting.
- Downstream calculations (e.g., LTV, market state, mint/redeem pricing) operate on degraded inputs.
- Users may face worse pricing, higher effective fees, or failed operations.

While the attacker cannot directly withdraw protocol fees (owner-only), the **state manipulation itself causes economic harm** and may create arbitrage or MEV opportunities for other actors.

## Impact

- **Availability & Correctness:** Artificial reduction of `baseSupply` degrades market functionality.
- **Economic Grief:** Honest users experience worse UX, pricing distortions, and reduced liquidity.
- **Indirect Value Loss:** Altered accounting can lead to user losses during trades or redemptions.
- **Likelihood:** Realistic — function is public, cheap to call, and accrual effects compound over time.

## Recommended Mitigations

### Restrict Callers

Limit who may invoke `updateState`:

```solidity
function updateState(...) external payable onlyAuthorizedUpdater lock(marketId) { ... }
```

### Rate Limit Per Market

Introduce a minimum interval between updates:

```solidity
require(
    block.timestamp >= marketState[marketId].lastUpdateTimestamp + minUpdateInterval,
    "update too soon"
);
```

### Additional Hardening

- Enforce non-zero or bounded `msgValue` where oracle updates are expected.
- Separate passive accrual from user-triggered paths.
- Document intended callers and update cadence.

## Proof of Concept

Place the following code in `test_submissionValidity` located at `test/poc/C4PoC.t.sol`.

```solidity
function test_submissionValidity() public {
    MarketParams memory mParams = MarketParams({
        baseToken: _mockBaseAsset,
        quoteToken: _mockQuoteAsset,
        curator: _mockOracle,
        lex: address(lex)
    });

    MintParams memory seedMp = MintParams({
        marketId: _marketId,
        marketParams: mParams,
        baseAmountIn: 1e18,
        to: address(this),
        minATokenAmountOut: 0,
        minZTokenAmountOut: 0,
        data: hex"",
        msgValue: 0
    });

    MockERC20(_mockBaseAsset).approve(address(covenant), 1e18);
    covenant.mint{value: 0}(seedMp);

    MarketState memory msBefore = covenant.getMarketState(_marketId);
    uint128 beforeProtocolFees = msBefore.protocolFeeGrowth;
    uint256 beforeBaseSupply = msBefore.baseSupply;

    vm.warp(block.timestamp + 1 hours);
    covenant.updateState{value: 0}(_marketId, mParams, hex"", 0);

    MarketState memory msAfter1 = covenant.getMarketState(_marketId);

    bool feeIncreasedOrBaseReduced =
        (msAfter1.protocolFeeGrowth > beforeProtocolFees) ||
        (msAfter1.baseSupply < beforeBaseSupply);

    assertTrue(feeIncreasedOrBaseReduced);

    for (uint256 i = 0; i < 3; ++i) {
        vm.warp(block.timestamp + 1 hours);
        covenant.updateState{value: 0}(_marketId, mParams, hex"", 0);
    }

    MarketState memory msFinal = covenant.getMarketState(_marketId);
    assertTrue(msFinal.protocolFeeGrowth >= msAfter1.protocolFeeGrowth);
    assertTrue(msFinal.baseSupply <= msAfter1.baseSupply);
}
```

## How to Run

```bash
forge test --mt test_submissionValidity -vvv
```

