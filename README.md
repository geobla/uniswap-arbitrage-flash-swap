# Uniswap Arbitrage Flash Swap Contract

You want to play with "millions" on the blockchain then try https://uniswap.org/docs/v2/smart-contract-integration/using-flash-swaps/. This contract is able to arbitrage between DEX on the blockchain without providing any own capital.
Basically is usable on the **Binance Smart Chain** and provides callbacks for common DEX, feel free to extend.

**This is not a full project, build your or own infrastructure around it!**

## Clone the repo and install Hardhat

To setup our smart contract development stack we need to install Hardhat. When you can deploy a code, it doesn’t matter if you use Truffle or Hardhat, even online SDE such as Remix for smart contract development.

Confirm your machine has git software already and clone the repo, running npm install bring all required package including Hardhat in your local repository. Hardhat is one of extensive solidity development tools and it is getting traction recently more than other tools.

Hardhat can be installed by NPM and in this repo Hardhat is used through a local installation in your project. This way your environment will be reproducible, and you will avoid future version conflicts. Now you can check what files are in the repository.
~~~
$ git clone https://github.com/geobla/uniswap-arbitrage-flash-swap.git 
$ cd uniswap-arbitrage-flash-swap
$ npm install
~~~

* `abi` directory provides flash loan contract abi. Application Binary Interface (ABI) is the standard way to interact with contracts in the Ethereum ecosystem, both from outside the blockchain and for contract-to-contract interaction.
* `contracts` directory contains a smart contract code `Flashswap.sol` and required interfaces for other protocol’s smart contract.
`deployments` directory has a deployment script to deploy a smart contract in target network.
* `src` directory contains all required helper Javascript codes that are called in `main.js` and `test.js` . `main.js` is Node.js script to monitor arbitrage opportunity and invoke a transaction for profits in BSC main network while `test.js` can run the same thing in BSC test network.
* `hardhat.config.js` is a configuration file for Hardhat and we will set network endpoint and your private key of wallet addresses. Also you can set BscScan’s API key to verify your smart contract on BscScan and make your contract code visible for everyone.

We are about to deploy a smart contract code but before doing this, we need to have BSC network endpoint we interact with and set up your wallet to extract a private key information. In the next section, I’ll cover how to create a project and endpoints in Chainstack service.

## Create a project and endpoints in Chainstack

Please note that Node.js script we develop requires WSS (WebSocket over SSL/TLS) therefore you need to pick up a service provider which supports WSS endpoint to integrate with. There seems like no workable WSS endpoints available from the official provided endpoints.

I was using Moralis speedy nodes but the policy changed recently and stopped serving speedy nodes. I switched to Chainstack instead of Moralis. It suppors both https and wss endpoints, which is fabulous. 

Copy both HTTPS endpoint and WSS endpoint information from the page and go to the repo. We will configure the parameters in the configuration file and deploy a smart contract by using Hardhat that you installed in the previous section.

## Deploy a smart contract

Rename `.env.template` file to `.env` and paste the copied endpoint information in the file. To deploy a smart contract in BSC mainnet, you have to fill out `MAINNET_ENDPOINT` and `MAINNET_KEY` at least. People can see only bytecodes of deployed smart contract on BscScan contract addres page.

If you’d like to verify the smart contract code and make it visible to everyone in BscScan page, get an `API key` from BscScan and fill out the value in `.env` file `BSCSCAN_KEY` as well. If you haven’t created archive node and elastic or archive node in Testnet project, then please delete unnecessary configuration from the file.

## Links

 * https://github.com/yuyasugano/pancake-bakery-arbitrage 
 * https://github.com/Uniswap/uniswap-v2-periphery/blob/master/contracts/examples/ExampleFlashSwap.sol

## Transactions

![transactions](examples-min.png "Transactions failed and success")

## Improvements

All examples are more or less overengineered for a quick start, so here we have a KISS solution.

 * Reduce overall contract size for low gas fee on deployment
 * Provide a on chain validation contract method to let nodes check arbitrage opportunities
 * Stop arbitrage opportunity execution as soon as possible if its already gone to reduce gas fee on failure
 * Interface callbacks for common DEX on "Binance Smart Chain" are included

## Infrastructure

Basically arbitrage opportunity dont last long, your transaction must make it into the next block. So you have <3 seconds watching for opportunities, decide and execute transaction. Sometimes there are also a chance to 2-3 have block, see example below.  

```
[7920960] [6/1/2021, 5:50:37 PM]: alive (bsc-ws-node.nariox.org) - took 308.42 ms
[7920991] [6/1/2021, 5:52:09 PM]: [bsc-ws-node.nariox.org] [BAKE/BNB ape>bakery] Arbitrage opportunity found! Expected profit: 0.007 $2.43 - 0.10%
[7920991] [6/1/2021, 5:52:09 PM] [bsc-ws-node.nariox.org]: [BAKE/BNB ape>bakery] and go:  {"profit":"$1.79","profitWithoutGasCost":"$2.43","gasCost":"$0.64","duration":"539.35 ms","provider":"bsc-ws-node.nariox.org"}
[7920992] [6/1/2021, 5:52:13 PM]: [bsc-ws-node.nariox.org] [BAKE/BNB ape>bakery] Arbitrage opportunity found! Expected profit: 0.007 $2.43 - 0.10%
[7920992] [6/1/2021, 5:52:13 PM] [bsc-ws-node.nariox.org]: [BAKE/BNB ape>bakery] and go:  {"profit":"$1.76","profitWithoutGasCost":"$2.43","gasCost":"$0.67","duration":"556.28 ms","provider":"bsc-ws-node.nariox.org"}
[7921000] [6/1/2021, 5:52:37 PM]: alive (bsc-ws-node.nariox.org) - took 280.54 ms
```

### Requirements / Hints

 * You have a time window of 1000ms every 3 seconds (blocktime on BSC) and you should make it into the next transaction
 * Websocket connection is needed to listen directly for new incoming blocks 
 * Public provided Websocket are useless `bsc-ws-node.nariox.org` simply they are way behind notify new blocks
 * Use a non public provider; or build your own node (light node helps) and better have multiple owns; let the fastest win
 * Spread your transaction execution around all possible providers, first one wins (in any case transactions are only execute once based on `nonce`)
 * Find suitable pairs with liquidity but not with much transaction
 * You can play with full pair liquidity, but dont be too greedy think of a price impact you would have
 * Common opportunities are just between 0,5 - 1%
 * Do not estimate transaction fee, just calculate it once and provide a static gas limit. Simply its takes to long
 * There is block parameter until the transaction is valid, so you can abort execution eg after +3 blocks
 * Payback is directly calculated by calling the foreign contracts so its project independent (no hardcoded fee calculation) 
 * The profit is transferred to the owner / creator of the contract :)

### Function

The contract is plain and simple [contracts/Flashswap.sol] some basic hints:

Check arbitrage opportunity between DEX. Read only method the one blockchain

```
    function check(
        address _tokenBorrow, // example: BUSD
        uint256 _amountTokenPay, // example: BNB => 10 * 1e18
        address _tokenPay, // example: BNB
        address _sourceRouter,
        address _targetRouter
    ) public view returns(int256, uint256) {
```

Starts the execution. You are able to estimate the gas usage of the function, its also directly validating the opportunity. Its slow depending on connected nodes.

```
    function start(
        uint _maxBlockNumber,
        address _tokenBorrow, // example BUSD
        uint256 _amountTokenPay,
        address _tokenPay, // our profit and what we will get; example BNB
        address _sourceRouter,
        address _targetRouter,
        address _sourceFactory
    ) external {
```

As all developers are lazy and just forking projects around without any rename a common implementation is possible. Basically the pair contract call the foreign method of the contract. You can find them the naming in any pair contract inside the `swap()` method.

Example: https://bscscan.com/address/0x0eD7e52944161450477ee417DE9Cd3a859b14fD0#code: `if (data.length > 0) IPancakeCallee(to).pancakeCall(msg.sender, amount0Out, amount1Out, data);`

Extend method if needed:

```
# internal callback 
function execute(address _sender, uint256 _amount0, uint256 _amount1, bytes calldata _data) internal

# foreign methods that get called
function pancakeCall(address _sender, uint256 _amount0, uint256 _amount1, bytes calldata _data) external
function uniswapV2Call(address _sender, uint256 _amount0, uint256 _amount1, bytes calldata _data) external
```

### Run it

Its not a full infrastructure, but a working workflow, if you deploy the contract.

``` bash
cp env.template .env # replace values inside ".env"
node watcher.js
```

```
started: wallet 0xXXXX - gasPrice 5000000000 - contract owner: 0xXXXX
[bsc-ws-node.nariox.org] You are connected on 0xXXXX
[8124531] [6/8/2021, 7:53:20 PM]: [bsc-ws-node.nariox.org] [BUSD/BNB pancake>panther] Arbitrage checked! Expected profit: -0.015 $-4.99 - -0.15%
[8124532] [6/8/2021, 7:53:21 PM]: [bsc-ws-node.nariox.org] [BUSD/BNB pancake>panther] Arbitrage checked! Expected profit: -0.015 $-4.99 - -0.15%
[8124533] [6/8/2021, 7:53:24 PM]: [bsc-ws-node.nariox.org] [BUSD/BNB pancake>panther] Arbitrage checked! Expected profit: -0.014 $-4.71 - -0.14%
[8124534] [6/8/2021, 7:53:27 PM]: [bsc-ws-node.nariox.org] [BUSD/BNB pancake>panther] Arbitrage checked! Expected profit: -0.014 $-4.61 - -0.14%
```

#### Hints

 * Designed to have multiple chain connectivities, play with some non public providers to be faster then the public once. Its all designed as "first win"
 
