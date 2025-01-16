# **Audit Report for `Manager` Contract**

## **Summary**
The `Manager` contract implements an ERC-4626 vault for managing wstETH collateral and minting FUSD stablecoins. It includes functionality for depositing, withdrawing, liquidating undercollateralized positions, and handling unrealized yield. The following issues were identified during the audit.

---

## **Findings**

### **1. Liquidation Penalty BPS Misconfiguration**

- **Severity**: High
- **Issue**: The `LIQUIDATION_PENALTY_BPS` is set to `1100`, representing an 11% penalty. However, it is intended to represent a 110% penalty, which should be `11000` basis points.
- **Impact**: Liquidators are undercompensated, making the liquidation process unprofitable for them. This disincentivizes liquidation and could lead to undercollateralized positions remaining unresolved.
- **Recommendation**: Update the value of `LIQUIDATION_PENALTY_BPS` to `11000` in the contract.

**Code Reference:**
```solidity
uint public constant LIQUIDATION_PENALTY_BPS = 1100; // Change to 11000
```

---

### **2. Absence of Minimum Liquidation Amount**

- **Severity**: Medium
- **Issue**: Without a minimum liquidation amount, it is possible to perform multiple small liquidations to avoid paying the protocol's liquidation fee. 
- **Impact**: This behavior undermines the protocol’s fee model and allows bad actors to exploit the system for profit. 
- **Recommendation**: Introduce a minimum liquidation amount to prevent fee circumvention. For example:

```solidity
uint public constant MIN_LIQUIDATION_AMOUNT = 100 ether; // Example minimum amount

require(amount >= MIN_LIQUIDATION_AMOUNT, "Liquidation amount too low");
```

---

### **3. Minting Shares as Performance Fee on Unrealized Yield**

- **Severity**: Medium
- **Issue**: The contract mints new shares to the protocol (as performance fees) when the wstETH ↔ stETH ratio increases. However, this yield is unrealized, as no additional wstETH has been deposited. This dilutes the shares of existing depositors, particularly long-term holders, while benefiting short-term holders.
- **Impact**: This introduces an unfair distribution of protocol fees, penalizing long-term holders who keep their deposits intact.
- **Recommendation**: Consider applying performance fees at the time of withdrawals based on realized yield. For example:

```solidity
function withdraw(uint assets, address receiver, address owner) public override returns (uint shares) {
    // Calculate performance fee on the realized yield
    uint fee = calculatePerformanceFee(owner);
    distributeFee(fee); // Mint or transfer shares as fees

    // Proceed with withdrawal
    shares = super.withdraw(assets, receiver, owner);
}
```

**Additional Recommendation**: Include logic to track realized yield for each depositor and apply fees accordingly.

---

## **Additional Suggestions**

- **Gas Optimization**: 
  - Use `previewDeposit` in the `harvest` function for better readability and adherence to ERC-4626 standards.
  - Consolidate fee calculation logic into reusable utility functions to improve code maintainability.
  
- **Improved Test Cases**:
  - Add tests for edge cases, such as very small liquidation amounts, to ensure fee calculations behave as expected.
  - Verify that depositors’ shares are not unfairly diluted under different yield scenarios.

---

## **Conclusion**

The `Manager` contract is well-implemented overall but has a few critical and medium-priority issues that need to be addressed to ensure fairness and security. By fixing the liquidation penalty configuration, introducing a minimum liquidation amount, and revising the performance fee mechanism, the protocol can achieve better economic incentives and fairness for its users.

**Priority Fixes**:
1. Update `LIQUIDATION_PENALTY_BPS` to `11000`.
2. Add a minimum liquidation amount.
3. Transition to a realized-yield-based performance fee mechanism.

--- 

If you’d like assistance with implementing these changes or reviewing the fixes, let me know!

