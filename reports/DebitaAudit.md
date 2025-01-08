> zkillua
> 
> # High 01
> 
> # `extendedTime` calculation in `DebitaV3Loan::extendLoan` shall Cause denial of Service, due to overflow-underflow error.
> ### Summary
> The `extendLoan` function in `DebitaV3Loan` contract contains a flawed calculation for `extendedTime` that causes arithmetic underflow, making the loan extension feature unusable for certain duration combinations. The double subtraction of `block.timestamp` in the calculation leads to arithmetic underflow. The bug effectively creates a denial of service for loan extensions under common and valid loan scenarios, making it critical to fix.
> 
> ### Root Cause
> [In DebitaV3Loan.sol](https://github.com/sherlock-audit/2024-11-debita-finance-v3/blob/main/Debita-V3-Contracts/contracts/DebitaV3Loan.sol#L590) `extendedTime` calculation causes arithmetic errors leading to revert.
> 
> ```solidity
> uint alreadyUsedTime = block.timestamp - m_loan.startedAt;
> uint extendedTime = offer.maxDeadline - alreadyUsedTime - block.timestamp;
> ```
> 
> The extendedTime calculation is incorrect. It subtracts the current timestamp twice:
> 
> * Once through alreadyUsedTime (which includes block.timestamp - startedAt)
> * Again directly with block.timestamp
> 
> This creates a mathematical impossibility for many valid timestamp combinations, causing arithmetic underflow.
> 
> (https://github.com/sherlock-audit/2024-11-debita-finance-v3/blob/main/Debita-V3-Contracts/contracts/DebitaV3Loan.sol#L590)
> 
> ### Internal pre-conditions
> _No response_
> 
> ### External pre-conditions
> _No response_
> 
> ### Attack Path
> * Borrower takes a 60-day loan with lenders offering 100 and 111-day max durations
> * At day 55, borrower tries to extend the loan
> 
> ### Impact
> * Loan Extension fails due to arithmetic underflow
> * Borrower loses ability to extend loan despite being within valid timeframes
> * Could affect multiple loans with similar duration patterns
> * The bug effectively creates a denial of service for loan extensions under common and valid loan scenarios, making it critical to fix.
> 
> ### PoC
> In `TwoLendersERC20Loan.t.sol`:
> 
> Set initial borrower duration to : `5184000` in `setUp()`
> 
> ```solidity
> address borrowOrderAddress = DBOFactoryContract.createBorrowOrder(
>             oraclesActivated,
>             ltvs,
>             1400,
>             5184000, //864000,
>             acceptedPrinciples,
>             USDC,
>             false,
>             0,
>             oraclesPrinciples,
>             ratio,
>             address(0x0),
>             10e18
>         );
> ```
> 
> and Add the following test function
> 
> ```solidity
> function testExtendLoan_underflow() public {
>     
>     // initialDuration = 60days= 5184000;  // 10 days
>         matchOffers();
> 
>     // first lender:  maxDeadline1 = 8640000;   // 100 days
>     // second lender: maxDeadline2 = 9640000;   // ~111 days
>     
>     // Warp to day 55
>     vm.warp(block.timestamp + 55 days); 
>     
>     // Try to extend loan
> 
>     vm.startPrank(borrower);
>     AEROContract.approve(address(DebitaV3LoanContract), type(uint256).max);    
>     // This call will revert with arithmetic overflow- underflow error. 
>     vm.expectRevert();  
>     DebitaV3LoanContract.extendLoan();
>     vm.stopPrank();
> }
> ```
> 
> `run : forge test --mt testExtendLoan_underflow`
> 
> ### Mitigation
> _No response_

> 
> # High-02 
> 
> # `DebitaPyth::getThePrice` function returns Price without accounting for `exponent` of `priceData`
> ### Summary
> In `getThePrice` [function](https://github.com/sherlock-audit/2024-11-debita-finance-v3/blob/main/Debita-V3-Contracts/contracts/oracles/DebitaPyth.sol#L25) of DebitaPyth contract, `pyth.getPriceNoOlderThan()` is used which returns a `Price struct` that contains the `price , conf, expo and publishTime.` The function does not take into account the exponent returned and proceeds to return `priceData.price` without the `exponent` taken into account. `priceData.price` is in `fixed-point numeric representation`, it should be multiplied by `10 ** expo`.
> 
> ### Root Cause
> In [DebitaPyth](https://github.com/sherlock-audit/2024-11-debita-finance-v3/blob/main/Debita-V3-Contracts/contracts/oracles/DebitaPyth.sol#L25)
> 
> ```solidity
> function getThePrice(address tokenAddress) public view returns (int) {
>         // falta hacer un chequeo para las l2
>         bytes32 _priceFeed = priceIdPerToken[tokenAddress];
>         require(_priceFeed != bytes32(0), "Price feed not set");
>         require(!isPaused, "Contract is paused");
> 
>         // Get the price from the pyth contract, no older than 90 seconds
>         PythStructs.Price memory priceData = pyth.getPriceNoOlderThan(
>             _priceFeed,
>             600
>         );
> 
>         // Check if the price feed is available and the price is valid
>         require(isFeedAvailable[_priceFeed], "Price feed not available");
>         require(priceData.price > 0, "Invalid price");
>  @--->   return priceData.price; //@audit should be multipled by 10**expo
>     }
> ```
> 
> contract PythStruct has Price struct as follows:
> 
> ```solidity
> struct Price {
>         // Price
>         int64 price;
>         // Confidence interval around the price
>         uint64 conf;
>         // Price exponent
>         int32 expo;
>         // Unix timestamp describing when the price was published
>         uint publishTime;
>     }
> ```
> 
> As mentioned in [Pyth network Docs](https://docs.pyth.network/price-feeds/best-practices#fixed-point-numeric-representation) and [API reference](https://api-reference.pyth.network/price-feeds/evm/getPriceNoOlderThan): `The integer representation of price value can be computed by multiplying by 10^exponent`
> 
> ### Internal pre-conditions
> _No response_
> 
> ### External pre-conditions
> _No response_
> 
> ### Attack Path
> _No response_
> 
> ### Impact
> DebitaV3Aggregator, has`getPriceFrom` which calls `getThePrice` function of the Oracle. whenever the oracle is Pyth, the returned price will be inaccurate, which results in incorrect calculations of principle prices, collateral prices etc.
> 
> ### PoC
> _No response_
> 
> ### Mitigation
> Apply the returned expo of the Price struct to the price.
> 
> ```solidity
> function getThePrice(address tokenAddress) public view returns (int) {
>         // falta hacer un chequeo para las l2
>         bytes32 _priceFeed = priceIdPerToken[tokenAddress];
>         require(_priceFeed != bytes32(0), "Price feed not set");
>         require(!isPaused, "Contract is paused");
> 
>         // Get the price from the pyth contract, no older than 90 seconds
>         PythStructs.Price memory priceData = pyth.getPriceNoOlderThan(
>             _priceFeed,
>             600
>         );
> 
>         // Check if the price feed is available and the price is valid
>         require(isFeedAvailable[_priceFeed], "Price feed not available");
>         require(priceData.price > 0, "Invalid price");
>  -     return priceData.price; //@audit should be multipled by 10**expo
>  ++  int256 priceFinal = priceData.expo >= 0 ? (priceData.price * 10 ** priceData.expo) : (priceData.price / 10**(-priceData.expo));
> 
>  +   return priceFinal
> 
>   }
> ```



> 
> 
> # Medium 01
> 
> # Variable Shadowing in `ChangeOwner` Function of multiple contracts Prevents Ownership Transfer
> ### Summary
> The `changeOwner` function is used in 3 different contracts:
> 
> * [debitaV3Aggregator.sol: 683](https://github.com/sherlock-audit/2024-11-debita-finance-v3/blob/1465ba6884c4cc44f7fc28e51f792db346ab1e33/Debita-V3-Contracts/contracts/DebitaV3Aggregator.sol#L682),
> * [AuctionFactory.sol:219](https://github.com/sherlock-audit/2024-11-debita-finance-v3/blob/1465ba6884c4cc44f7fc28e51f792db346ab1e33/Debita-V3-Contracts/contracts/auctions/AuctionFactory.sol#L218),
> * [buyOrderFactory.sol: 186](https://github.com/sherlock-audit/2024-11-debita-finance-v3/blob/main/Debita-V3-Contracts/contracts/buyOrders/buyOrderFactory.sol#L186),
> 
> The function is all three instances is unusable and does not update the `owner` state variable due to `variable shadowing`.
> 
> * The authorization check `require(msg.sender == owner)`  validates `msg.sender` against the input parameter `owner ` instead of the state variable `owner`, making it impossible for the current owner to call the function (with a new owner address).
> * The local function parameter "owner" `shadows the state variable`, causing the assignment statement to have no effect on the intended state variable.
>   As a result, `the owner of the contract cannot be changed using this function.`
> 
> ### Root Cause
> `changeOwner` function of three contracts:
> 
> ```solidity
>  @-->  function changeOwner(address owner) public { //@audit : input param has same name as state var
>         require(msg.sender == owner, "Only owner");
>           ......
>            .......
>         owner = owner;
>     }
> ```
> 
> The Input parameter `owner` in `changeOwner` has the same name as the state variable `owner` of the contract. Within the function, the local parameter takes precedence, effectively "hiding" the state variable. - so everytime `owner` calls `changeOwner` function with a new owner address as input argument, the `require` statement does not pass. - also, the statement `owner = owner; ` only operates on the local parameter and does not modify the state variable.
> 
> ### Internal pre-conditions
> _No response_
> 
> ### External pre-conditions
> _No response_
> 
> ### Attack Path
> 1.Owner has to call `changeOwner` function .
> 
> ### Impact
> * After Initial deployment,  any attempt to transfer/change ownership fails.
> 
> ### PoC
> In file `BasicDebitaAggregator.t.sol`
> 
> add the following test function in `DebitaAggregatorTest` contract:
> 
> ```solidity
> function testChangeOwnerBug() public {
>     // Get initial owner (deployer/test contract)
>     address initialOwner = address(this);
>     address newOwner = address(0x123);
> 
>    
>     
>     //   - should fail because msg.sender is checked against input parameter , not original owner
>     vm.prank(initialOwner);
>     vm.expectRevert("Only owner");  // Reverts because msg.sender != owner parameter
>     DebitaV3AggregatorContract.changeOwner(newOwner);
> 
>    
> }
> ```
> 
> ### Mitigation
> Use a different variable name for input param instead of `owner`
> 
> ```solidity
> function changeOwner(address _newOwner) public {
>     require(msg.sender == owner, "Only owner");  // checks against state variable
>     require(deployedTime + 6 hours > block.timestamp, "6 hours passed");
>     owner = _newOwner;  // proper assignment
> }
> ```

> 
> 
> # Low 01
> 
> # Race Condition at Loan Deadline Creates Temporary DOS
> ### Summary
> The DebitaV3Loan contract contains inconsistent deadline checks between `payDebt` and `claimCollateralAsLender` functions, creating a race condition at the exact deadline timestamp where neither borrower nor lender can interact with the loan.
> 
> ### Root Cause
> In [`payDebt`](https://github.com/sherlock-audit/2024-11-debita-finance-v3/blob/main/Debita-V3-Contracts/contracts/DebitaV3Loan.sol#L186) function of DebitaV3loan contract:
> 
> ```solidity
> require(nextDeadline() >= block.timestamp, "Deadline passed to pay Debt");
> ```
> 
> In [claimCollateralAsLender](https://github.com/sherlock-audit/2024-11-debita-finance-v3/blob/main/Debita-V3-Contracts/contracts/DebitaV3Loan.sol#L340) function:
> 
> ```solidity
> require(_nextDeadline < block.timestamp && _nextDeadline != 0, "Deadline not passed");
> ```
> 
> Inconsistent deadline comparison operators
> 
> * `>=`  in payDebt prevents payment at exact deadline
> * `<` in claimCollateralAsLender prevents claim at exact deadline
>   No clear specification of who should have priority at deadline
> 
> ### Internal pre-conditions
> _No response_
> 
> ### External pre-conditions
> _No response_
> 
> ### Attack Path
> _No response_
> 
> ### Impact
> When block.timestamp == nextDeadline():
> 
> * Borrower's payment reverts because nextDeadline() >= block.timestamp fails
> * Lender's claim reverts because _nextDeadline < block.timestamp fails
> 
> Loan enters temporary deadlock:
> 
> * Temporary denial of service at exact deadline timestamp.
> * Neither borrower can repay nor lender can claim collateral
> * Affects core loan functionality at critical moment
> 
> ### PoC
> _No response_
> 
> ### Mitigation
> ```solidity
> // Option 1: Give borrower priority at deadline
> 
> 
> function payDebt() {
>     require(nextDeadline() > block.timestamp, "Deadline passed");
> }
> 
> function claimCollateralAsLender() {
>     require(_nextDeadline <= block.timestamp, "Deadline not passed");
> }
> 
> // Option 2: Give lender priority at deadline
> function payDebt() {
>     require(nextDeadline() >= block.timestamp, "Deadline passed");
> }
> 
> function claimCollateralAsLender() {
>     require(_nextDeadline <= block.timestamp, "Deadline not passed");
> }
> ```


> 
> 
> # Low 02
> # Array Out of Bounds error in `getActiveBuyOrders` of `buyOrderFactory` contract Causes DOS
> ### Summary
> The `getActiveBuyOrders` function in `buyOrderFactory` contract contains two critical array-related errors:
> 
> * Incorrect array size calculation using wrong variable ("limit - offset")
> * Array `index out-of-bounds` error in loop iteration. ("i < offset + limit")
>   This can cause denial of service for buy order retrieval functionality in most cases
> 
> ### Root Cause
> In BuyOrderfactory contract [getActiveBuyOrders](https://github.com/sherlock-audit/2024-11-debita-finance-v3/blob/main/Debita-V3-Contracts/contracts/buyOrders/buyOrderFactory.sol#L139) function,
> 
> ```solidity
>  function getActiveBuyOrders(
>         uint offset,
>         uint limit
>     ) public view returns (BuyOrder.BuyInfo[] memory) {
>         uint length = limit;
> 
>         if (limit > activeOrdersCount) {
>             length = activeOrdersCount;
>         }
> 
>         BuyOrder.BuyInfo[] memory _activeBuyOrders = new BuyOrder.BuyInfo[](
>  @-->           limit - offset         //@audit should be "length - offset"
>         );
> 
>   @--->      for (uint i = offset; i < offset + limit; i++) {
>             address order = allActiveBuyOrders[i]; //@audit can result in out of bounds error
>             _activeBuyOrders[i] = BuyOrder(order).getBuyInfo();
>         }
>         return _activeBuyOrders;
>     }
> ```
> 
> Error 1: Array Size Calculation:
> 
> * Uses limit - offset instead of length - offset
> * limit could be larger than activeOrdersCount
> * Creates array larger than available items
> 
> Error 2: for loop condition `i<offset+limit` where i starts from `offset`:
> 
> * offset + limit value can exceed activeOrdersCount value, upon which , function throws `Error: "Index out of bounds"`
> 
> ### Internal pre-conditions
> _No response_
> 
> ### External pre-conditions
> _No response_
> 
> ### Attack Path
> _No response_
> 
> ### Impact
> * denial of service for buy order retrieval functionality
> * Function reverts on  call majority of times
> * Protocol's order discovery mechanism is broken
> 
> ### PoC
> Add the following test function in `BuyOrder.t.sol` :
> 
> ```solidity
> function testGetActiveBuyOrdersRevert() public {
>     // Create multiple buy orders
>     deal(AERO, buyer, 1000e18 * 10, false);  // Fund for 10 orders
>     vm.startPrank(buyer);
>     AEROContract.approve(address(factory), 1000e18 * 10);
>     
>     // Create 10 buy orders
>     for(uint i = 0; i < 10; i++) {
>         factory.createBuyOrder(
>             AERO,
>             address(receiptContract),
>             100e18,
>             7e17
>         );
>     }
>     vm.stopPrank();
> 
>      vm.expectRevert();
>     factory.getActiveBuyOrders(2, 8);
> }
> ```
> 
> Run: - anvil: anvil --fork-url https://mainnet.base.org --fork-block-number 21151256 - forge test --mt testGetActiveBuyOrdersRevert --fork-url http://localhost:8545 -vvvv
> 
> ### Mitigation
> ```solidity
> function getActiveBuyOrders(uint offset, uint limit) public view returns (BuyOrder.BuyInfo[] memory) {
>     require(offset <= activeOrdersCount, "Invalid offset");
>     uint length = limit;
>     if (limit > activeOrdersCount) {
>         length = activeOrdersCount;
>     }
>     
>     @--> BuyOrder.BuyInfo[] memory _activeBuyOrders = new BuyOrder.BuyInfo[](length - offset); //fixed
>     @--> for (uint i = 0; i < length - offset; i++) {
>      @-->   address order = allActiveBuyOrders[offset + i];
>         _activeBuyOrders[i] = BuyOrder(order).getBuyInfo();
>     }
>     return _activeBuyOrders;
> }
> ```
> 
> Above fix ensure:
> 
> * Proper array size calculation using capped length
> * Correct array indexing relative to offset
> * No out-of-bounds access
> * Function remains usable

