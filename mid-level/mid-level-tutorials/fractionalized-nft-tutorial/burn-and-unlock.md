# Burn and Unlock

**Unlock NFT**

As the final step of this tutorial, we will unlock the token by burning the fractions. Basically, we designed the transfer function slightly different than regular cis-2 multi contract, it checks if a token is transferring to the same contract back, which indicates that the user wants to burn it.&#x20;

We have 1000 tokens in our state, all of them owned by us, in order to test it properly we will first burn 400 of them, transfer some of them to another account and then burn the rest from both accounts and finally, we will check the state.

In order to transfer it, first create a JSON file and fill it like below.

```
[
 {
  "token_id": "<FRACTION-ID>",
  "amount": "<AMOUNT-TO-BURN>",
  "from": {
   "Account": [
    "<YOUR-ADDRES>"
   ]
  },
  "to": {
   "Contract": [
    {
     "index": <FRACTIONALIZER-CONTRACT-INDEX>,
     "subindex": 0
    },
    ""
   ]
  },
  "data": ""
 }
]
```

Run the command below after the necessary changes in the parameter JSON.&#x20;

```
concordium-client contract update <YOUR-CONTRACT-INSTANCE> --entrypoint transfer --parameter-json cis2-fractionalizer/burn-20.json --schema dist/schema.bin --sender <YOUR-ADDRESS> --energy 6000 --grpc-port 10000 --grpc-ip node.testnet.concordium.com
```

As usual, after a change let’s check our state.&#x20;

<figure><img src="https://cdn-images-1.medium.com/max/1600/1*sBU_7vcuVsa54A4nzKvd6w.png" alt=""><figcaption></figcaption></figure>

Unsurprisingly, we have successfully burned 400 fractions and the account has 600 left. In the next section, we will mix some scenarios up.&#x20;

**Transfer another person**

Let’s test this a bit deeper by transferring some of it to an actual account. You could either modify the _cis2-multi-transfer.json_ file or create one like below. We will call it _transfer-account.json_ and transfer 100 fractions to the second account.

```
[
    {
        "token_id": "<FRACTION-ID>",
        "amount": "<AMOUNT-TO-BURN>",
        "from": {
            "Account": [
                "<YOUR-ADDRESS-FROM>"
            ]
        },
        "to": {
            "Account": [
                "<YOUR-ADDRESS-TO>"
            ]
        },
        "data": ""
    }
]
```

Run the transfer command below.

```
concordium-client contract update <YOUR-CONTRACT-INSTANCE> --entrypoint transfer --parameter-json cis2-fractionalizer/transfer-account.json --schema dist/schema.bin --sender <YOUR-ADDRESS>  --energy 6000 --grpc-port 10000 --grpc-ip node.testnet.concordium.com
```

<figure><img src="https://cdn-images-1.medium.com/max/1600/1*oA-D1ExGr4j8EoC_jZpTtQ.png" alt=""><figcaption></figcaption></figure>

Check the state again. We expect to see two owners with balances 500 and 100.

<figure><img src="https://cdn-images-1.medium.com/max/1600/1*0OsgFrfi3sgyisfLR7-YBA.png" alt=""><figcaption></figcaption></figure>

Nice. Both of the accounts have some assets now. As a final step, we will burn all these from them and check both fractionalizer’s state and the collateralized token’s state.

First, we are going to burn 500 from the first account.&#x20;

<figure><img src="https://cdn-images-1.medium.com/max/1600/1*ZhfWgpqBdekYG1-jPxSZwg.png" alt=""><figcaption><p>View the state if you want to double-check it.</p></figcaption></figure>

Let’s burn the remaining 100 from the second account. The expected behavior is to be able to unlock the first asset meaning in the token’s state we should see our account should have it.

Don’t forget to invoke the function from the second account as its the owner of the assets.&#x20;

<figure><img src="https://cdn-images-1.medium.com/max/1600/1*22yGyaKaDMx86OoiX8V-TQ.png" alt=""><figcaption></figcaption></figure>

As the final step, let’s check the token contract state. In the previous steps, we saw that the account has 999 and the fractionalizer contract has 1. But our smart contract is designed to transfer the token back when all of them burned. So there should be 1000 in the primary account intact.&#x20;

<figure><img src="https://cdn-images-1.medium.com/max/1600/1*Ab9VgBW6LPZej-XCMp2qag.png" alt=""><figcaption></figcaption></figure>

By looking at the state, we can confirm. The primary account received that token back and the fractionalizer contract has zero balance. Congratulations you have successfully completed this tutorial.
