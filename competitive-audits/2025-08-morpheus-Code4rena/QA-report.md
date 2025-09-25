## [Low -1] (Contract :: setDistributor) Unsafe use of `approve` .
### **Description:**  

The function uses a raw `approve` call when updating token allowances. This may introduce compatibility issues with tokens that do not fully comply with the ERC20 standard, particularly with non-zero to non-zero allowance updates.  

```solidity
IERC20(token).approve(spender, amount);
```

### **Impact:**  
- Potential incompatibility with some ERC20 tokens.  
- Minor governance/UX risk if integration fails silently.  


### **Recommended Mitigation:**  
- Use `safeApprove` pattern to first reset the allowance to `0`, then set the new value.  
```solidity
IERC20(token).approve(spender, 0);
IERC20(token).approve(spender, type(uint256).max);
```

## [Low -2] (DepositPool::migrate) Unsafe use of `transfer` for `remainder_`

### **Description:**  
DepositPool uses the raw `transfer()` method to send `remainder_` tokens to the Distributor during migration. Some ERC20 tokens are non-standard (e.g., they don’t return `bool`, or implement fee-on-transfer logic). This can cause unexpected failures or silent misbehavior, reducing compatibility and safety of migrations. 
Raw `transfer()` calls are unsafe for tokens that deviate from the ERC20 standard. Using `SafeERC20.safeTransfer` ensures compatibility with non-standard tokens by correctly handling return values and revert conditions.

```solidity
// Current (unsafe)
IERC20(depositToken).transfer(distributor, remainder_);
```

### **Impact:**  

- Potential compatibility issues with non-standard ERC20 tokens.  
- Risk of migration failure or silent token loss if a token doesn’t strictly follow ERC20 spec.  
- Reduced safety and auditability.

### **Recommended Mitigation:**  

- Use `safeApprove` pattern  using OpenZeppelin’s SafeERC20 .

```solidity
 IERC20(depositToken).safeTransfer(distributor, remainder_);
```
## [Low-3] {_claimReferrerTier::no zero pendingRewards check}

### Description
The `_claimReferrerTier` function calls `ReferrerLib.claimReferrerTier` and assigns the result to `pendingRewards_`.  
However, the function does not validate whether `pendingRewards_ > 0` before making an external call to the distributor and emitting an event.

This allows users to claim with zero rewards, leading to unnecessary external calls and meaningless events.

### Impact
- **Gas waste:** Distributor external call is made even if no rewards are due.  
- **Log clutter:** `ReferrerClaimed` events may be emitted with `pendingRewards_ = 0`.  
- **UX impact:** Users may be misled by events showing zero reward claims.  

### Recommended Mitigation
Add a zero-reward check before proceeding:
```solidity
require(pendingRewards_ > 0, "DS: no pending rewards");
```

## [Gas- 1] {_withdraw::require(amount_ > 0) – Late validation causes wasted gas}

### Description
In the `_withdraw` function, the statement `require(amount_ > 0)` is placed after multiple external/storage reads (`isRewardPoolPublic`, `userData.deposited`, and `rewardPoolsProtocolDetails`).  
If `amount_ == 0`, these expensive operations are performed unnecessarily before the revert.  

### Impact
Users submitting zero withdrawals will consume more gas than necessary before reverting.  
This does not break functionality, but increases gas costs.  

### Recommended Mitigation
Move the `require(amount_ > 0)` check earlier in the function, ideally right after adjusting `amount_` against `deposited_`.  


## [Gas- 2] {_claim::rewardPoolsProtocolDetails redundant SLOADs}

### Description
In the `_claim` function, `rewardPoolsProtocolDetails[rewardPoolIndex_]` is accessed multiple times in back-to-back `require` statements.  
Each access incurs an additional SLOAD (~2100 gas).  

### Impact
Increased gas costs during claims due to redundant storage reads.  

### Recommended Mitigation
Cache the struct reference in a local variable:  
```solidity
RewardPoolProtocolDetails storage poolDetails = rewardPoolsProtocolDetails[rewardPoolIndex_];
require(block.timestamp > userData.lastStake + poolDetails.claimLockPeriodAfterStake, ...);
require(block.timestamp > userData.lastClaim + poolDetails.claimLockPeriodAfterClaim, ...);
```

### [Gas-3] (Contract::_getCurrentUserReward) Struct parameter can use `calldata` instead of `memory`

**Description**  
The function `_getCurrentUserReward` currently accepts the `UserData` struct parameter as `memory`. Since this struct is only read and not modified, using `memory` incurs unnecessary memory allocation and copying. Marking the struct as `calldata` is cheaper, as it avoids extra data copying and instead directly references the input.  

```solidity
function _getCurrentUserReward(
    uint256 currentPoolRate_,
    UserData memory userData_
) private pure returns (uint256) {
    uint256 deposited_ = userData_.virtualDeposited == 0 ? userData_.deposited : userData_.virtualDeposited;
    uint256 newRewards_ = ((currentPoolRate_ - userData_.rate) * deposited_) / PRECISION;
    return userData_.pendingRewards + newRewards_;
}
```


## [Info -1] (DepositPool::editReferrerTiers) Inconsistent error message wording

### **Description:**  
The revert/error message uses wording `"tiers (2)"` instead of `"tiers (1)"`, creating a mismatch in expected output.  
This does not affect contract logic or security but may confuse integrators, auditors, or frontends that rely on consistent error messages.

### **Impact:**  
- No security or functional impact.  
- Minor developer experience / clarity issue.  
- Could slightly increase confusion in audits, testing, or UI integrations.

### **Recommended Mitigation:**  
- Standardize wording in error/revert messages for consistency, e.g., ensure all error strings reference `"tiers (1)"` or whichever index is correct.  
Prefer using **custom errors** instead of string messages for gas efficiency and clarity:
```solidity
error InvalidTier();

if (tierIndex != 1) {
    revert InvalidTier();
}
```

