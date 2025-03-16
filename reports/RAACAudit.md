# Core Contracts - Findings Report

# Table of contents
- ## [Contest Summary](#contest-summary)
- ## [Results Summary](#results-summary)
- ## High Risk Findings
    - ### [H-01. Incorrect Debt Calculation in totalSupply function of  DebtToken Contract](#H-01)
    - ### [H-02. Incorrect amount is burned in Debt token contract](#H-02)
    - ### [H-03. Loss of funds from StabilityPool due to double debt scaling during Liquidations](#H-03)
    - ### [H-04. Incorrect Amount Used in RToken Mint Function Leading to Excess Token Minting](#H-04)
    - ### [H-05. Incorrect Return Value Sequence in RToken Mint Function](#H-05)
    - ### [H-06. Incorrect Scaling Calculation in RToken Burn Function](#H-06)
    - ### [H-07. RToken Burn Function Uses Unscaled Amount Leading to Excessive Token Burns](#H-07)
    - ### [H-08. Incorrect Return Value Sequence in RToken's Burn Function](#H-08)
    - ### [H-09. Bypass of minimum lock duration in veRAACToken](#H-09)
    - ### [H-10. Bypass of MAX_TOTAL_LOCKED_AMOUNT via veRAACToken:: function increase](#H-10)
    - ### [H-11. Lock gets overwritten if a user calls `veRAACToken::function lock` twice](#H-11)
    - ### [H-12. Incorrect balanceIncrease calculation in debtToken::mint function](#H-12)
    - ### [H-13. Incorrect Debt Token Minting Amount Due to Unscaled Value Addition](#H-13)
    - ### [H-14. veRAAC::emergencyWithdraw does not update boost state and voting power state](#H-14)
    - ### [H-15. StabilityPool cannot perform Liquidations; Missing Asset Conversion Mechanism](#H-15)
    - ### [H-16. Different precisions used for maxBoost and minBoost in BaseGauge](#H-16)
    - ### [H-17. Performance fee is stuck/lost in the GaugeController contract ](#H-17)
    - ### [H-18. LendingPool's _depositIntoVault always fails due to token flow issue](#H-18)
    - ### [H-19. Partial Repayments Lost Alongside Full Collateral Seizure During Liquidation](#H-19)
    - ### [H-20. _withdrawFromVault function Misdirects Assets to LendingPool, Causing RToken Liquidity Crisis](#H-20)
    - ### [H-21. Unfair Collateral Seizure Despite Full Debt Repayment Due to Manual Liquidation Closure Requirement](#H-21)
    - ### [H-22. loan repayments are accepted post grace period](#H-22)
    - ### [H-23. Incorrect Debt Comparison in Repayment Capping leading to Liquidations](#H-23)
    - ### [H-24.  Under-Collateralization Exploit: Attacker Drains Protocol Funds via Flawed WithdrawNFT function](#H-24)
    - ### [H-25. Critical Borrow Vulnerability: flaw in borrow function allows users to borrow more than threshold limit](#H-25)
- ## Medium Risk Findings
    - ### [M-01. RToken:: CalculateDust function has critical calculation error](#M-01)
    - ### [M-02. no stale price check in getNFT function of LendingPool](#M-02)
    - ### [M-03. Incorrect balanceIncrease calculation in debtToken contract](#M-03)
    - ### [M-04. Time-Weighted Average Corruption via Backdated Updates](#M-04)
    - ### [M-05. Time-Weighted Average Calculations Incorrect When Querying Historical Averages](#M-05)
    - ### [M-06. Missing pause control mechanism in veRAACToken contract](#M-06)
    - ### [M-07. Incorrect emission cap comparison while setting emissions in BaseGauge](#M-07)
    - ### [M-08. Incorrect voting power calculation in `GaugeController::function vote` ignores voting power decay ](#M-08)
- ## Low Risk Findings
    - ### [L-01. `getLockedBalance` function retrieves incorrect data in veRAACToken contract](#L-01)
    - ### [L-02. Incorrect lock expiry data retrieved in `veRAACToken::getLockEndTime` function](#L-02)


# <a id='contest-summary'></a>Contest Summary

### Sponsor: Regnum Aurum Acquisition Corp

### Dates: Feb 3rd, 2025 - Feb 24th, 2025

[See more contest details here](https://codehawks.cyfrin.io/c/2025-02-raac)

# <a id='results-summary'></a>Results Summary

### Number of findings:
- High: 25
- Medium: 8
- Low: 2


# High Risk Findings

## <a id='H-01'></a>H-01. Incorrect Debt Calculation in totalSupply function of  DebtToken Contract            



## Summary

The `totalSupply()` function uses `rayDiv` instead of  `rayMul `  when applying the normalized debt index, causing systematic underreporting of total debt. 

## Vulnerability Details

`scaledSupply`tracks total Debt tokens. this should be multiplied by the index to get total Debt.

In the return statement below ,`scaledSupply.rayDiv()` inverts the debt accrual logic, making total debt appear to shrink as interest accrues which is wrong. Debt should actually increase over time.

```Solidity
function totalSupply() public view override(ERC20, IERC20) returns (uint256) {
        uint256 scaledSupply = super.totalSupply();
  //@audit should be rayMul, not div. 
  return scaledSupply.rayDiv(ILendingPool(_reservePool).getNormalizedDebt());
    }
```

## Impact

* totalSupply() function is used in mint,burn functions and also in other contracts to track the total debt .
* This math error causes underreporting of total Debt in multiple places of the codebase.

## Tools Used

Manual review

## Recommendations

Instead of rayDiv(), we need to call rayMul() on scaledSupply

```Solidity
return scaledSupply.rayMul(ILendingPool(_reservePool).getNormalizedDebt());
```

## <a id='H-02'></a>H-02. Incorrect amount is burned in Debt token contract            



## Summary

burn function of DebtToken contract, burns incorrect amount of debtTokens. Input argument `amount`is in terms of asset Tokens, so amountScaled would be the corresponding Debt Token amount.

Instead of burning `amountScaled`, function burns `amount`, causing loss of funds to protocol. 

## Vulnerability Details

The \`burn\` function improperly uses raw `amount` instead of scaled debt amount (\`amountScaled\`) when burning debt tokens:

```Solidity
 function burn(
        address from,
        uint256 amount,
        uint256 index
    ) external override onlyReservePool returns (uint256, uint256, uint256, uint256)
{  
  
       ......
  
        uint256 amountScaled = amount.rayDiv(index);

        if (amountScaled == 0) revert InvalidAmount();

        _burn(from, amount.toUint128()); //@audit should be amountScaled
        emit Burn(from, amountScaled, index);
```

## Impact

Whenever users repay the debt in partial amounts, It causes Over-burning of DebtTokens (if interest has accrued, index > 1), which causes loss of funds to the protocol. \
users will pay lesser debt than they have to. Can be exploited by malicious attackers to steal funds from protocol.

## Tools Used

Manual review

## Recommendations

burn `amountScaled` instead of `amount`.

```Solidity
      _burn(from, amountScaled.toUint128()); 
```

## <a id='H-03'></a>H-03. Loss of funds from StabilityPool due to double debt scaling during Liquidations            



## Summary

The `liquidateBorrower` function in the StabilityPool contract contains a critical vulnerability that leads to direct loss of funds. By incorrectly scaling user debt twice, the function forces the Stability Pool to pay out significantly more crvUSD than required during liquidations. Each liquidation event results in excess crvUSD being withdrawn from the Stability Pool, effectively draining user deposits beyond the legitimate liquidation amounts.

## Vulnerability Details

In the `liquidateBorrower` function, user debt is fetched from the lending pool.

The `getUserDebt()` function already returns the normalized debt amount (including accrued interest).

 However, the function then incorrectly multiplies this value again with `getNormalizedDebt()`, to get `scaledUserDebt` ,resulting in double scaling of the debt amount.

```Solidity
//LendingPool contract
function getUserDebt(address userAddress) public view returns (uint256) {
        UserData storage user = userData[userAddress];
        return user.scaledDebtBalance.rayMul(reserve.usageIndex);
    }


//stabilityPool contract
function liquidateBorrower(address userAddress) external onlyManagerOrOwner nonReentrant whenNotPaused {
    _update();
    uint256 userDebt = lendingPool.getUserDebt(userAddress); 
    // userDebt is already normalized (includes accrued interest)
  
    uint256 scaledUserDebt = WadRayMath.rayMul(userDebt, lendingPool.getNormalizedDebt());
  //this .rayMul() inflates the debt value.

    if (userDebt == 0) revert InvalidAmount();
    
    uint256 crvUSDBalance = crvUSDToken.balanceOf(address(this));
    if (crvUSDBalance < scaledUserDebt) revert InsufficientBalance();
    // ... rest of the function
}

```

## Impact

`HIGH`: `Loss of funds from StabilityPool`

* Users will be liquidated for amounts larger than their actual debt
* The Stability Pool will pay more crvUSD than necessary during liquidations
* Protocol's accounting will be incorrect, leading to imbalances
* Excess liquidations could lead to unnecessary losses for borrowers
* The protocol's solvency calculations will be distorted

## Tools Used

Manual review

## Recommendations

* Either Remove the additional scaling operation and use the userDebt value directly,&#x20;
* or set `scaledUserDebt`to `userDebt`as it is already scaled.

  ```Solidity
  function liquidateBorrower(address userAddress) external onlyManagerOrOwner nonReentrant whenNotPaused {
      _update();
      uint256 userDebt = lendingPool.getUserDebt(userAddress); 
      // userDebt is already normalized (includes accrued interest)
    
      uint256 scaledUserDebt = userDebt;

      if (userDebt == 0) revert InvalidAmount();
      
      uint256 crvUSDBalance = crvUSDToken.balanceOf(address(this));
      if (crvUSDBalance < scaledUserDebt) revert InsufficientBalance();
      // ... rest of the function
  }

  ```

## <a id='H-04'></a>H-04. Incorrect Amount Used in RToken Mint Function Leading to Excess Token Minting            



## Summary

The RToken contract's mint function contains a critical vulnerability where it uses the unscaled amount (`amountToMint`) instead of the scaled amount (`amountScaled`) when minting tokens. This results in users receiving more RTokens than they should, leading to protocol fund loss.

## Vulnerability Details

In RToken contract, mint function: 

The function correctly calculates the scaled amount by dividing `amountToMint` by the liquidity index using ray math (27 decimal precision). However, it then incorrectly uses the original `amountToMint` value in the `_mint` function instead of the calculated `amountScaled` value.

```Solidity
function mint(
        address caller,
        address onBehalfOf,
        uint256 amountToMint, //; would be in crvUSD terms. 
        uint256 index
    ) external override onlyReservePool returns (bool, uint256, uint256, uint256) {
.....
  
  uint256 amountScaled = amountToMint.rayDiv(index);
_mint(onBehalfOf, amountToMint.toUint128()); // @audit uses amountToMint instead of amountScaled
....
  }
```

## Impact

**`HIGH - The vulnerability leads to direct fund loss for the protocol`**

1. Users receive more RTokens than they should based on their deposited collateral
2. These excess RTokens represent claims on the underlying assets that exceed the actual deposited amount
3. When users redeem their RTokens, they can withdraw more funds than they should be entitled to
4. This creates a deficit in the protocol's reserves, potentially leading to insolvency

## Tools Used

Manual review

## Recommendations

Modify the mint function to use the scaled amount:

```Solidity

uint256 amountScaled = amountToMint.rayDiv(index);
_mint(onBehalfOf, amountScaled.toUint128()); // Fix: use scaled amount

```

## <a id='H-05'></a>H-05. Incorrect Return Value Sequence in RToken Mint Function            



## Summary

The RToken contract's mint function returns values in an incorrect sequence compared to its documented return specification. This mismatch between documentation and implementation can lead to incorrect interpretations by calling contracts, particularly the LendingPool, potentially causing accounting errors in the protocol.



## Vulnerability Details

LendingPool::deposit --> ReserveLibrary:deposit --> RToken.mint()

LendingPool & ReserveLibrary expect `amountScaled`  but instead recieve `amountToMint`.

```Solidity
/**
 * @return A tuple containing:
 *         - bool: True if this is the first mint for the recipient
 *         - uint256: The amount of scaled tokens minted
 *         - uint256: The new total supply after minting
 *         - uint256: The amount of underlying tokens minted
 */
return (isFirstMint, amountToMint, totalSupply(), amountScaled);//@audit incorrect order

```

## Impact

severity: HIGH (Likelihood: Always, Impact: High )

1. The LendingPool contract receives `amountToMint` when it expects `amountScaled` in the second position
2. Since scaled amounts are used to track user balances and protocol accounting, this leads to:

   1. Incorrect user balance calculations
   2. Inaccurate interest accrual
   3. Potential loss of funds due to miscalculated withdrawals

## Tools Used

## Recommendations

Modify the return statement to return correct sequence, as expected. 

`return (isFirstMint, amountScaled, totalSupply(), amountToMint);`

## <a id='H-06'></a>H-06. Incorrect Scaling Calculation in RToken Burn Function            



## Summary

The RToken contract's burn function incorrectly uses `rayMul` instead of `rayDiv` when calculating the `scaled amount`, resulting in inflated burn values that cause users to lose funds by burning more RTokens than necessary for the same amount of underlying assets. This benefits the protocol and remaining RToken holders at the expense of users performing burns.

## Vulnerability Details

The calculation multiplies the amount by the index instead of dividing it. Since the liquidity index typically grows over time and is greater than 1 RAY, this results in an inflated `amountScaled` value.

Example:

* Amount to burn: 100 asset tokens
* Current index: 2.0 RAY
* Current calculation of RTokens: 100 \* 2.0 = 200 (incorrect)
* Expected calculation of RTokens: 100 / 2.0 = 50 (correct)

```Solidity
function burn(
    address from,
    address receiverOfUnderlying,
    uint256 amount,
    uint256 index
) external override onlyReservePool returns (uint256, uint256, uint256) {
    // ...
    uint256 amountScaled = amount.rayMul(index); // @audit incorrect: using rayMul instead of rayDiv
    // ...
}

```

## Impact

HIGH - Users are losing more Tokens than intended. 

1. Users burning RTokens lose significant value
2. Less underlying tokens received per RToken burned
3. The loss increases as the liquidity index grows over time
4. The protocol unfairly gains value at the expense of users



## Recommendations

Replace the rayMul operation with rayDiv

```Solidity
-- uint256 amountScaled = amount.rayMul(index);
++ uint256 amountScaled = amount.rayDiv(index);

```

## <a id='H-07'></a>H-07. RToken Burn Function Uses Unscaled Amount Leading to Excessive Token Burns            



## Summary

The RToken contract's burn function uses the unscaled amount instead of the scaled amount when burning tokens, resulting in an incorrect number of tokens being burned and potential loss of user funds.

## Vulnerability Details

The function calculates a scaled amount but ignores it, instead burning the raw amount. This means the number of RTokens burned doesn't account for the accumulated interest represented by the index.

Example:

* User wants to burn equivalent of 100 underlying Tokens
* Current index: 2.0 RAY
* Should burn: 50 RTokens for 100 underlying tokens
* Actually burns: 100 RTokens for 100 underlying tokens

```Solidity
function burn(
    address from,
    address receiverOfUnderlying,
    uint256 amount,
    uint256 index
) external override onlyReservePool returns (uint256, uint256, uint256) {
    // ...
    
    _burn(from, amount.toUint128()); // @audit incorrect: using amount instead of amountScaled
    // ...
}

```

## Impact

HIGH - Users losing RTokens. This vulnerability results in:

1. Users losing more RTokens than they should when burning
2. Misalignment between RToken supply and underlying asset reserve

## Tools Used

## Recommendations

Use the scaled amount in the burn operation.

```Solidity
-- _burn(from, amount.toUint128());

++ _burn(from, amountScaled.toUint128());


```

## <a id='H-08'></a>H-08. Incorrect Return Value Sequence in RToken's Burn Function            



## Summary

The RToken contract's burn function returns values in an incorrect sequence compared to its expected return values. This mismatch affects the LendingPool contract which calls this function, potentially leading to incorrect accounting and state management.

## Vulnerability Details

LendingPool::withdraw --> ReserveLibrary.withdraw--> RToken.burn()

LendingPool & ReserveLibrary expect `amountScaled`  but instead recieve `amount`, which is higher than than amountScaled.  

```Solidity
 /* @return A tuple containing:
     *         - uint256: The amount of scaled tokens burned
     *         - uint256: The new total supply after burning
     *         - uint256: The amount of underlying asset transferred
     */
function burn(
    address from,
    address receiverOfUnderlying,
    uint256 amount,
    uint256 index
) external override onlyReservePool returns (uint256, uint256, uint256) {
    // ...
    return (amount, totalSupply(), amount); // @audit incorrect sequence
}

```

## Impact

HIGH - This incorrect return sequence leads to:

1. The LendingPool receives `amount` when it expects `amountScaled`
2. This causes incorrect accounting in the protocol as:

   * Scaled amounts are used for internal accounting
   * Wrong values are emitted during events
   * User balances may be incorrectly updated

## Tools Used

## Recommendations

Modify the return tuple to right sequence

`return (amountScaled, totalSupply(), amount);`

## <a id='H-09'></a>H-09. Bypass of minimum lock duration in veRAACToken            



## Summary

The `increase` function in veRAACToken.sol allows users to bypass the minimum lock duration requirement by adding tokens to an existing lock after some time has passed, enabling shorter-than-intended lock periods and potential governance and boost manipulation.

## Vulnerability Details

When users lock tokens via the function lock in veRAAC token, there are proper checks in place to see to that the `MIN_LOCK_DURATION` is not breached:

```solidity
function lock(
        uint256 amount,
        uint256 duration
    ) external nonReentrant whenNotPaused {
       //...
        if (duration < MIN_LOCK_DURATION || duration > MAX_LOCK_DURATION) 
            revert InvalidLockDuration(); //condition to check the MIN_LOCK_DURATION
//rest of the code...
    }
```

But, the `veRAACToken::function increase` updates the lock amount without recalculating or validating the remaining lock duration:

```solidity
function increase(uint256 amount) external nonReentrant whenNotPaused {
        //@audit - we can bypass the MIN lock duration by calling this function after the lock is created and some time has passed.
        // Increase lock using LockManager
        _lockState.increaseLock(msg.sender, amount); 
        _updateBoostState(msg.sender, locks[msg.sender].amount);

        // Update voting power
        LockManager.Lock memory userLock = _lockState.locks[msg.sender];
        (int128 newBias, int128 newSlope) = _votingState
            .calculateAndUpdatePower(
                msg.sender,
                userLock.amount + amount,
                userLock.end
            );

        // Update checkpoints
        uint256 newPower = uint256(uint128(newBias));
        _checkpointState.writeCheckpoint(msg.sender, newPower);

        // Transfer additional tokens and mint veTokens
        raacToken.safeTransferFrom(msg.sender, address(this), amount);
        _mint(msg.sender, newPower - balanceOf(msg.sender));

        emit LockIncreased(msg.sender, amount);
    }
```

A user could use the following attack path to exploit the vulnerability to manipulate governance and boost they might receive:

## Attack path

```markdown
1. User creates lock with 100 RAAC for 365 days (meets minimum)
2. Waits 300 days
3. Calls increase(900 RAAC) - total locked = 1000 RAAC
4. Lock now expires in 65 days  
5. Effective lock duration for new tokens: 65 days << 365 minimum
```

Due to this vulnerabillity, the malicious user gains voting power and boosts their rewards for 1000 RAAC with only 65-day commitment.

## Impact

This loophole undermines the protocol's governance integrity by allowing users to gain disproportionate voting power with minimal commitment. It creates an unfair advantage for sophisticated users who can exploit this mechanism, potentially leading to skewed governance outcomes.

## Tools Used

Manual review

## Recommendations

Can add checks in the increase function to solve this issue:

```solidity
uint256 remaining = lock.end - block.timestamp;
require(remaining >= MIN_LOCK_DURATION, "Can't add to expiring lock");
```

## <a id='H-10'></a>H-10. Bypass of MAX_TOTAL_LOCKED_AMOUNT via veRAACToken:: function increase            



## Summary

The `increase` function allows bypassing the MAX\_TOTAL\_LOCKED\_AMOUNT limit by adding tokens to existing locks after initial creation, enabling the protocol to exceed its intended maximum locked value. Which in turn lets users to dilute voting rights and get unfair boosts.

## Vulnerability Details

The protocol enforces MAX\_TOTAL\_LOCKED\_AMOUNT during initial lock creation but fails to validate it when increasing existing locks, allowing:

1. Initial lock creation under the limit
2. Subsequent increases that push total locked over MAX\_TOTAL\_LOCKED\_AMOUNT

```solidity
function increase(uint256 amount) external nonReentrant whenNotPaused {
        // Increase lock using LockManager //@audit - attacker can bypass the MAX_TOTAL_LOCKED_AMOUNT by calling this function after the lock is created and some time has passed.
        _lockState.increaseLock(msg.sender, amount); 
        _updateBoostState(msg.sender, locks[msg.sender].amount);

        // Update voting power
        LockManager.Lock memory userLock = _lockState.locks[msg.sender];
        (int128 newBias, int128 newSlope) = _votingState
            .calculateAndUpdatePower(
                msg.sender,
                userLock.amount + amount,
                userLock.end
            );
// ... rest of function ...
       
    }
```

The vulnerability arises because:

* MAX\_TOTAL\_LOCKED\_AMOUNT check exists in `lock()`
* No equivalent check in `increase()`

## Impact

Because the MAX\_TOTAL\_LOCKED\_AMOUNT is breached through the increase function, this leads to dilution in the governance voting rights for the people that actually deposited before the limit is beached. This also ends up distorting the boost mechanism, where the users who increase their locks are unfairly rewarded with boosts in spite of having reached the MAX\_TOTAL\_LOCKED\_AMOUNT.

## Tools Used

Manual review

## Recommendations

```solidity
// In veRAACToken.sol's increase function:
function increase(uint256 amount) external nonReentrant whenNotPaused {
    // Add total supply check
    if (totalSupply() + amount > MAX_TOTAL_SUPPLY)
        revert TotalSupplyLimitExceeded();
        
    _lockState.increaseLock(msg.sender, amount);
    // ... rest of function ...
}
```

## <a id='H-11'></a>H-11. Lock gets overwritten if a user calls `veRAACToken::function lock` twice            



## Summary

The `lock()` function allows users to overwrite existing lock positions, resulting in permanent loss of previously locked funds and incorrect voting power calculations.

## Vulnerability Details

The function fails to check for existing locks before creating new positions, allowing complete overwrite of existing locks. And this leads to permanent loss of previously locked RAAC tokens.

Relevant code link: 

This function does not check if the user calling this function already has locked or not, if you call lock() again while having an existing lock:

1. The function would first transfer RAAC tokens to the contract
2. It would then overwrite the existing lock position completely with the new lock and end time
3. The user's previous lock's data would be lost and replaced with the new amount and duration

## Impact

The user that calls the function twice ends up losing all the tokens they deposited the first time as that data is completely overwritten with the new lock's amount and end duration. The likelihood of this happening is high and the user loses all the locked tokens permanently.

## Tools Used

Manual review

## Recommendations

Add checks to prevent locking existing lock or the protocol should suggest to use increase or extend.

```solidity
// In veRAACToken.sol's lock function:
function lock(uint256 amount, uint256 duration) external {
    require(locks[msg.sender].amount == 0, "Existing lock detected");
    
    // Existing logic...
}
```

## <a id='H-12'></a>H-12. Incorrect balanceIncrease calculation in debtToken::mint function            



## Summary

## Vulnerability Details

The DebtToken contract contains a critical mathematical error in the calculation of `balanceIncrease` during minting operations. The current implementation fails to properly account for the fact that `scaledBalance` is already scaled by the current index inside `balanceOf`function.\
multiplying scaledBalance with index again is inflating the variable, which inflates balanceIncrease variable.

`balance Increase= balance@index - balance@oldIndex`

`balance@index= scaledbalance`

`balance@oldIndex= (scaledBalance/index )* _userState[onBehalfOf].index`

correct formula would be:

&#x20;`balanceIncrease = scaledBalance - (scaledBalance.rayMul(_userState[onBehalfOf].index).rayDiv(index)`)

```Solidity
function balanceOf(address account) public view override(ERC20, IERC20) returns (uint256) {
        uint256 scaledBalance = super.balanceOf(account);
@>>    return scaledBalance.rayMul(ILendingPool(_reservePool).getNormalizedDebt());
    }



function mint(
        address user,
        address onBehalfOf,
        uint256 amount,
        uint256 index
    )  external override onlyReservePool returns (bool, uint256, uint256 ) {
  
  .....//
  
        uint256 scaledBalance = balanceOf(onBehalfOf);
        bool isFirstMint = scaledBalance == 0;

        uint256 balanceIncrease = 0;
        if (_userState[onBehalfOf].index != 0 && _userState[onBehalfOf].index < index) {
          //@audit scaledBalance is already result of .rayMul() 
          balanceIncrease = scaledBalance.rayMul(index) - scaledBalance.rayMul(_userState[onBehalfOf].index);
          
          ....//
          }
```

## Impact

With present formula of balanceIncrease, users are minted excessive Debt tokens than intended.\
this is unfair to users.\
Debt accounting is corrupted across the protocol. 

## Tools Used

## Recommendations

```Solidity
//replace the faulty formula with the correct one:
 if (_userState[onBehalfOf].index != 0 && _userState[onBehalfOf].index < index) {
          //@audit scaledBalance is already result of .rayMul() 
     --   balanceIncrease = scaledBalance.rayMul(index) - scaledBalance.rayMul(_userState[onBehalfOf].index);
       ++ balanceIncrease = scaledBalance - (scaledBalance.rayMul(_userState[onBehalfOf].index).rayDiv(index))

          ....//
          
```

## <a id='H-13'></a>H-13. Incorrect Debt Token Minting Amount Due to Unscaled Value Addition            



## Summary

## Vulnerability Details

In the DebtToken's mint function, there is a critical error in calculating `amountToMint`. The function incorrectly adds the raw `amount` with `balanceIncrease`

the mint() function contains following lines of code

```Solidity
uint256 amountToMint = amount + balanceIncrease; // Incorrect, amount is not scaled.
_mint(onBehalfOf, amountToMint.toUint128());

```

1. `amount` is the raw, unscaled input amount
2. `balanceIncrease` is already scaled according to the index
3. Adding these mismatched values leads to incorrect debt token minting

## Impact

* **Incorrect Debt Accounting**: Users will receive excessive amounts of debt tokens due to mixing scaled and unscaled values, as unscaled `amount`is way higher than `amountScaled`.
* **Protocol Insolvency Risk**: The mismatch between actual debt and minted tokens could lead to protocol insolvency

## Tools Used

## Recommendations

Instead of  `amount` use `amountScaled`

```Solidity
uint256 amountToMint = amountScaled + balanceIncrease;
_mint(onBehalfOf, amountToMint.toUint128());

```

## <a id='H-14'></a>H-14. veRAAC::emergencyWithdraw does not update boost state and voting power state            



## Summary

The emergencyWithdraw function in the veRAAC token contract fails to properly update the Boost state and voting power state when executed. This leads to incorrect protocol boost calculations and governance voting power

## Vulnerability Details

The emergencyWithdraw function is designed to allow users to withdraw their funds in emergency situations. However, the current implementation lacks crucial state updates:

```solidity
function emergencyWithdraw() external nonReentrant {
        //@audit - does not update the boost state and the voting power state
        if (
            emergencyWithdrawDelay == 0 ||
            block.timestamp < emergencyWithdrawDelay
        ) revert EmergencyWithdrawNotEnabled();

        LockManager.Lock memory userLock = _lockState.locks[msg.sender];
        if (userLock.amount == 0) revert NoTokensLocked();

        uint256 amount = userLock.amount;
        uint256 currentPower = balanceOf(msg.sender);

        delete _lockState.locks[msg.sender];
        delete _votingState.points[msg.sender];

        _burn(msg.sender, currentPower);
        raacToken.safeTransfer(msg.sender, amount);

        emit EmergencyWithdrawn(msg.sender, amount);
    }
```

When a user performs an emergency withdrawal, the contract should:

1. Update the Boost state to reflect the reduced staked amount
2. Adjust the user's voting power accordingly
3. The absence of these updates means that the system continues to calculate governance weights and boosts based on outdated information, leading to inaccurate voting power distribution and reward calculations.

## Impact

This vulnerability causes incorrect voting power allocations in governance decisions, incorrect boost calculations for reward distributions. Some users will face unfair second order effects because of this.

## Tools Used

Manual review

## Recommendations

Add Boost state update logic in emergencyWithdraw:
`boostState.updateBoost(msg.sender);`

Also, implement voting power adjustment:
`votingPower.adjustVotingPower(msg.sender);`

## <a id='H-15'></a>H-15. StabilityPool cannot perform Liquidations; Missing Asset Conversion Mechanism            



## Summary

The StabilityPool contract currently lacks the necessary functionality to convert rTokens to crvUSD and handle crvUSD deposits. This oversight renders the protocol's liquidation mechanism non-functional, as liquidations require crvUSD transfers that the system cannot facilitate.

## Vulnerability Details

The vulnerability arises because of the following reasons:

1.The StabilityPool only accepts rToken deposits, with no provision for crvUSD:

```solidity
function deposit(
        uint256 amount
    ) external nonReentrant whenNotPaused validAmount(amount) {
        _update();
        rToken.safeTransferFrom(msg.sender, address(this), amount);
        uint256 deCRVUSDAmount = calculateDeCRVUSDAmount(amount);
        deToken.mint(msg.sender, deCRVUSDAmount);

        userDeposits[msg.sender] += amount;
        _mintRAACRewards();

        emit Deposit(msg.sender, amount, deCRVUSDAmount);
    }
```

2.The stability contract doesn't implement any functions to convert RTokens to crvUSD.

3.Liquidations require crvUSD transfers, but the system has no way to obtain or handle crvUSD:

```solidity
function finalizeLiquidation(
        address userAddress
    ) external nonReentrant onlyStabilityPool {
        //...
        // Transfer reserve assets from Stability Pool to cover the debt
        IERC20(reserve.reserveAssetAddress).safeTransferFrom(
            msg.sender,
            reserve.reserveRTokenAddress,
            amountScaled
        );

        // rest of the code
    }
```

## Impact

This vulnerability has severe consequences for protocol functionality:

1. All liquidation attempts will revert
2. Protocol's liquidation mechanism is non-functional
3. StabilityPool cannot fulfill its core purpose.

## Tools Used

Manual review

## Recommendations

Since the issue arises because the StabilityPool does not have any crvUSD to supply for the liquidations, the following solutions could help solve it:

Add crvUSD deposit functionality to StabilityPool:

```solidity
function depositCrvUSD(uint256 amount) external {
    crvUSD.safeTransferFrom(msg.sender, address(this), amount);
    emit CrvUSDDeposited(msg.sender, amount);
}
```

Implement rToken to crvUSD conversion in RToken:

```solidity
function convertToCrvUSD(uint256 rTokenAmount) external {
    uint256 crvUSDAmount = calculateConversion(rTokenAmount);
    crvUSD.safeTransfer(msg.sender, crvUSDAmount);
    _burn(msg.sender, rTokenAmount);
}
```

PLEASE NOTE: Proper access controls need to be implemented through `require` statements or modifiers for new functions.

## <a id='H-16'></a>H-16. Different precisions used for maxBoost and minBoost in BaseGauge            



## Summary

The BaseGauge contract contains a critical precision inconsistency in its boost parameter initialization. The `maxBoost` and `minBoost` values use different decimal precisions, leading to unintended behavior where `minBoost` is effectively greater than `maxBoost`.

## Vulnerability Details

The issue is located in the BaseGuage constructor:

```solidity
// Initialize boost parameters
boostState.maxBoost = 25000; // 2.5x
boostState.minBoost = 1e18; //@audit - different decimals are used here and also minBoost>maxBoost because of it
```

The problems are:

1. `maxBoost` uses a precision of 1e4 (25000 represents 2.5x)
2. `minBoost` uses a precision of 1e18 (1e18 represents 1x)
3. Due to different precisions, `minBoost` is effectively much larger than `maxBoost`
4. This inconsistency will cause incorrect boost calculations throughout the system.

The following is one instance where these variables are used in calculations:

```solidity
function _applyBoost(
        //@note: audit done
        address account,
        uint256 baseWeight
    ) internal view virtual returns (uint256) {
        if (baseWeight == 0) return 0;

        IERC20 veToken = IERC20(IGaugeController(controller).veRAACToken());
        uint256 veBalance = veToken.balanceOf(account);
        uint256 totalVeSupply = veToken.totalSupply();

        // Create BoostParameters struct from boostState
        BoostCalculator.BoostParameters memory params = BoostCalculator
            .BoostParameters({
                maxBoost: boostState.maxBoost,
                minBoost: boostState.minBoost,
                boostWindow: boostState.boostWindow,
                totalWeight: boostState.totalWeight,
                totalVotingPower: boostState.totalVotingPower,
                votingPower: boostState.votingPower
            });

        uint256 boost = BoostCalculator.calculateBoost(
            veBalance,
            totalVeSupply,
            params
        );

        return (baseWeight * boost) / 1e18;
    }
```

## Impact

This vulnerability has several negative consequences:

1. Boost calculations will produce incorrect results
2. Users may receive incorrect voting power allocations
3. Reward distributions will be inaccurate
4. The protocol's governance mechanism may behave unpredictably
5. Potential economic losses for users due to incorrect boost calculations

## Tools Used

Manual review

## Recommendations

Standardize boost precision to 1e18 throughout the system (and in all the calculations involving minBoost and maxBoost):

```solidity
// Use consistent 1e18 precision
boostState.maxBoost = 2.5e18; // 2.5x
boostState.minBoost = 1e18; // 1x
```

Add validation checks to ensure minBoost <= maxBoost:

```solidity
require(
    boostState.minBoost <= boostState.maxBoost,
    "minBoost must be <= maxBoost"
);
```

## <a id='H-17'></a>H-17. Performance fee is stuck/lost in the GaugeController contract             



## Summary

A calculated 20% performance fee remains permanently stranded/lost in the protocol due to incomplete fee handling logic. While the fee is correctly computed during revenue distribution, there's no mechanism to store or withdraw these funds.

## Vulnerability Details

```solidity
function distributeRevenue(
        GaugeType gaugeType,
        uint256 amount
    ) external onlyRole(EMERGENCY_ADMIN) whenNotPaused {
        if (amount == 0) revert InvalidAmount();

        uint256 veRAACShare = (amount * 80) / 100; // 80% to veRAAC holders
        uint256 performanceShare = (amount * 20) / 100; // 20% performance fee 
//@audit this performance fee is just stuck in the contract

        revenueShares[gaugeType] += veRAACShare;
        _distributeToGauges(gaugeType, veRAACShare);

        emit RevenueDistributed(
            gaugeType,
            amount,
            veRAACShare,
            performanceShare
        );
    }
```

The protocol calculates a 20% performance fee during revenue distribution (distributeRevenue()), but:

1. Fee amounts aren't stored in any storage variables
2. There is no withdrawal/claim functions for these funds
3. Fee value only appears in event emissions

This leaves 20% of every distributed revenue permanently inaccessible.

## Impact

20% of all distributed revenue becomes protocol-owned but unreachable. And if fee collection was intentional, protocol fails to implement core functionality

## Tools Used

Manual review

## Recommendations

Add a state variable and a mechanism to store the fees via the variable. And then create a withdrawal function (with necessary access controls):

```solidity
function withdrawPerformanceFees(address recipient) external onlyRole(TRUSTED_ADMIN) {
    uint256 amount = totalAccumulatedPerformanceFees;
    totalAccumulatedPerformanceFees = 0;
    IERC20(token).transfer(recipient, amount);
    emit PerformanceFeesWithdrawn(recipient, amount);
}
```

## <a id='H-18'></a>H-18. LendingPool's _depositIntoVault always fails due to token flow issue            



## Summary

The `_depositIntoVault` function in the LendingPool contract attempts to deposit tokens from LendingPool contract into the Curve vault, but this operation will fail because by design, the LendingPool contract does not hold any asset tokens.

## Vulnerability Details

The issue occurs because:

1. All user deposits(asset Tokens) in the protocol are held by the RToken contract
2. The LendingPool contract never receives or holds the actual asset tokens
3. The approve and deposit calls will revert due to insufficient token balance in LendingPool

```Solidity
function _depositIntoVault(uint256 amount) internal {
    IERC20(reserve.reserveAssetAddress).approve(address(curveVault), amount);
    curveVault.deposit(amount, address(this));//@audit this will fail as this contract does not have any asset Tokens
    totalVaultDeposits += amount;
}

```

## Impact

1. **Failed Transactions**: All vault deposit operations will revert due to insufficient token balance
2. **Blocked Functionality**: Protocol cannot utilize Curve vault for yield generation

## Recommendations

1. Either Implement a function in RToken contract that performs vault deposits or
2. transfer assets from RToken contract to LendingPool first, then attempt to deposit into vault.

   for example:

```Solidity
function _depositIntoVault(uint256 amount) internal {
    // First transfer from RToken to LendingPool
++    IRToken(reserve.reserveRTokenAddress).transferAsset(address(this), amount);
    
    // Now LendingPool can deposit into vault
    IERC20(reserve.reserveAssetAddress).approve(address(curveVault), amount);
    curveVault.deposit(amount, address(this));
    totalVaultDeposits += amount;
}

```

## <a id='H-19'></a>H-19. Partial Repayments Lost Alongside Full Collateral Seizure During Liquidation            



## Summary

The protocol's liquidation mechanism actively discourages users from making partial repayments during defaults, worsening protocol liquidity risks & increasing bad debt.

When a user repays part of their debt during liquidation, the protocol retains both the partial repayment amount and all NFT collateral, leaving users with no benefit from their repayment efforts.

## Vulnerability Details

When users partially repay debt via \`\_repay()\` but fail to fully clear it before liquidation finalization,the liquidation process unjustly confiscates both partial repayments and full collateral.

```Solidity
// Scenario:
// - User debt: 10 ETH
// - Partial repayment: 3 ETH via _repay()
// - Remaining debt: 7 ETH
// - Liquidation finalizes,StabilityPool pays remaining 7 ETH debt + confiscates all user NFTs
// Result: User loses 3 ETH (repayment) + all NFTs (collateral)
```

The root cause is:

* No tracking of partial repayments (\`user.scaledDebtBalance\` reduction ≠ credit tracking)
* `finalizeLiquidation` seizes collateral without refunding `partial repayments`.

```Solidity
function _repay(uint256 amount, address onBehalfOf) internal {
        //...

        reserve.totalUsage = newTotalSupply;
        user.scaledDebtBalance -= amountBurned;
//rest of the code...

        emit Repay(msg.sender, onBehalfOf, actualRepayAmount);
    }
       function finalizeLiquidation(
        address userAddress
    ) external nonReentrant onlyStabilityPool {
       //...

        // Update user's scaled debt balance
        user.scaledDebtBalance -= amountBurned; //@audit only the remaining debt is paid off after the nft is liquidated, the user loses the partial repayment they made which is not kept track off at all
        reserve.totalUsage = newTotalSupply;

        // rest of the code...

        emit LiquidationFinalized(.....);
    }
```

## Impact

Users suffer **double financial loss**:

* Irrecoverable partial repayments
* Full collateral seizure

**Severity**: **High**

* **Financial Harm**: Direct asset loss to users
* **Systemic Effect**: Discourages partial repayments → Increased bad debt
* **Protocol Reputation Risk**: Erodes user trust in fair treatment

Users lose their partial repayment amounts with no compensation for the payment they made. All NFT collateral is forfeited regardless of owed amount.

## Tools Used

Manual review

## Recommendations

* Implement a repayment credit system:&#x20;

  1\. Track partial repayments in a \`partialRepaymentCredits\` mapping&#x20;

  &#x20; 2\. Refund credits during liquidation finalization:

## <a id='H-20'></a>H-20. _withdrawFromVault function Misdirects Assets to LendingPool, Causing RToken Liquidity Crisis            



## Summary

**Vault Assets Stranded in LendingPool Instead of RToken**

1. `_withdrawFromVault` **Misdirection**:  functions sends assets to LendingPool instead of RToken contract.The protocol's operational design is that all asset tokens are stored in and managed by the RToken contract.  During user deposits and withdrawls, crvUSD is moved in and out  from RToken contract.  

## Vulnerability Details

During user deposits, All assetTokens(crvUSD) are stored in RToken contract.\
Idea is to move assetTokens between `curve-vault and RToken` contract  . However,

`DepositIntoVault`: assetTokens move from : RToken → Vault \
`WithdrawFromVault`: assetTokens move from:  Vault → LendingPool (should go to RToken instead)

`withdrawFromVault`function is used in other functions like `ensureLiquidity`, `rebalanceLiquidity`, to manage assetTokens between curve Vault and Rtoken contract.

```Solidity
 function _withdrawFromVault(uint256 amount) internal {
  //@audit vault funds are going into LendingPool(address(this)),not into RToken contract
    curveVault.withdraw(amount, address(this), msg.sender, 0, new address[](0)); 
    totalVaultDeposits -= amount;
}

```

## Impact

* functions like `_ensureLiquidity`,`rebalanceLiquidity`are no longer useful.

- failed user withdrawls: If funds from vault don't go into RToken contract,eventually RToken contract will not have sufficient balance to process user withdrawls.
- **Stranded Liquidity**: Assets accumulate in LendingPool instead of circulating back to RToken



## Recommendations

Adjust the call to the following:

```Solidity
curveVault.withdraw(
        amount, 
        reserve.reserveRTokenAddress, // ✅ Correct destination
        msg.sender, 
        0, 
        new address[](0)
    );
```

## <a id='H-21'></a>H-21. Unfair Collateral Seizure Despite Full Debt Repayment Due to Manual Liquidation Closure Requirement            



## Summary

The protocol allows collateral to be seized even after a user fully repays their debt if they fail to manually call \`closeLiquidation\`. This design flaw forces users to take an unnecessary extra step to protect their assets, unfairly penalizing compliant borrowers.

## Vulnerability Details

* **Manual Closure Requirement**: Full repayment does not automatically terminate liquidation status, instead users must explicitly call `closeLiquidation` after repayment to avoid collateral seizure.
* Also, `closeLiquidation` is callable only by the user, preventing third-party/bot assistance.
* **Upon Full Repayment of loan, we should have a  Auto-Close Mechanism.**

### Faulty Workflow

1.User Repays Full Debt:

```Solidity
function repay(...) internal {
  user.scaledDebtBalance -= amountBurned; // Debt reduced to zero
  }
```

2.Liquidation Finalization Still Possible:

```Solidity
function finalizeLiquidation(...) external onlyStabilityPool {
// Transfers all collateral to StabilityPool
transferAllCollateral(stabilityPool);
}
```

## Impact

**High Severity** (Direct Financial Loss + Protocol Trust Erosion):

* **Collateral Theft**: Users lose assets despite fulfilling obligations
* **UX Hazard**: Relies on user awareness of obscure manual step
* **Reputational Damage**: Perceived as predatory protocol behavior

## Recommendations:

* Either:  Access Control Expansion

  * Allow trusted keepers (e.g., Chainlink Automation) to call `closeLiquidation`

- Or, Modify the repayment logic to auto-close liquidations when debt is fully repaid:

  ```Solidity
  function \_repay(...) internal {
  // Existing repayment logic
  if (user.scaledDebtBalance == 0) {
  \_closeLiquidation(onBehalfOf); // Auto-trigger closure
  }
  }

  ```

## <a id='H-22'></a>H-22. loan repayments are accepted post grace period            



## Summary

The protocol permits debt repayments after the liquidation grace period expires but before \`finalizeLiquidation\` is executed. This creates a race condition where users lose both their repayment and collateral if liquidated, violating fundamental fairness in debt resolution.

## Vulnerability Details

Key issues:

* **No Repayment Deadline**: Repayments are accepted indefinitely until `finalizeLiquidation` is triggered.
* **Collateral/Repayment Double Loss**: Users lose both despite post-grace repayments
* **Centralization Risk**: StabilityPool managers control liquidation timing

Faulty Workflow

1. **Grace Period Expires**:

```Solidity
require(block.timestamp > liquidationStartTime + gracePeriod, "Grace period ongoing");
```

1. **User Repayment is accepted/allowed post Grace period**:

```Solidity
function repay(...) internal {
// No check for grace period expiration
user.scaledDebtBalance -= amountBurned;
}
```

1. **StabilityPool Finalizes Liquidation**, but pays zero amount as debt is fully paid by user.

```Solidity
function liquidateBorrower(...) external onlyManagerOrOwner {
lendingPool.finalizeLiquidation(userAddress); // Seizes collateral
}
```

## Impact

**Critical Severity** (Protocol Logic Failure + Direct Asset Loss):

* **Permanent Fund Loss**: Users forfeit repayments made after grace period
* **Manipulatable Liquidations**: StabilityPool can delay finalization to maximize penalty
* **Contractual Breach**: Violates implied loan resolution terms

## Recommendations

### 1. Block Post-Grace Repayments

```Solidity
function _repay(...) internal {
if (block.timestamp > liquidationStartTime[onBehalfOf] + gracePeriod) {
revert gracePeriodEnded();
}
// ... existing logic ...
}
```

### 2. Auto-Finalize on Grace Expiry

provide a public function that can be called by anyone/bots.

```solidity
function checkLiquidationStatus(address user) public {
if (block.timestamp > liquidationStartTime[user] + gracePeriod) {
finalizeLiquidation(user);
}
}
```

## <a id='H-23'></a>H-23. Incorrect Debt Comparison in Repayment Capping leading to Liquidations            



## Summary

Exisiting logic in \_repay function, Prevents full debt repayment .

## Vulnerability Details

```Solidity
function _repay(uint256 amount, address onBehalfOf) internal {
...
uint256 userDebt = IDebtToken(reserve.reserveDebtTokenAddress).balanceOf(onBehalfOf);
uint256 userScaledDebt = userDebt.rayDiv(reserve.usageIndex);
uint256 actualRepayAmount = amount > userScaledDebt ? userScaledDebt : amount; 
// Wrong comparison
}

//DebtToken::balanceOf function:
function balanceOf(address account) public view override(ERC20, IERC20) returns (uint256) {
        uint256 scaledBalance = super.balanceOf(account);
        return scaledBalance.rayMul(ILendingPool(_reservePool).getNormalizedDebt());
    }
```

* Compares repayment `amount` (in underlying asset terms) against `userScaledDebt` (scaled value)
* Should compare against unscaled `userDebt` instead
* Present comparison Caps repayments at *scaled debt* (`userScaledDebt`) instead of *actual debt* (`userDebt`)

## Impact

* actualRepayAmount will be capped at userScaledDebt which is far less than userDebt(actual debt of user).
* This will prevent full debt repayment, leading to unnecesary liquidations, loss of user Collateral. 

## Recommendations

fix the following line in the \_repay function:

```Solidity
function _repay(...){
  ..//remaining code

  --  uint256 actualRepayAmount = amount > userScaledDebt ? userScaledDebt : amount;
  ++  uint256 actualRepayAmount = amount > userDebt ? userDebt : amount; //correct comparison

```

## <a id='H-24'></a>H-24.  Under-Collateralization Exploit: Attacker Drains Protocol Funds via Flawed WithdrawNFT function            



## Summary

A critical vulnerability in the `withdrawNFT` function allows attackers to systematically withdraw collateral while retaining excessive debt, leading to under-collateralized positions. By repeating this attack, malicious actors can drain the protocol’s liquidity pool entirely.

&#x20;

## Vulnerability Details

The `withdrawNFT` function's flawed collateral check allows users to withdraw NFT collateral even when it leaves their position under-collateralized.

Root-cause: The formula incorrectly calculates the minimum required collateral as `userDebt × liquidationThreshold`

Insted we should be checking if  `(collateral - NFT value) * liquidationThreshold< userDebt ` and revert withdrawl if true. 

```Solidity
 function withdrawNFT(uint256 tokenId) external nonReentrant whenNotPaused {
   .....
   .....
   //@audit wrong check. 
   if (collateralValue - nftValue < userDebt.percentMul(liquidationThreshold)) {
            revert WithdrawalWouldLeaveUserUnderCollateralized();
        }
   ....
   ...
   }
```

example :

* **current User Debt**: 75 ETH, collateral = 100ETH
* **Liquidation Threshold**: 80% (0.8) 
* User wants to withdraw an NFT worth 20ETH.
* **Current Check**: `(collateral - NFT value) < userDebt * liquidationThreshold)`\
  collateralValue - nftValue = 100 ETH - 20 ETH = 80 ETH\
  userDebt.percentMul(liquidationThreshold) = 75 ETH \* 0.8 = 60 ETH

  if (80 ETH < 60 ETH) → false → withdrawal allowed
* **Resulting Position After Withdrawal : **
  * **Remaining Collateral**: 80 ETH
  * **Maximum Loan Allowed**:\
    80 ETH×0.8=64 ETH80 ETH×0.8=64 ETH
  * **Existing Debt**: 75 ETH (**exceeds** 64 ETH)
  * **Position becomes Undercollateralized**&#x20;

## Impact

* **Direct Fund Loss**
* **Total Drain**: Repeat attacks deplete the liquidity pool.
* **Systemic Collapse**: Protocol becomes insolvent, unable to honor withdrawals.



## Recommendations

Fix the Formula\
Use the correct collateral check to enforce:

```solidity
if ((collateralValue - nftValue).percentMul(liquidationThreshold) < userDebt) {
revert WithdrawalWouldLeaveUserUnderCollateralized();
}
```

## <a id='H-25'></a>H-25. Critical Borrow Vulnerability: flaw in borrow function allows users to borrow more than threshold limit            



## Summary

The `borrow` function in the `LendingPool` contract contains an incorrect collateral sufficiency check, enabling users to borrow amounts that exceed safe collateralization limits.

This flaw allows the creation of under-collateralized loans, threatening protocol solvency.

## Vulnerability Details

```Solidity
function borrow(uint256 amount) external {
// ...
uint256 userTotalDebt = user.scaledDebtBalance.rayMul(reserve.usageIndex) + amount;
  //@audit wrong check
if (collateralValue < userTotalDebt.percentMul(liquidationThreshold)) {
revert NotEnoughCollateralToBorrow();
}
// ...
}
```

Exploit Example:

**Parameters**:

* Collateral: **100 ETH**
* Liquidation Threshold: **80% (0.8)**
* 1st Borrow Attempt: **90 ETH**

**Current Check Execution**:\
collateralValue (100 ETH) < ( 90 ETH) \* 0.8 → 100 < 72 → false(so does not revert) → borrow allowed

**Resulting Position**:

* **Debt/Collateral Ratio**: 90% (90 ETH debt vs. 100 ETH collateral)
* **Required Collateral (Correct)**:\
  $\frac{90\ \text{ETH}}{0.8} = 112.5\ \text{ETH}$
* **Undercollateralized by 12.5 ETH**

## Impact

**Critical Severity** (Protocol Insolvency Risk):

1. **Bad Debt Accumulation**: Users can borrow **more** than collateral allows.
2. **Liquidation Failure**: Undercollateralized positions cannot be profitably liquidated.
3. **Protocol Insolvency**: Total debt exceeds collateral value, leading to systemic collapse.

## Recommendations

### Mitigation Code

change the check to the following

```solidity
++ if (collateralValue.percentMul(liquidationThreshold) < userTotalDebt) {
            revert NotEnoughCollateralToBorrow();
        }
```

    
# Medium Risk Findings

## <a id='M-01'></a>M-01. RToken:: CalculateDust function has critical calculation error            



## Summary

\`transferAccruedDust\` internally calls \`calculateDust\` function to transfer Dust amounts,however \`calculateDust\` has a math error that results in returning dust amount as always zero.

## Vulnerability Details

`totalSupply()` function already returns totalSupply normalized by interest accrual.

```Solidity
function totalSupply() public view override(ERC20, IERC20) returns (uint256) {
        return super.totalSupply().rayMul(ILendingPool(_reservePool).getNormalizedIncome());
```

\
In `totalRealBalance` calculation, applying \`rayMul\` again with the normalized income index  inflates the value. 

```Solidity
function calculateDustAmount() public view returns (uint256) {
        
        uint256 contractBalance = IERC20(_assetAddress).balanceOf(address(this)).rayDiv(ILendingPool(_reservePool).getNormalizedIncome());

        // Calculate the total real obligations to the token holders
        uint256 currentTotalSupply = totalSupply();//; this gives in crvUSD terms

        // Calculate the total real balance equivalent to the total supply
      //@audit multiplying with index again
  uint256 totalRealBalance = currentTotalSupply.rayMul(ILendingPool(_reservePool).getNormalizedIncome()); 
        
        //@audit Due to double multiplication with index in totalRealBalance, it always return 0. 
        return contractBalance <= totalRealBalance ? 0 : contractBalance - totalRealBalance;
    }
```

## Impact

* Due to double multiplication, `contractBalance will always be <= totalRealBalance`, resulting in dust amount as zero. 

- This leads to dust accumulation in the contract, cannot withdraw dust anymore,

## Tools Used

Manual Review

## Recommendations

remove the second rayMul opereation in totalRealBalance calculation.

set totalRealBalance variable to currentTotalSupply.

&#x20;&#x20;

```solidity

 totalRealBalance = currentTotalSupply.

```

## <a id='M-02'></a>M-02. no stale price check in getNFT function of LendingPool            



## Summary

## Vulnerability Details

The `getNFTPrice` function in the lending pool contract retrieves NFT prices **without validating the freshness of the data**. This allows stale prices (e.g., outdated by hours/days) to be used for critical operations like loan issuance and liquidations.

```Solidity
function getNFTPrice(uint256 tokenId) public view returns (uint256) {
        //@audit no stale price check. 
        (uint256 price, uint256 lastUpdateTimestamp) = priceOracle.getLatestPrice(tokenId);
        if (price == 0) revert InvalidNFTPrice();
        return price;
    }
```

## Impact

* **Undercollateralized Loans**: Stale high prices enable borrowers to over-leverage against depreciated collateral.
* **Unjust Liquidations**: Stale low prices trigger incorrect liquidations of solvent positions.
* **Protocol Insolvency Risk**: Mismatch between real NFT values and protocol-reported values.

## Tools Used

## Recommendations

* Add timestamp validation

```Solidity
 ++  require(block.timestamp - lastUpdateTimestamp < STALENESS_THRESHOLD, "Stale price");     return data.price;
```

## <a id='M-03'></a>M-03. Incorrect balanceIncrease calculation in debtToken contract            



## Summary

the burn() function in DebtToken contract, calculates `balanceIncrease` which is part of return tuple.

## Vulnerability Details

`userBalance`is already scaled as `balanceOf()`function already internally multiplies with the index.\
multiplying `userBalance`again in `balanceIncrease`calculation is incorrect, it over inflates the variable wrongly.



```Solidity
function balanceOf(address account) public view override(ERC20, IERC20) returns (uint256) {
        uint256 scaledBalance = super.balanceOf(account);
  // already multiplied with index.
        return scaledBalance.rayMul(ILendingPool(_reservePool).getNormalizedDebt());
    }

function burn(
     address from,
        uint256 amount,
        uint256 index
    ) external override onlyReservePool returns (uint256, uint256, uint256, uint256) {
        if (from == address(0)) revert InvalidAddress();
        if (amount == 0) {
            return (0, totalSupply(), 0, 0);
        }

        uint256 userBalance = balanceOf(from); //@audit already scaled. 

        uint256 balanceIncrease = 0;
        if (_userState[from].index != 0 && _userState[from].index < index) {
            uint256 borrowIndex = ILendingPool(_reservePool).getNormalizedDebt(); //;usage index
           //@audit wrong formula userBalance is already Scaled. 
            balanceIncrease = userBalance.rayMul(borrowIndex) - userBalance.rayMul(_userState[from].index);
            amount = amount;
        }  
```

## Impact

due to double multiplication with index, balanceIncrease variable stores inflated value. this can cause further accounting problem wherever balanceIncrease variable is used. 

## Tools Used

manual review

## Recommendations

adjust the balanceIncrease calculations correctly. divide the userbalance with the index before multiplying with another index.\
for example:

`balanceIncrease = userBalance* (borrowIndex  -  _userState[from].index) / borrowIndex`

## <a id='M-04'></a>M-04. Time-Weighted Average Corruption via Backdated Updates            



## Summary

The TimeWeightedAverage library's `updateValue` function contains a significant design flaw that could enable manipulation of historical data within periods. While the current implementation's usage may not pose an immediate threat, this vulnerability warrants attention due to its potential impact in future implementations

## Vulnerability Details

[TimeWeightedAverage.sol:: UpdateValue()](https://github.com/Cyfrin/2025-02-raac/blob/main/contracts/libraries/math/TimeWeightedAverage.sol#L134)

Current Validation (Insufficient):

`if (timestamp < self.startTime || timestamp > self.endTime) {     revert InvalidTime(); }`

The validation only checks if the timestamp falls within the period bounds.

* Updates can be backdated within a period
* No chronological order enforcement
* Historical data manipulation possible.





## Impact

`Medium Risk - Latent Vulnerability`

* The vulnerability allows backdating updates within a period
* Current implementation's usage limits practical exploitation
* Time-weighted calculations could be corrupted
* Future implementations could be severely impacted if this library is reused

While current usage patterns don't expose critical vulnerabilities, the design flaw represents a "time bomb" that could be exploited in future implementations. The library's fundamental role in time-weighted calculations makes this a significant concern for protocol security and data integrity.

## Tools Used

Manual review

## Recommendations
add an additional check:

```Solidity
  +     if (timestamp <= self.lastUpdateTime) { 
  +     revert InvalidUpdateTime();  
  +     }
```

## <a id='M-05'></a>M-05. Time-Weighted Average Calculations Incorrect When Querying Historical Averages            



## Summary

The `calculateAverage` function in the TimeWeightedAverage library fails to validate that the queried timestamp occurs after the most recent value update (`lastUpdateTime`). This allows querying historical averages while incorrectly including future updates in the calculation, corrupting time-weighted average.

## Vulnerability Details

The vulnerability stems from the `calculateAverage` function including future updates when calculating historical averages. This violates the fundamental principle of time-weighted average calculations which should only consider values known at the query timestamp.

while current protocol implementations are not directly affected due to strict sequential updates, this vulnerability in the library's core logic poses a significant risk for future protocol components or modifications that might rely on historical average calculations, potentially leading to corrupted time-weighted values and economic exploits.

## Impact

This bug affects  contracts that will be using TimeWeightedAverage -calculateAverage function. for example: 

* FeeCollector - Incorrect reward distributions
* GaugeController - Corrupted gauge weights&#x20;
* RAAC/RWA Gauges - Invalid emission calculations &#x20;

## POC:

```Solidity
function test_calculateAverage() public{
       /* 
       |-------------|-------------------|----------|
       1sec    v1   11sec     v2      31sec  v3   51sec
       
       v1=100,v2=200,v3=300
       
       Timeline: t=1 (100) → t=11 (200) → t=31 (300)  
       
       calculateAverage at t=21sec. 
       average should be = [200*(21-11) + 100*(11-1)]/(21-1) = (2000+1000)/20 =150
        */
  
       //create period
        period.createPeriod(block.timestamp, 50, 100, 1e18);
        
        vm.warp(11);
        period.updateValue(200,block.timestamp);
        assertEq(period.value,200);//test pass
       
        vm.warp(31);
        period.updateValue(300,block.timestamp);
        assertEq(period.value,300);//test pass
       
        uint avg=period.calculateAverage(21);
        console.log("average is ",avg);//avg comes to be 250
        assertEq(avg,150);//fails 
    }
```

## Recommendations

* Can add timestamp validation and revert for historical queries

```Solidity
++ require( timestamp >= self.lastUpdateTime,
          "TWAV: Timestamp precedes last update" 
         );
```

* Or enable historical queries by using  a snapshot mechanism to Implement historical tracking for legitimate past queries



  ```Solidity
  struct Snapshot { 
    uint256 timestamp; 
    uint256 value; 
  }
  Snapshot[] public history;
  ```


## <a id='M-06'></a>M-06. Missing pause control mechanism in veRAACToken contract            



## Summary

The veRAACToken contract implements a pause modifier but provides no way to actually pause the contract, creating a false sense of security and preventing legitimate emergency protocol freezing.

## Vulnerability Details

The contract declares but does not implement critical pause controls:

```solidity
modifier whenNotPaused() {
        //@audit - there is no mechanism to set this bool paused variable.
        if (paused) revert ContractPaused();
        _;
    }
```

The veRAAC contract declares a paused variable and defines the above modifier, but it does not provide a function/mechanism to set the value of the bool `paused`.

So, because of this, the whenNotPaused modifier can never activate as paused remains uninitialized (default false) with no way to change state.Protocol admins cannot freeze operations during security incidents

## Impact

The lack of pause functionality leaves the protocol vulnerable during emergencies, as there's no way to halt operations if a critical issue arises. This oversight undermines the intended safety mechanism and could lead to significant financial and reputational damage if exploited.

## Tools Used

Manual review

## Recommendations

Add pause control functions:

```solidity
function pause() external onlyOwner {
    _pause();
}

function unpause() external onlyOwner {
    _unpause();
}
```

## <a id='M-07'></a>M-07. Incorrect emission cap comparison while setting emissions in BaseGauge            



## Summary

The `setEmission` function in BaseGauge contains a logic error where it compares the new emission amount with the current emission instead of the maximum allowed emission (`maxEmission`).&#x20;

## Vulnerability Details

The issue is in the `setEmission` function of the BaseGuage contract:

```solidity
function setEmission(uint256 emission) external onlyController {
    // Incorrect comparison with current emission instead of maxEmission
    if (emission > periodState.emission) revert RewardCapExceeded();
    periodState.emission = emission;
    emit EmissionUpdated(emission);
}
```

The function compares new emission with periodState.emission instead of maxEmission. So, even if the protocol wants to increase emissions, it can never be done. Since no new emission can be higher than the past ones because of the incorrect variable used. 

## Impact

The protocol will not be able to set the desired emission as trying to increase emissions from the periodState.emission will always revert. The protocol will not function as intended.

## Tools Used

Manual review

## Recommendations

Update the comparison to use `maxEmission`:

```solidity
function setEmission(uint256 emission) external onlyController {
    if (emission > maxEmission) revert RewardCapExceeded();
    periodState.emission = emission;
    emit EmissionUpdated(emission);
}
```

## <a id='M-08'></a>M-08. Incorrect voting power calculation in `GaugeController::function vote` ignores voting power decay             



## Summary

The GaugeController's voting mechanism miscalculates user voting power by using static token balances instead of time-decayed voting power, enabling users to retain full voting weight indefinitely regardless of lock duration.

## Vulnerability Details

When a user wants to make changes to a specific guage's weight, they call the following function:

```solidity
function vote(
        address gauge,
        uint256 weight
    ) external override whenNotPaused {
        if (!isGauge(gauge)) revert GaugeNotFound();
        if (weight > WEIGHT_PRECISION) revert InvalidWeight();

        uint256 votingPower = veRAACToken.balanceOf(msg.sender); //@audit this assumes voting power doesn't decay
        if (votingPower == 0) revert NoVotingPower();

        uint256 oldWeight = userGaugeVotes[msg.sender][gauge];
        userGaugeVotes[msg.sender][gauge] = weight;

        _updateGaugeWeight(gauge, oldWeight, weight, votingPower);

        emit WeightUpdated(gauge, oldWeight, weight);
    }
```

The `veRAACToken` contract implements time-decayed voting power through `getVotingPower()`, but the GaugeController incorrectly uses the static `balanceOf()` method. This returns the raw token amount without accounting for voting power decay over the lock duration.

This violates the protocol's core governance mechanism where voting power should diminish as locks approach expiration.

## Impact

Users can maintain maximum voting power longer than what the protocol intends. The protocols intends for the votes to decay over the duration of the lock.

## Tools Used

Manual review

## Recommendations

Replace balanceOf with proper voting power calculation:

```solidity
uint256 votingPower = veRAACToken.getCurrentVotes(msg.sender);
```


# Low Risk Findings

## <a id='L-01'></a>L-01. `getLockedBalance` function retrieves incorrect data in veRAACToken contract            



## Summary

The `getLockedBalance` function in the `veRAACToken` contract incorrectly references the locks mapping instead of the `_lockState.locks` mapping, leading to inaccurate balance readings for locked tokens.

## Vulnerability Details

The function `getLockedBalance` is designed to return the amount of RAAC tokens locked by a specific account.

However, it retrieves this data from the locks mapping, which is unused in the contract. The correct data source should be the `_lockState.locks` mapping, which contains the actual lock state. This discrepancy causes the function to return stale or zero values, even when valid locks exist in `_lockState`.

```solidity
/**
     * @notice Gets the amount of RAAC tokens locked by an account
     * @dev Returns the raw locked token amount without time-weighting
     * @param account The address to check
     * @return The amount of RAAC tokens locked by the account
     */
    function getLockedBalance(address account) external view returns (uint256) {
        //@audit - The function is incorrectly referencing the locks mapping when it should be using the _lockState.locks mapping
        return locks[account].amount;
    }
```

## Impact

This bug can lead to incorrect reporting of locked balances. Users may be unable to verify their locked balances correctly, leading to confusion and potential loss of trust in the protocol.

## Tools Used

Manual review

## Recommendations

Update the function to reference `_lockState.locks` instead of `locks`:

```solidity
function getLockedBalance(address account) external view returns (uint256) {
    return _lockState.locks[account].amount;
}
```

## <a id='L-02'></a>L-02. Incorrect lock expiry data retrieved in `veRAACToken::getLockEndTime` function            



## Summary

The `getLockEndTime` function in the `veRAACToken` contract is flawed as it retrieves lock expiration data from the wrong storage location, resulting in incorrect lock end times being returned.

## Vulnerability Details

The `getLockEndTime` function is intended to return the timestamp when a user's lock expires.

```solidity
/**
     * @notice Gets the lock end time for an account
     * @dev Returns the timestamp when the lock expires
     * @param account The address to check
     * @return The unix timestamp when the lock expires
     */
    function getLockEndTime(address account) external view returns (uint256) {
        //@audit - The function retrieves wrong data, it should be using the _lockState.locks mapping
        return locks[account].end;
    }
```

However, it incorrectly queries the locks mapping, which is not actively used in the contract. The correct data source is the `_lockState.locks` mapping, which holds the actual lock state. This error causes the function to return outdated or zero values, even when valid locks exist in `_lockState`.

## Impact

This issue can mislead users about their lock expiration times, potentially causing them to miss opportunities to extend or withdraw their locks. It could also disrupt governance processes, as users may base their decisions on incorrect lock duration information.

## Tools Used

Manual review

## Recommendations

Modify the function to use `_lockState.locks` instead of `locks`:

```solidity
function getLockEndTime(address account) external view returns (uint256) {
    return _lockState.locks[account].end;
}
```



