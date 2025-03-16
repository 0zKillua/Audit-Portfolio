## 1. High-Fee Fraxswap Pair Can Sabotage Liquidity Migration

### Severity : High
## description
In LiquidityManager::addLiquidityToFraxswap

During Migration the LiquidityManager contract checks if a Fraxswap pair already exists before migrating liquidity. If a pair exists, it automatically uses that pair for migration.
However, the contract does not validate the fee parameter of the existing pair before using it.
Since any user can create a Fraxswap pair at any time, for a given agentToken and it's currencyToken,
a malicious actor could preemptively create the pair with an extremely high swap fee.
## Impact
High-Fee Fraxswap Pair Becomes Unusable:
Users will be forced to trade with an extremely high fee, making the Fraxswap pool ineffective.
Fraxswap pairs do not allow fee changes after creation.
Once a high-fee pair is used, the protocol is permanently stuck with it.
## Recommended mitigation:
Instead of waiting until target liquidity is reached, create the Fraxswap Pair at Agent Token creation.
Have a max limit on fee and check against it.
+++ require(fraxswapPair.fee() <= MAX_ALLOWED_FEE, "Pair exists but fee is too high");
## Links to affected code
LiquidityManager.sol#L133-L150


## 2. Underflow & Division by Zero revert in getAmountIn Function

## Severity Low
## description
The BootstrapPool::getAmountIn function contains an issue due to which users cannot compute the amount required to swap, to get the desired output.
uint256 _denominator = (_reserveOut - _amountOut) * fee;
amountOut is the user input, whenever we have amountOut >= reserveOut, the getAmountIn function reverts.
## Impact
Function reverts with underflow , division with zero errors.
The function does not gracefully handle user inputs.
## Recommended mitigation
Graceful error handling:

Add a require check to ensure amountOut is within limits.
Or, Instead of reverting, return an indicative error code with available liquidity.
++ require(_amountOut < _reserveOut, "Invalid amountOut: exceeds available liquidity");
## Links to affected code
BootstrapPool.sol#L171