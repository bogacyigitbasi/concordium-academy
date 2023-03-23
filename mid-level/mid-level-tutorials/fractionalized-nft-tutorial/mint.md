# Mint

#### Mint Fractionalize NFTs

We have deployed fractionalizer and are ready to mint some NFTs as a fraction of another one. We will follow these steps:

* Mint a CIS-2 NFT
* Transfer NFT to the fractionalizer contract and lock it.
* Mint fractions
* Check state
* Transfer some
* Burn all of them
* Check the state

**Mint a Regular NFT (To be collateralized)**

Remember our fractionalization logic starts with collateralizing the CIS-2 NFT. In order to mint a CIS-2 NFT you can follow [this](https://developer.concordium.software/en/mainnet/smart-contracts/tutorials/sft-minting/index.html) tutorial. As previously mentioned in this tutorial, we will use cis2-multi smart contract for minting our token and you can find the full code here. The screenshot below shows only the _mint()_ function. You can find build/deploy processes in the tutorial.

<figure><img src="https://cdn-images-1.medium.com/max/1600/1*FLDwyQnUe3PXB1rYj8QIxw.png" alt=""><figcaption></figcaption></figure>

Let’s check the state by calling the _view()_ function below.

```
concordium-client contract invoke <YOUR-INDEX>--entrypoint view --schema ../cis2-multi/dist/schema.bin --grpc-port 10000 --grpc-ip node.testnet.concordium.com
```

<figure><img src="https://cdn-images-1.medium.com/max/1600/1*DbGusfXPYt5bbiDvz4Uiiw.png" alt=""><figcaption></figcaption></figure>

**Transfer CIS-2 NFT**

Nice. We have successfully minted a CIS-2 NFT and now we can transfer it to the fractionalizer contract. In order to do that, you need to call the _transfer()_ function from the token’s contract using its index and schema file. Create a JSON file to give the input parameters like the below and specify the fractionalizer contract index value.

```
[
 {
  "token_id": "<YOUR-TOKEN-ID>",
  "amount": "<TOKEN-AMOUNT-TO-LOCK>",
  "from": {
   "Account": [
    "<YOUR-ACCOUNT>"
   ]
  },
  "to": {
   "Contract": [
    {
     "index": <YOUR-FRACTIONALIZER-CONTRACT-INDEX>,
     "subindex": 0
    },
    "onReceivingCIS2"
   ]
  },
  "data": ""
 }
]
```

Please don't confuse in this part, you need to use the **token’s** _schema_ to change **its** state. We want to keep both schemas in the same project folder for the sake of the order. That's why we created a section for the _cis2-multi_ contracts schema file, so you can either copy the _schema_ file from _cis2-multi_ to your fractionalizer directory or you can call it from that file using the JSON above. We will do it the first way, create a file and copy & paste the schema from _cis2-multi_.&#x20;

Run the command below to transfer the token to the fractionalizer.

```
concordium-client contract update <YOUR-TOKEN-CONTRACT-INSTANCE> --entrypoint transfer --parameter-json cis2-fractionalizer/cis2-multi-transfer.json --schema multi/dist/schema.bin --sender <YOUR-ADDRESS> --energy 6000 --grpc-port 10000 --grpc-ip node.testnet.concordium.com
```

<figure><img src="https://cdn-images-1.medium.com/max/1600/1*NLRt7kFi-AOqQODYRR3LEg.png" alt=""><figcaption></figcaption></figure>

Nice. Let’s check the token contract’s state.

<figure><img src="https://cdn-images-1.medium.com/max/1600/1*64EQib5cuKNYyW8AqDviaQ.png" alt=""><figcaption></figcaption></figure>

Super cool! As you can see the fractionalizer contract has one token and our account has the rest.

Now, let’s check the fractionalizer’s state.&#x20;

<figure><img src="https://cdn-images-1.medium.com/max/1600/1*i6hF4WMIGBgIU7xqcZXA9g.png" alt=""><figcaption></figcaption></figure>

Even nicer! As you can see, we have locked the _“NFT 01”_ token as _received\_token\_amount_ with _token\_id:01_ from the _cis2-multi_ contract.

**Mint Fractions**

We are ready to mint fractions of it now. Create a JSON file like below.

```
{
 "owner": {
  "Account": ["<YOUR-ACCOUNT>"]
 },
 "tokens": [
  [
   "<YOUR-TOKEN-ID>",
   {
    "metadata": {
     "url": "<METADATA-URL>",
     "hash": "<HASH>"
    },
    "amount": "<FRACTION-AMOUNT>",
    "contract": { "index": <YOUR-TOKEN-CONTRACT-INDEX>, "subindex": 0 },
    "token_id": "<YOUR-TOKEN-ID-COLLATERAL>"
   }
  ]
 ]
}
```

We need to mint new tokens based on the collateralized one, so specify the exact _token\_id_ and contract _index_. We will specify the _amount_ that sets how many fractions are going to be minted.

```
concordium-client contract update <YOUR-CONTRACT-INSTANCE> --entrypoint mint --parameter-json ../sample-artifacts/cis2-fractionalizer/mint.json --schema ../cis2-fractionalizer/schema.bin --sender $ACCOUNT --energy 6000 --grpc-port 10000 --grpc-ip node.testnet.concordium.com
```

<figure><img src="https://cdn-images-1.medium.com/max/1600/1*3PdyHKaMEp03gw5P-zy3Tg.png" alt=""><figcaption></figcaption></figure>

Now, let’s check the fractionalizer’s state.&#x20;

```
concordium-client contract invoke <YOUR-INDEX> --entrypoint view --schema dist/schema.bin  --grpc-port 10000 --grpc-ip node.testnet.concordium.com
```

<figure><img src="https://cdn-images-1.medium.com/max/1600/1*GwVvUSGn6tIC8fVO3cZsug.png" alt=""><figcaption></figcaption></figure>

Congrats! You have now locked an NFT and created 1000 fractions that represent the token.

New fraction’s are CIS-2 tokens, you can transfer them or sell them on a marketplace. Anything that can apply to a CIS-2 token is available for these fractions too.&#x20;
