# Coding your own MultiSig wallet on Polygon
This quest will not involve any Hradhat setup. You can just go to Remix and ensure you are on Polygon. The contract to be shown in this quest was deployed to Mumbai testnet, so you can use Mumbai to follow along.
So, In this quest, we will build a 2-of-2 multi-signature wallet using Solidity. But first, what is a multi-signature wallet? A multi-signature wallet, AKA MultiSig wallet, is a wallet that is owned by more than one owner and demands more than one signature to send a transaction. An n-of-m MultiSig wallet requires that n out of m owners confirm (sign) the transaction for it to get sent. In our case, our wallet has two owners and both of them have to sign a transaction before funds get released. You can use such a wallet if you trust someone and you would like to create a shared address with them. Or maybe, you can use it to secure your money by creating two different private keys and storing them in different places. This way, if an attacker steals on key, they will not be able to use your money. Without further ado, let’s get our hands on the keyboard!

## Starting with the smart contract:
Ok, now open your IDE and write these on top of your code:

```js
//SPDX-License-Identifier: MIT
pragma solidity 0.8.0;
pragma experimental ABIEncoderV2;
```

This is just regular practice, adding a license identifier and specifying solidity version.  And you can see ABIEncoderV2, which you may need when writing a contract. It makes it possible to return arrays from functions and pass structs to functions without workarounds.
Now, let’s cut to the chase and create state variables and a constructor.

```js
contract MultiSig {
    address public owner1;
    address public owner2;
    uint256 public id;
    struct Transaction {
        address payable to;
        uint256 amount;
        bool signedByOwnerOne;
        bool signedByOwnerTwo;
    }
    Transaction[] public transactions;

    constructor(address _owner1, address _owner2) {
        owner1 = _owner1;
        owner2 = _owner2;
    }

    modifier onlyOwner() {
        require(msg.sender == owner1 || msg.sender == owner2);
        _;
    }
```

In the code snippet above, you can see two addresses (owners). And a Transaction, which is a struct that contains the necessary details to confirm a 2-of-2 wallet’s transaction. There is a field of type address payable to allow the receiver to receive funds, a uint to store the value to Ethers to transfer, and two bool variables to keep track of who signed what. We store Transactions in a dynamic array called transactions. Then there is a constructor that initializes the owners. So if you want to share a 2-of-2 wallet with your best friend, you have to provide both of your addresses. Finally, you can see the famous onlyOwner modifier to help restrict function calls to owners.

## Initiating a transaction:
We want our wallet to let us send transactions. How can we do that programmatically? Well, first we should create a function that allows an owner to request a transaction. Then we need a function that allows the other owner to approve. Take a look at this function:

```js
function initiateTransaction(address payable _to, uint256 _amount)
        public
        onlyOwner
        returns (uint256)
    {
        Transaction memory transaction;
        transaction.to = _to;
        transaction.amount = _amount;
        if (msg.sender == owner1) {
            transaction.signedByOwnerOne = true;
        } else {
            transaction.signedByOwnerTwo = true;
        }
        transactions.push(transaction);
        return id++;
    }
```

This is a function restricted to owners, no one other than them can call it. It takes two parameters, the intended destination of the transactions and the amount of MATICs to be sent. A new object of type transaction is created every time this function is called. Its fields get populated respectively: the _to_ and _amount_ fields are received directly from the function’s parameters. Then the transaction gets signed by whoever called the function. And then the transaction gets added to the array of transactions, waiting to get approved. That is why we used the keyword memory and not storage for the transaction, we just need it to store information in an intermediary step, then it gets stored in the array so no need to keep it in storage.

## Approving a transaction:
Now you initiated a transaction, but it is still just a request for now. It needs to get approved by the second owner. So we need a function that allows an owner to approve transactions. This can be easily done by signing the transaction using one of the boolean fields in each transaction:

```js
 function approveTransaction(uint256 _id) public onlyOwner {
        require(_id < transactions.length);
        if (msg.sender == owner1) {
            transactions[_id].signedByOwnerOne = true;
        } else {
            transactions[_id].signedByOwnerTwo = true;
        }
        withdraw(_id);
    }
```

So if you are an owner and would like to approve a transaction, you provide its id as a parameter to this function. The function then checks what bool field to flip to true indicating that you have signed. Now that both owners are on board, we can safely transfer the funds. The latter step is achieved using a function called withdraw. Now take a breath and get ready, we are moving to the most important function in this contract.

## Executing a transaction:
What now? We have the transaction initiated and signed. Now, we just need to run a couple of checks for security and release the funds.

```js
  function withdraw(uint256 _id) private {
        require(address(this).balance >= transactions[_id].amount);
        require(
            transactions[_id].signedByOwnerOne &&
                transactions[_id].signedByOwnerTwo
        );
        require(transactions[_id].amount != 0);
        transactions[_id].to.transfer(transactions[_id].amount);
        transactions[_id].amount = 0;
    }
```

You check if there are enough MATICs in the wallet’s balance. Then a require() to ensure that the transaction is signed by the owners. Finally, if the amount of MATICs is not zero, release the funds to the designated address. Then set the amount to zero to mark that the money is spent (and it is useful in your frontend in case you want to show pending transactions - you can search for transactions with amount > 0).

## Last touches & conclusion:
It is always beneficial to add utility functions that you think can be useful. 
```js
receive() external payable {}

    function getBalance() public view returns (uint256) {
        return address(this).balance;
    }

    function getTransactions() public view returns (Transaction[] memory) {
        return transactions;
    }
```

Note that solidity gives you a getter function for all public state fields for free. So we have a getter function for transactions. This implies that the getTransactions function is useless, right? Well not exactly, getters for arrays expect a specific index and return only the element in that index. Meanwhile, getTransactions returns the whole array at once thanks to ABIEncoderV2.
So, now you have written a MultiSig wallet on Polygon! Feel free to deploy it, send it some MATICs, try it, and build on it. Happy coding!  



