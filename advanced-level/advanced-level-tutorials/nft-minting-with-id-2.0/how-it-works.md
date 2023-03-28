# How it works?

#### Concordium NFT Minting with ID 2.0 Part 3

In this last section, we will show you how to run dApp in order to mint an NFT.&#x20;

**Verifier Backend**

First you need to run the verifier backend, our dApp will be communicating with it to get the statement, challenge and post the proof and get the signature back. It expects parameters from the terminal but you are free to use all of them from a JSON file. We will use a mixture by giving the keys (verify and sign) as a parameter from the terminal and the statement from a JSON. In order to create a custom statement, you can check this [link](https://developer.concordium.software/en/mainnet/net/guides/create-proofs.html). For this tutorial scenario, we will use age proofs to be able to verify if a person is older than 18 or not but you can also check if he is from a certain country or not.

```
[
    {
        "type": "AttributeInRange",
        "attributeTag": "dob",
        "lower": "19000327",
        "upper": "20050327"
    }
]
```

When you have the statement JSON file, run the application inside of your executable path. If you are using your own node change the node IP to localhost but since we are going to use the testnet node keep it as below.&#x20;

```
./<Executable-Name> --node http://node.testnet.concordium.com --port 8100 --log-level info --verify-key <YOUR-VERIFY-KEY> --sign-key <YOUR-SIGN-KEY> --statement "$(<PATH-TO-YOUR-STATEMENT/statement.json)"
```

<figure><img src="https://cdn-images-1.medium.com/max/1600/1*z2MnnbqdyQTg8LulxWGr_g.png" alt=""><figcaption></figcaption></figure>

**Running the dApp & Asking a proof**

In the _mint-ui_ directory start the dApp by:

```
yarn start
```

We will create a new instance of the cis2-multi contract and try to mint an NFT with another account. Click “DEPLOY NEW” to create a new instance as you notice, it sends the _verify\_key_ as an initiation parameter.

<figure><img src="https://cdn-images-1.medium.com/max/1600/1*M6oqv0fIMcyNgDbt8Izl4w.png" alt=""><figcaption><p>Our contract index is &#x3C;4168,0></p></figcaption></figure>

Click “GET SIGNATURE” and accept the request. Wait for the proof verification, if it’s verified you have a signature signed by the private key (_signKey_) given when running the application.

<figure><img src="https://cdn-images-1.medium.com/max/1600/1*2CYOwO2rO1-_q5iNitQYJA.png" alt=""><figcaption></figcaption></figure>

If everything goes well, we should have a signature like the one below.

<figure><img src="https://cdn-images-1.medium.com/max/1600/1*zhaWYdcaC3uf1irbomdGXg.png" alt=""><figcaption></figcaption></figure>

We are almost there, since we have now the signature we will be able to mint a token because we have proven that we are older than 18! Let’s provide the metadata and the contract index to mint our token!

<figure><img src="https://cdn-images-1.medium.com/max/1600/1*i61tKIhwlWLniUQ6cdV8Nw.png" alt=""><figcaption></figcaption></figure>

When the transaction is finalized, we will have an alert notifying our minting is successful!

<figure><img src="https://cdn-images-1.medium.com/max/1600/1*27tmGgUAS8XCjphaYbIbRA.png" alt=""><figcaption></figcaption></figure>

\
