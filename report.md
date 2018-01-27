# Audit report for Dadi Public Sale
##### January 28 2018


## Summary

Audit Report prepared by Solidified for **Dadi** covering the public sale contracts.


## Scope of the audit

The audit was based on the solidity compiler *0.4.19+commit.c4cbbb05*. The scope of the audit is limited to the following source code file

1. DadiPublicSale.sol

Which contains the following libraries and contracts

1. SafeMath
2. Ownable
3. ERC20Basic
4. BasicToken
5. ERC20
6. StandardToken
7. DadiPublicSale


## Issues with TokenDistribution
### DadiPublicSale.sol

Balances mapping in the StandardToken is empty during the token distribution. It can cause the distribution to fail during token transfer. This can be resolved by adding the total token supply to the mapping.

```
token.transfer(_address, tokens);
```

## 1. Sales Wallet is not implemented properly
### DadiPublicSale.sol / Line 232

There is no option to remove the added address from salesWallet array. If any address is added accidently it cannot be removed and can potentially lead to the loss of funds.

```
 address[] private saleWallets;
```

It has a small chance of occurrence, but is a potential threat to the funds.

This threat can be avoided by adding an option to remove a particular address from the salesWallet array. Removing an element from the array can be expensive, but it is the trade-off that we can make for potential loss of funds. Use of mapping can also be considered.

Since it is a private array and no getters associated with it, there is no way to check for the list of addresses that are present. Also consider adding a getter method private to owner to verify the list.

```
function getWalletAddress() external onlyOwner view returns (address[]) {
    return saleWallets;
}
```

Also, there is a chance of duplicate wallet entry. This has to be taken care.

## 2. Chance of funds getting locked up after the sale is closed

If there is no wallet address present in the `saleWallets` array during the crowdsale, the funds collected will still be locked in the contract. Owner will not know if the saleWallet array is empty during the token buying process. There is no option to retrieve the funds once the sale is closed.

This can be solved with any one of the following methods.

1. Modify the existing `forwardFunds` to allow access for the owner so that it can be called by the owner manually.

    ```
    function forwardFunds (uint256 _value) public onlyOwner {
        uint accountNumber;
        address account;

        // move funds to a random saleWallet
        if (saleWallets.length > 0) {
            accountNumber = getRandom(saleWallets.length) - 1;
            account = saleWallets[accountNumber];
            account.transfer(_value);
            LogFundTransfer(account, _value);
        }
    }
    ```
2. Add an address to the `saleWallets` array during the contract initialization.
    ```
    function DadiPublicSale (StandardToken _token, uint256 _tokenSupply) public {
        require(_token != address(0));
        require(_tokenSupply != 0);
        token = StandardToken(_token);
        tokenSupply = _tokenSupply * (uint256(10) ** 18);
        maxGasPrice = 60000000000;       // 60 Gwei
        saleWallets.push(msg.sender); // Adding owner as one of the wallet address
    }
    ```


## 3. Sale status is not fully checked
### DadiPublicSale.sol

Several functions which modifies the contract state can be called before or after the sale has been started or completed. For example restarting the public sale with `startPublicSale` even after the sale is closed.

Consider adding state checking for functions through modifiers.

```
modifer saleShouldbe(SaleState _state) {
    require(state == _state);
    _;
}

function startPublicSale (uint256 rate) public onlyOwner saleShouldbe(SaleState.Preparing)

function distributeTokens (address _address) public onlyOwner saleShouldbe(SaleState.TokenDistribution) returns (bool)
```

## 4. Older compiler version can lead to some bugs
### DadiPublicSale.sol / Line 1

```
pragma solidity ^0.4.11;
```

Older compilers might be susceptible to some bugs. It is always suggested to use the latest stable version. As of this writing the recommended version is `0.4.19`.

List of known compiler bugs and their severity can be found [here](https://etherscan.io/solcbuginfo)


## 5. Use of older OpenZeppelin libraries
### DadiPublicSale.sol / Line 7 - 31

`SafeMath`, `ERC20Basic`, `BasicToken`, `ERC20`, `StandardToken` are outdated versions of OpenZepplin library. Update the libraries to the latest version which includes changes that can affect gas costs, some assertions and even reduce potential attack surface.

## 6. Install OpenZeppelin via NPM
### DadiPublicSale.sol

`SafeMath`, `Ownable`, `ERC20Basic`, `ERC20`, `BasicToken`, and `StandardToken` were copied from the OpenZeppelin repository. OpenZeppelin’s MIT license requires the license and copyright notice to be included if its code is used, and makes it difficult and error-prone to update to a more recent version.

Consider following the recommended way to use OpenZeppelin contracts, which is via the zeppelin-solidity NPM package. This allows for any bug fixes to be easily integrated into the codebase.


## 7. Redundant owner declaration and assignment
### DadiPublicSale.sol / Line 231 & 278

```
address public owner;

owner = msg.sender;
```

owner address declaration and assignment is already taken care by the `Ownable` contract. This redundant declaration and assignment can be removed from `DadiPublicSale`.


## 8. Redundant function declaration to get tokensPurchased
### DadiPublicSale.sol / Line 492 - 494

Public variables generate a getter by default and the function to retrieve the same can be removed.

```
function getTokensPurchased () public constant returns (uint256) {
    return tokensPurchased;
}
```

## 9. Reduntant msg.value check
### DadiPublicSale.sol / Line 574

`nonZero` modifier performs the msg.value check in the fallback function. The same check can be avoided in the `buyTokens` function.

```
require(msg.value > 0);
```


## 10. Use `view`/`pure` instead of `constant`
### DadiPublicSale.sol

Make use of view/pure. Use view if your function does not modify storage and pure if it does not even read any state information.

```
// Line 484
function getTokensAvailable () public constant returns (uint256)

// Line 501
function ethToUsd (uint256 _amount) public constant returns (uint256)

// Line 509
function getInvestorCount () public constant returns (uint count)

// Line 520
function getInvestor (address _address) public constant returns (uint256 contribution, uint256 tokens, uint index) 

// Line 530
function isInvested (address _address) internal constant returns (bool isIndeed)

// Line 605
function isValidContribution (address _address, uint256 _amount) internal constant returns (bool valid)

// Line 614
function isBelowCap (uint256 _amount) internal constant returns (bool)

// Line 623
function getRandom(uint max) internal constant returns (uint randomNumber)
```

## 11. Use modifiers
### DadiPublicSale.sol

Make use of modifiers for checking the sale status, msg.value and gas cost.

Consider following the below structure for sale status

```
modifer saleShouldbe(SaleState _state) {
    require(state == _state);
    _;
}

function distributeTokens (address _address) public onlyOwner saleShouldbe(SaleState.TokenDistribution) returns (bool)
```

## 12. Make use of the SafeMath library
### DadiPublicSale.sol

In places of arithmatic operations we can make use of the SafeMath library. It is not needed everywhere, but it is a good practive to follow the same pattern throughout the contract.

```
tokens = _amount * ethRate / tokenPrice;

tokens = _amount.mul(ethRate).div(tokenPrice);
```

## 13. Remove duplicate typecasting
### DadiPublicSale.sol

Reduntant typecasting is performed in multiple places. This can be removed.

```
function setState (uint256 _state) public onlyOwner {
    state = SaleState(uint(_state));
    LogStateChange(state);
}
```
