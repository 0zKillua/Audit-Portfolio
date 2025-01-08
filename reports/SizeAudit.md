## High 01 
### Title - LiquidatorReward is Incorrectly calculated: Inconsistent Units 

 #### Lines of code
 https://github.com/code-423n4/2024-06-size/blob/main/src/libraries/actions/Liquidate.sol#L96-L100 https://github.com/code-423n4/2024-06-size/blob/main/src/libraries/actions/Liquidate.sol#L103 https://github.com/code-423n4/2024-06-size/blob/main/src/libraries/actions/Liquidate.sol#L119
 
 ### Vulnerability details
 #### Impact
 In `executeliquidate` function of `liquidate.sol`, `liquidatorProfitCollateralToken` is wrongly calculated due to an error in computation of `liquidatorReward`.
 
 * `liquidatorProfitCollateralToken` is sum of `debtInCollateralToken` and `liquidatorReward`. `debtInCollateralToken` is measured in terms of collateral tokens (18 decimals), whereas `liquidatorReward` is calculated by comparing two values with different decimals, leading to an inconsistent comparison:
 
 ```
 uint256 liquidatorReward = Math.min(
    assignedCollateral - debtInCollateralToken, //18 decimals
   Math.mulDivUp(debtPosition.futureValue, state.feeConfig.liquidationRewardPercent, PERCENT)//6 decimals
 );
 ```
 
 * a 6 decimal value might often be the output of above comparison which is totally unintended.
 * Instead of using `debtPosition.futureValue`,  `debtInCollateralToken` (18 decimals) should be used in the Math.min(,) comparison to ensure consistency in units.
 * This error in `liquidatorReward` further effects calculations of `collateralRemainder`, `protocolProfitCollateralToken`, leading to a significantly incorrect amount of collateral being transferred to liquidators.
 
 ```
 state.data.collateralToken.transferFrom(debtPosition.borrower, msg.sender, liquidatorProfitCollateralToken);
 ```

