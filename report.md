# Audit report for BradyCoin
February 5 2018


## Summary

Audit Report prepared for BradyCoin covering the collectible contract.


## Scope of the audit

The audit was based on the solidity compiler *0.4.19+commit.c4cbbb05*. The scope of the audit is limited to the following source code file

1. BradyCoin.sol


## Critical

### 1. Unsafe arithmetic operations

Unsafe arithmetic operations are carried out across the contract. This should be avoided to prevent integer underflow and overflow.

```
balanceOf[seller]--;
balanceOf[bid.bidder]++;
```

**Recommendation**

Make use of [SafeMath](https://github.com/OpenZeppelin/zeppelin-solidity/blob/master/contracts/math/SafeMath.sol) library by OpenZepplin. It strictly checks for integer overflow and underflow conditions which can potentially prevent loss of funds. Suggested code change is as follows

```
// Import SafeMath
import "./SafeMath.sol";

// Use safemath for uint
using SafeMath for uint;

// Perform arithmetic operations using SafeMath
balanceOf[seller] = balanceOf[seller].sub(1);
balanceOf[bid.bidder] = balanceOf[bid.bidder].add(1);
```

## Minor

### 2. Bidding can be restricted by anyone

Anyone can reset the Bid array making the Bid functionality unusable. This can be performed by placing a bid higher than the current bid and withdrawing it. Placing a higher bid removes the current bid and withdrawing it empties the bid array. 

**Recommendation**

An option to raise a request for withdrawal instead of actually withdrawing the bid immediately.

There are several ways to overcome this situation some with less flexibility for the user. It should be well thought before implementing it.


### 3. *bradyNoLongerForSale* can affect the bids placed

If the seller decides to make the coin no longer available for sale, the bid placed still remains.

**Recommendation**

Transfer the funds placed in bids to the bidder when the owner decides to take back the selling offers.


### 4. Older solidity version

Older compilers might be susceptible to some bugs. List of known compiler bugs and their severity can be found [here](https://etherscan.io/solcbuginfo).

**Recommendation**

It is always suggested to use the latest stable version of Solidity.

```
pragma solidity ^0.4.19;
```


### 5. Redundant code makes the contract more expensive to call

Redundant code is used throughout the contract which will make the contract deployment and function call more expensive. It may be negligible in short term, but when considering the lifetime of the contract, this can collectively become large.

An example of such redundant code is

```
if (msg.value < minBidIncrement) revert();
uint minBidValue = existing.value + minBidIncrement;
if (existing.hasBid){
    if (msg.value < minBidValue) revert();
}
if (msg.value <= existing.value) revert();
```

**Recommendation**

Remove redundant code to decrease the gas cost. It can potentially save the amount that is used just to execute the functions. The above mentioned code can be replaced with

```
require(msg.value >= existing.value + minBidIncrement);
```
A number of recommendations to save gas is also listed in the suggestions. 

## Suggestion

### 6. Multiple loops can be avoided to save gas

Constructor function contains multiple loops and duplicate code which makes the contract deployment more expensive.

**Recommendation**

Suggestions are given as comments

```
function BradyCoin() public payable {
    owner = msg.sender; // Make use of Ownable.sol which will handle ownership. 
    totalSupply = 12;                                  
    name = "BradyCoins";
    symbol = "BRAϾ";
    decimals = 0;
    setInitialOwners(); // Contains the first loop that iterates from 0 to 11
    for (uint i=0; i < totalSupply; i++) { // Second loop to iterate from 0 to 11
        bradyBids[i] = Bid(false, i, 0x0, 0); 
        bradyNoLongerForSale(i);
    }
}

function setInitialOwner(address toAddress, uint bradyIndex) private {
    if (msg.sender != owner) revert(); // Can be removed since it is called from constructor.
    if (bradyIndex >= totalSupply) revert(); // Can be removed since index is handled by for loop
    if (bradyIndexToAddress[bradyIndex] != toAddress) { // Always true when invoked from the constructor
        if (bradyIndexToAddress[bradyIndex] != 0x0) { // Always false when invoked from the constructor
            balanceOf[bradyIndexToAddress[bradyIndex]]--; // Unreachable code
        }
        bradyIndexToAddress[bradyIndex] = toAddress;
        balanceOf[toAddress]++;  // Can be assigned the total supply.
    }
}

function setInitialOwners() private {
    if (msg.sender != owner) revert(); // Can be removed since it is called from constructor.
    for (uint i = 0; i < totalSupply; i++) { // First for loop. can be merged.
        setInitialOwner(msg.sender, i);
    }
}
```

### 7. Raise Event after completing the operation

It is suggested to raise an event after completing the task. It prevents the event listener from performing other operations before the task is actually completed.

```
BradyBidWithdrawn(bradyIndex, bid.value, msg.sender);
bradyBids[bradyIndex] = Bid(false, bradyIndex, address(0), 0);
msg.sender.transfer(bid.value);
```

**Recommendation**

The event is raised even before the amount is transferred to the bidder or the bid array is reset. This can be modified as follows. Follow the similar structure for all events.

```
bradyBids[bradyIndex] = Bid(false, bradyIndex, address(0), 0);
msg.sender.transfer(bid.value);
BradyBidWithdrawn(bradyIndex, bid.value, msg.sender);
```

### 8. Use modifiers
Modifiers are the standard to check eligibility for function call. Make use of modifiers to check admin rights and withdraw status.

**Recommendation**

Create a modifier to check for withdraw status. The same patterns can be followed for others.

```
// modifier declaration
modifier restrictWithdrawal() {
    require(canWithdraw);
    _;
}

// modifer usage
function withdrawBidForBrady(uint bradyIndex) restrictWithdrawal public {}
```

### 9. Use memory instead of storage

It is suggested to use `memory` for declaring local variables inside functions. Using `storage` is not necessary unless you are trying to modify the solidity state. `memory` also consumes less gas when comparing with `storage`.

```
Bid storage bid = bradyBids[bradyIndex];
Offer storage offer = bradyIsOfferedForSale[bradyIndex];
```

**Recommendation**

Use `memory` during local variable declaration.

```
Bid memory bid = bradyBids[bradyIndex];
Offer memory offer = bradyIsOfferedForSale[bradyIndex];
```

### 10. Make use of require

Use `require` for checking condition inside functions. This helps in better code readability and follows solidity best practices.

```
if (bradyIndex >= totalSupply) revert();
if (msg.sender == bradyIndexToAddress[bradyIndex]) revert();
```

**Recommendation**

The `if` `revert` conditions can be replaced with `require`

```
require(bradyIndex < totalSupply);
require(msg.sender != bradyIndexToAddress[bradyIndex]);
```

### 11. Use standard libraries like Ownable.sol

There are standard libraries available for handling tasks in a smart contract. There libraries have well tested code that is used in many contracts.

**Recommendation**

Ownable contract from OpenZepplin can be used to handle the ownership of the contract and restrict access.

```
// Import ownable.sol
import "./Ownable.sol";

// Use modifer to restrict access
function acceptAllBids() onlyOwner public { }
```

### 12. Use address(0)

For checking invalid empty address, address(0) is the more standard approach.

**Recommendation**

```
require(bradyIndexToAddress[bradyIndex] != address(0));
```

### 13. Inconsistency in variable declaration

Follow consistency when assigning a value to a variable and actually using it.

```
if (bradyIndexToAddress[bradyIndex] != msg.sender) revert();
address seller = msg.sender;
BradyBought(bradyIndex, bid.value, seller, bid.bidder);
```
**Recommendation**

```
require(bradyIndexToAddress[bradyIndex] == msg.sender);
BradyBought(bradyIndex, bid.value, msg.sender, bid.bidder);
```

### 14. Consider replacing `mapping` with `array`

Since the index is from 0 to totalSupply, array can also be used to achieve the functionality. Arrays can reduce the gas amount and provides more features like returning the whole array using a view function.

```
mapping (uint => address) public bradyIndexToAddress;
mapping (uint => Offer) public bradyIsOfferedForSale;
mapping (uint => Bid) public bradyBids;
```

**Recommendation**

It also helps restrict access to values greater than totalSupply.

```
uint256 private constant TOTAL_SUPPLY = 12;

address[TOTAL_SUPPLY] public bradyIndexToAddress;
Offer[TOTAL_SUPPLY] public bradyIsOfferedForSale;
Bid[TOTAL_SUPPLY] public bradyBids;
```

### 15. Use constant

Declare variables as constant if it will not be modified later in the contract.

### 16. Make functions simpler

It is suggested to avoid making a single function to do all the heavy lifting. Code should be modularized and reused. This helps in better readability and maintenance.

**Recommendation**

One such example is the token transfer between buyer and seller. This functionality is repeated in a few places. This can be written as a separate function.

### 17. Consider implementing state management

State management functionality can be incorporated to manage the initial auction. It will help track the current state and function level restrictions can be made possible for each state.
