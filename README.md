# Flash Loans

![](https://i.imgur.com/HZQWDCW.png)

Have you ever wanted to become a billionaire without having to collateralize anything? Well, that's flash loans. 🤑🤑

In this level we will learn how to take a Flash Loan from [Aave](https://aave.com) and utilize this new concept in DeFi which doesnt exist in the web2 world. There is no good analogy for this from the traditional finance world, since this is simply impossible outside blockchains.

Are you excited? Well, I surely am 🥳🥳

## Traditional Banking Systems?

How do traditional banking systems work? If you want a loan you have to put forward a collateral against which you can take the loan. This is typically how lending/borrowing in DeFi also works. 

However, you may need just a shit ton of money at times to execute some sort of attack that you cannot possibly provide collateral for, perhaps to execute a huge arbitrage trade or attack some contracts.

## What are Flash Loans?
As you might be thinking its some kind of loan? Well yes it is. Its a special type of a loan where a borrower can borrow an asset as long as they return the borrowed amount and some interest **before the end of the transaction**. Since the borrowed amount is returned back, with interest, in the same transaction, there is no possibility for anyone to run away with the borrowed money. If the loan is not repaid in the same transaction, the transaction fails overall and is reverted.

This simple, but fascinating, detail is what allows you to borrow billions with no upfront capital or collateral, because you *need* to pay it back in the same transaction itself. However, you can go wild with that money in between borrowing it and paying it back.

Remember that all of this happens in one transaction 👀

## Applications of a Flash Loan

They help in arbitrage between assets, often play part in DeFi hacks, and other use cases. You can essentially use your own creativity to create something new 😇

In this tutorial we will only focus on how `Simple Flash Loan` works which includes being able to borrow one asset. There are alternatives where you can borrow multiple assets as well. To read about other kinds of flash loans, read the documentation from [Aave](https://docs.aave.com/developers/guides/flash-loans)

Let us try to go a little deep on one use case which is of arbitrage. What is arbitrage? Imagine there are two crypto exchanges - A and B. Now A is selling a token `LW3` for less price than B. You can make profits if you buy `LW3` from A in exchange of DAI and then sell it on B gaining more DAI than the amount you initially started with. 

Trading off price differences across exchanges is called arbitrage. Arbitrageurs are a necessary evil that help keep prices consistent across exchanges.

## How do Flash Loans work?

There are 4 basic steps to any flash loan. To execute a flash loan, you first need to write a smart contract that has a transaction that uses a flash loan. Assume the function is called `createFlashLoan()`. The following 4 steps happen when you call that function, in order:
- Your contract calls a function on a Flash Loan provider, like Aave, indicating which asset you want and how much of it
- The Flash Loan provider sends the assets to your contract, and calls back into your contract for a different function, `executeOperation`
- `executeOperation` is all custom code you must have written - you go wild with the money here. At the end, you approve the Flash Loan provider to withdraw back the borrowed assets, along with a premium
- The Flash Loan provider takes back the assets it gave you, along with the premium.

![](https://i.imgur.com/wbg8rZ2.png)

If you look at this diagram, you can see how a flash loan helped the user make a profit in an arbitrage trade. Initially the user started a transaction by calling lets say a method `createFlashLoan` in your contract which is named as `FlashLoan Contract`. When the user calls this function, your contract calls the `Pool Contract` which exposes the liquidity management methods for a given pool of assets and has already been deployed by Aave. When the `Pool Contract` recieves a request to create a flash loan, it calls the `executeOperation` method on your contract with the DAI in the amount user has requested. Note that the user didnt have to provide any collateral to get the DAI, he just had to call the transaction and that `Pool Contract` requires you to have the `executeOperation` method for it to send you the DAI

Now in the `executeOperation` method after recieving the DAI, you can call the contract for `Exchange A` and buy some `LW3` tokens from all the DAI that the  `Pool Contract` sent you. After recieving the`LW3 Tokens` you can again swap them for DAI by calling the `Exchange B` contract. 

By this time now your contract has made a profit, so it can allow the `Pool Contract` to withdraw the amount which it sent our contract along with some interest and return from the `executeOperation` method.

Once our contract returns from the `executeOperation` method, the `Pool Contract` has allowance to withdraw the DAI it originally sent along with the interest from our `FlashLoan Contract`, so it withdraws it.

All this happens in one transaction, if anything is not satfified during the transaction like for example our contract fails in doing the arbitrage, rememeber everything will get reverted and it will be as if our contract never got the DAI in the first place. All you would have lost is the gas fees for executing all this.

User can now withdraw profits from the contract after the transaction is completed

It has been suggested by Aave to withdraw the funds after a successful arbitrage and not keep them long in your contract because it can cause a `griefing attack`. An example has been provided [here](https://ethereum.stackexchange.com/questions/92391/explain-griefing-attack-on-aave-flash-loan/92457#92457).

## Build

Lets build an example where you can experience how we can start a flash loan. Note we wont be actually doing an arbitrage here, because finding profitable arbitrage opportunities is the hardest part and not related to the code, but will esssentially just learn how to execute a flash loan.

Lets get started 🚀

- To setup a Hardhat project, Open up a terminal and execute these commands

  ```bash
  npm init --yes
  npm install --save-dev hardhat
  ```
  
- If you are not on mac, please do this extra step and install these libraries as well :)

  ```bash
  npm install --save-dev @nomiclabs/hardhat-waffle ethereum-waffle chai @nomiclabs/hardhat-ethers ethers
  ```

- In the same directory where you installed Hardhat run:

  ```bash
  npx hardhat
  ```

  - Select `Create a basic sample project`
  - Press enter for the already specified `Hardhat Project root`
  - Press enter for the question on if you want to add a `.gitignore`
  - Press enter for `Do you want to install this sample project's dependencies with npm (@nomiclabs/hardhat-waffle ethereum-waffle chai @nomiclabs/hardhat-ethers ethers)?`

Now you have a hardhat project ready to go!

Install OpenZeppelin contracts, Aave contracts and dotenv in the same terminal

```bash
  npm install @openzeppelin/contracts @aave/core-v3 dotenv
```

Now let's gets started with writing our smart contract, create your first smart contract inside the `contracts` directory and name it `FlashLoanExample.sol`

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.10;

import "@openzeppelin/contracts/utils/math/SafeMath.sol";
import "@aave/core-v3/contracts/flashloan/base/FlashLoanSimpleReceiverBase.sol";
import "@openzeppelin/contracts/token/ERC20/IERC20.sol";


contract FlashLoanExample is FlashLoanSimpleReceiverBase {
  using SafeMath for uint;
  event Log(address asset, uint val);

  constructor(IPoolAddressesProvider provider)
    public
    FlashLoanSimpleReceiverBase(provider)
  {}

  function createFlashLoan(address asset, uint amount) external {
      address receiver = address(this);
      bytes memory params = ""; // use this to pass arbitrary data to executeOperation
      uint16 referralCode = 0;

      POOL.flashLoanSimple(
       receiver,
       asset,
       amount,
       params,
       referralCode
      );
  }

   function executeOperation(
    address asset,
    uint256 amount,
    uint256 premium,
    address initiator,
    bytes calldata params
  ) external returns (bool){
    // do things like arbitrage here
    // abi.decode(params) to decode params
    
    uint amountOwing = amount.add(premium);
    IERC20(asset).approve(address(POOL), amountOwing);
    emit Log(asset, amountOwing);
    return true;
  }
}
```

Now lets try to decompose this contract and understand it a little better. When we declared the contract we did it like this `contract FlashLoanExample is FlashLoanSimpleReceiverBase {`, our contract is named as `FlashLoanExample` and it is inheriting a contract named as `FlashLoanSimpleRecieverBase` which is a contract from [Aave](https://github.com/aave/aave-v3-core/blob/master/contracts/flashloan/base/FlashLoanSimpleReceiverBase.sol) which you use to setup your contract as the reciever for the flash loan. 

Now after declaring the contract, if we look at the constructor, it takes in a provider of type `IPoolAddressesProvider` which is essentially the address of the `Pool Contract` we talked about in the example above wrapped around an interface of type `IPoolAddressesProvider`. This interface is also provided to us by Aave and can be found [here](https://github.com/aave/aave-v3-core/blob/master/contracts/interfaces/IPoolAddressesProvider.sol). `FlashLoanSimpleReceiverBase` requires this provider in its constructor.

```solidity
  constructor(IPoolAddressesProvider provider)
    public
    FlashLoanSimpleReceiverBase(provider)
  {}
```

The first function we implemented was `createFlashLoan` which takes in the asset and amount from the user for which he wants to start the flash loan. Now for the reciever address, you can specify the address of the `FlashLoanExample Contract` and we have no params so lets just keep it as empty. For `referralCode` we kept it as 0 because this transaction was executed by user directly without any middle man. To read more about these parameters you can go [here](https://docs.aave.com/developers/core-contracts/pool). After declaring these variables, you can call the `flashLoanSimple` method inside the instance of the `Pool Contract` which is initialized within the `FlashLoanSimpleReceiverBase` which our contract had inherited, you can look at the code [here](https://github.com/aave/aave-v3-core/blob/master/contracts/flashloan/base/FlashLoanSimpleReceiverBase.sol#L19).


```solidity
function createFlashLoan(address asset, uint amount) external {
      address reciever = address(this);
      bytes memory params = "";
      uint16 referralCode = 0;

      POOL.flashLoanSimple(
       reciever,
       asset,
       amount,
       params,
       referralCode
      );
  }

```

After making a `flashLoanSimple` call, `Pool Contract` will perform some checks and will send asset in the amount that was requested, to the asset to the `FlashLoanExample Contract` and will call the `executeOperation` method. Now inside this method you can do anything with this asset but in this contract we just give approval to the `Pool Contract` to withdraw the amount that we owe along with some premium. Then we emit a log and return from the function

```solidity
   function executeOperation(
    address asset,
    uint256 amount,
    uint256 premium,
    address initiator,
    bytes calldata params
  ) external returns (bool){
    // do things like arbitrage, liquidation, etc
    // abi.decode(params) to decode params
    uint amountOwing = amount.add(premium);
    IERC20(asset).approve(address(POOL), amountOwing);
    emit Log(asset, amountOwing);
    return true;
  }
}
```

Now we will try to create a test to actually see this flash loan in action.

Now because `Pool Contract` is deployed on Polygon Mainnet, we need some way to interact with it in our tests. 

We use a feature of Hardhat known as `Mainnet Forking` which can simulate having the same state as mainnet, but it will work as a local development network. That way you can interact with deployed protocols and test complex interactions locally.

Note this has been referenced from the official documentation of [Hardhat](https://hardhat.org/hardhat-network/guides/mainnet-forking.html)

To configure this, open up your `hardhat.config.js` and replace its already existing content with the following lines of code

```javascript
require("@nomiclabs/hardhat-waffle");
require("dotenv").config({ path: ".env" });

const ALCHEMY_API_KEY_URL = process.env.ALCHEMY_API_KEY_URL;

/**
 * @type import('hardhat/config').HardhatUserConfig
 */
module.exports = {
  solidity: "0.8.10",
  networks: {
    hardhat: {
      forking: {
        url: ALCHEMY_API_KEY_URL,
      },
    },
  },
};
```

You will see that we configured hardhat forking here

Now lets add the env variable for `ALCHEMY_API_KEY_URL`. 

Create a new file called `.env` and add the following lines of code to it

```env
ALCHEMY_API_KEY_URL="ALCHEMY-API-KEY-URL-FOR-POLYGON-MAINNET"
```
Replace `ALCHEMY-API-KEY-URL-FOR-POLYGON-MAINNET` with the url of the node for Polygon Mainnet. To get this url go to [alchemy](https://alchemy.com) and login. After that click on `Create App` and from the dropdown select chain as `Polygon` and network as `Mainnet`. The app should now be created, click on `View Key` and copy the `HTTP` value.


After creating the `.env` file, you will need one more file before we can actually write the test.

Create a new file named `config.js` and add the following lines of code to it

```javascript
// Mainnet DAI Address
const DAI = "0x8f3Cf7ad23Cd3CaDbD9735AFf958023239c6A063";
// Random user's address that happens to have a lot of DAI on Polygon Mainnet
const DAI_WHALE = "0xD92B63D0E9F2CE9F77c32BfeB2C6fACd20989eB3";

// Mainnet Pool contract address
const POOL_ADDRESS_PROVIDER = "0xa97684ead0e402dc232d5a977953df7ecbab3cdb";
module.exports = {
  DAI,
  DAI_WHALE,
  POOL_ADDRESS_PROVIDER,
};
```

If you look at this file, we have three variables - `DAI`, `DAI_WHALE` and `POOL_ADDRESS_PROVIDER`. `DAI` is the address of the `DAI` contract on polygon mainnet. `DAI_WHALE` is an address on polygon mainnet with lots of DAI and `POOL_ADDRESS_PROVIDER` is the address of the `PoolAddressesProvider` on polygon mainnet that our contract is expecting in the constructor. The address can be found [here](https://docs.aave.com/developers/deployed-contracts/v3-mainnet/polygon).

Since we are not actually executing any arbitrage, and therefore will not be able to pay the premium if we run the contract as-is, we use another Hardhat feature called **impersonation** that lets us send transactions on behalf of *any* address, even without their private key. However, of course, this only works on the local development network and not on real networks. Using impersonation, we will steal some DAI from the `DAI_WHALE` so we have enough DAI to pay back the loan with premium.

Awesome 🚀, we have everything setup now lets go ahead and write the test

Inside your `test` folder create a new file `deploy.js` and add the following lines of code to it

```javascript
const { expect, assert } = require("chai");
const { BigNumber } = require("ethers");
const { ethers, waffle, artifacts } = require("hardhat");
const hre = require("hardhat");

const { DAI, DAI_WHALE, POOL_ADDRESS_PROVIDER } = require("../config");

describe("Deploy a Flash Loan", function () {
  it("Should take a flash loan and be able to return it", async function () {
    const flashLoanExample = await ethers.getContractFactory(
      "FlashLoanExample"
    );

    const _flashLoanExample = await flashLoanExample.deploy(
      // Address of the PoolAddressProvider: you can find it here: https://docs.aave.com/developers/deployed-contracts/v3-mainnet/polygon
      POOL_ADDRESS_PROVIDER
    );
    await _flashLoanExample.deployed();

    const token = await ethers.getContractAt("IERC20", DAI);
    const BALANCE_AMOUNT_DAI = ethers.utils.parseEther("2000");
    
    // Impersonate the DAI_WHALE account to be able to send transactions from that account
    await hre.network.provider.request({
      method: "hardhat_impersonateAccount",
      params: [DAI_WHALE],
    });
    const signer = await ethers.getSigner(DAI_WHALE);
    await token
      .connect(signer)
      .transfer(_flashLoanExample.address, BALANCE_AMOUNT_DAI); // Sends our contract 2000 DAI from the DAI_WHALE

    const tx = await _flashLoanExample.createFlashLoan(DAI, 1000); // Borrow 1000 DAI in a Flash Loan with no upfront collateral
    await tx.wait();
    const remainingBalance = await token.balanceOf(_flashLoanExample.address); // Check the balance of DAI in the Flash Loan contract afterwards
    expect(remainingBalance.lt(BALANCE_AMOUNT_DAI)).to.be.true; // We must have less than 2000 DAI now, since the premium was paid from our contract's balance
  });
});

```

Now lets try to understand whats happening in these lines of code

First using Hardhat's extended `ethers` version, we call the function `getContractAt` to get the instance of DAI deployed on Polygon Mainnet. Remember Hardhat will simulate Polygon Mainnet, so when you get the contract at the address of `DAI` which you had specified in the `config.js`, Hardhat will actually create an instance of DAI contract which matches that of Polygon Mainnet.

After that the lines given below, will again try to impersonate/simulate the account on Polygon Mainnet with the address as `DAI_WHALE`. Now the fascinating point is that even though Hardhat doesnt have the private key of `DAI_WHALE` in the local testing env, it will act as if we already know its private key and can sign transactions on the behalf of `DAI_WHALE`. It will also have the amount of DAI as it has on the polygon mainnet

```javascript
 await hre.network.provider.request({
      method: "hardhat_impersonateAccount",
      params: [DAI_WHALE],
    });
```

Now after that we create a signer for `DAI_WHALE` so that we can call the simlulated DAI contract with the address of `DAI_WHALE` and transfer some `DAI` to `FlashLoanExample Contract`. We need to do this so we can pay off the loan with premium, as we will otherwise not be able to pay the premium. In real world applications, the premium would be paid off the profits made from arbitrage or attacking a smart contract.

```javascript
const signer = await ethers.getSigner(DAI_WHALE);
    await token
      .connect(signer)
      .transfer(_flashLoanExample.address, BALANCE_AMOUNT_DAI);
```

After this we start a flash loan and checking that the remaining balance of `FlashLoanExampleContract` is less than the amount it initially started with, the amount will be less because the contract had to pay a premium on the loaned amount.

```javascript
    const tx = await _flashLoanExample.createFlashLoan(DAI, 1000);
    await tx.wait();
    const remainingBalance = await token.balanceOf(_flashLoanExample.address);
    expect(remainingBalance.lt(BALANCE_AMOUNT_DAI)).to.be.true;
```

To run the test you can on your terminal simply execute

```bash
npx hardhat test
```

If the tests pass, you have successfully executed a flash loan.

Hurray 🥂🥂🥂🥂🥂

This is huge 🚀🚀🚀
