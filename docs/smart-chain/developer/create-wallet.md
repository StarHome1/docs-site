# Key Management

This article is a guide about key management strategy on client side of your Decentralised Application on Binance Smart Chain

## Setup Web3

`web3.js` is a javascript library that allows our client-side application to talk to the blockchain. We configure web3 to communicate via Metamask.

`web3.js` doc is [here](https://web3js.readthedocs.io/en/v1.2.2/getting-started.html#adding-web3-js)

## Connect to BSC Testnet

```javascript
	const web3 = new Web3('https://data-seed-prebsc-1-s1.binance.org:8545');
```

## Set up account
If the installation and instantiation of web3 was successful, the following should successfully return a random account:
```javascript
    const account = web3.eth.accounts.create();
```
```
const Web3 = require('web3');
async function main() {

	const web3 = new Web3('https://data-seed-prebsc-1-s1.binance.org:8545');
    const loader = setupLoader({ provider: web3 }).web3;

    const account = web3.eth.accounts.create();
    console.log(account);
}
```