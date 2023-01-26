# Pi Network SDK Integration Guide

The following guide explains how to build a Pi SDK for any programming language, and how to integrate app-to-user (A2U)
payments into it. It walks you through how to use the various building blocks to achieve the A2U payment process.

Code snippets in this documentation are meant as high-level examples of the required steps, and will NOT run
if they are copy-pasted into an existing project. They are written in Javascript, but the whole point of
this document is to explain how to build a Pi SDK in _ANY language other_ than JS.

If you need to integrate Pi into your Node.JS project, you're looking for the
[Pi SDK for Node.JS](https://github.com/pi-apps/pi-node-js-sdk#coming-soon).


## Building blocks

A2U payments involve both interacting with the Pi Blockchain and the Pi backend. The Pi blockchain is obviously the source of truth for exchanging Pi. The Pi backend is used to improve the end-user experience (e.g. provide users understandable memos on their wallet, and link backs to your app, while preserving user and developer privacy by not making this information public on the chain) and to assist developers in avoiding to make payment mistakes (e.g. double payments due to server faults)

Your Pi SDK implementation needs to both be able to sign and submit blockchain transactions using the Stellar SDK and send additional API calls to the Pi backend using standard https libraries. As a reference, here is the list of the [official Stellar SDK](https://developers.stellar.org/docs/tools-and-sdks#sdk-library)'s in various languages. You will need to pick the one that is applicable to the language you are working on, include it as dependancy into your SDK, and use it as needed.


## Integration Guide

### Overall flow

There are certain steps you need to follow to make A2U payment. Before we take a look at the overall flow, there's one thing to keep in mind. 

> **important** At the moment, you can process only one A2U payment at a time, due to the nature of sequence number requirement on the Pi Blockchain. This is a safety feature of the blockchain. However, we will support batch payments, *i.e. creating an A2U payment with multiple user uids,* to increase the throughput. This is possible through simply adding more _operations_ into each _transaction_.

You essentiall need to encapsulate the following API calls into convenient SDK functions:

1. Send API request to the Pi server to create a payment
> Similarly to the U2A payments where app developers create a payment using the Pi SDK's `createPayment` method, you need to implement a method that creates a payment through an API endpoint `api.minepi.com/v2/payments`.
- Notice that unlike U2A payment, which requires the Server-Side Approval, your SDK can automatically approve A2U payments, as the flow starts from the app side, not from the user side. 

2. Load the account
> You need to look up the account every time before submitting a transaction, even if the Pi server gave you its address in the past. That's because the account could have been destroyed on the blockchain or the user could have changed their actuve wallet. By following this recommendation, developers can save the transaction fee that might have been wasted and avoid sending Pi to the wrong address. Remember that the transaction fee still gets spent even if a transaction fails.

3. Build the transaction
> You need to build a transaction with the relevant data such as the recipient's wallet address.

4. Sign the transaction
> After you build the transaction, you need to sign with the developer's keypair. Retrieving the keypair requires the secret seed.

5. Submit the transaction to the Pi blockchain
> This is when the transaction is sent to the Pi blockchain. Depending on the status, you will get a corresponding response.

6. Complete the payment by sending an API request to the `/complete` endpoint
> The transaction will be verified by the Pi server after you submit it in the previous step. Send an API request to the `/complete` endpoint to check the status and complete the payment.


### Example integration:

We are going to follow each step shown in the "Overall flow" section above to make an A2U payment.

It is worth noting that although `js-stellar-sdk` is available both on the frontend and backend,
since the A2U is a server-side operation, we are providing Node.js examples.

1. Send API request to the Pi server to create a payment

```javascript
  const axios = require('axios');

  // API Key of your app, available in the Pi Developer Portal
  // DO NOT hardcode this, read it from an environment variable and treat it like a production secret.
  const PI_API_KEY = "YOUR_PI_API_KEY"

  const axiosClient = axios.create({baseURL: 'https://api.minepi.com', timeout: 20000});
  const config = {headers: {'Authorization': `Key ${PI_API_KEY}`, 'Content-Type': 'application/json'}};
  
  // This is the user UID of this payment's recipient
  const userUid = "a1111111-aaaa-bbbb-2222-ccccccc3333d" // this is just an example uid!
  const body =  {amount: 1, memo: "Memo for user", metadata: {test: "your metadata"}, uid: userUid}; // your payment data and uid
  
  let paymentIdentifier;
  let recipientAddress;

  axiosClient.post(`/v2/payments`, body, config).then(response => {
    paymentIdentifier = response.data.identifier;
    recipientAddress = response.data.recipient;
  });
  
```

2. Load the account

```javascript
const StellarSdk = require('stellar-sdk');

const myPublicKey = "G_YOUR_PUBLIC_KEY" // your public key, starts with G

// an object that let you communicate with the Pi Testnet
// if you want to connect to Pi Mainnet, use 'https://api.mainnet.minepi.com' instead
const piTestnet = new StellarSdk.Server('https://api.testnet.minepi.com');

let myAccount;
piTestnet.loadAccount(myPublicKey).then(response => myAccount = response);

let baseFee;
piTestnet.fetchBaseFee().then(response => baseFee = response);
```

3. Build the transaction

```javascript
// create a payment operation which will be wrapped in a transaction
let payment = StellarSdk.Operation.payment({
  destination: recipientAddress,
  asset: StellarSdk.Asset.native(),
  amount: body.amount.toString()
});

// 180 seconds timeout
let timebounds;
piTestnet.fetchTimebounds(180).then(response => timebounds = response);

let transaction = new StellarSdk.TransactionBuilder(myAccount, {
  fee: baseFee,
  networkPassphrase: "Pi Testnet", // use "Pi Network" for mainnet transaction
  timebounds: timebounds
})
.addOperation(payment)
// IMPORTANT! DO NOT forget to include the payment id as memo
.addMemo(StellarSdk.Memo.text(paymentIdentifier));
transaction = transaction.build();
```

4. Sign the transaction

```javascript
// See the "Obtain your wallet's private key" section above to get this.
// And DON'T HARDCODE IT, treat it like a production secret.
const mySecretSeed = "S_YOUR_SECRET_SEED"; // NEVER expose your secret seed to public, starts with S
const myKeypair = StellarSdk.Keypair.fromSecret(mySecretSeed);
transaction.sign(myKeypair);
```

5. Submit the transaction to the Pi blockchain

```javascript
let txid;
piTestnet.submitTransaction(transaction).then(response => txid = response.id);
```

6. Complete the payment by sending API request to `/complete` endpoint

```javascript
// check if the response status is 200 
let completeResponse;
axiosClient.post(`/v2/payments/${paymentIdentifier}/complete`, {txid}, config).then(response => completeResponse = response);
```
