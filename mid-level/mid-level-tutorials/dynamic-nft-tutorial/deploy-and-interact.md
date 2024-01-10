---
description: >-
  In this section, we will deploy the contract, initialize it and invoke all of
  the entrypoints created on testnet.
---

# 🔃 Deploy and Interact

**Build**

We will build the smart contract with the following command, in your production releases we suggest using this flag and publishing the code. This way people can later check that the contract source matches the deployed module. More information about verifiable builds can be found [here](https://docs.rs/crate/cargo-concordium/latest).&#x20;

```
cargo concordium build --verifiable docker.io/concordium/verifiable-sc:1.70.0 -o module.wasm.v1 -e

```

{% hint style="info" %}
Please note that this build embeds the schema so you don't have to keep the schema as a separate compilation output anymore. The use of embedded schema is strongly suggested by Concordium.
{% endhint %}

<figure><img src="../../../.gitbook/assets/Screenshot 2023-12-28 at 15.00.10.png" alt=""><figcaption></figcaption></figure>

{% hint style="info" %}
Please note that the classical build command also works however, you will get the following warning:&#x20;

`This is not a verifiable build. Consider using the --verifiable option before deploying the module. A verifiable build packages sources and makes it possible to verify that the sources match the deployed module.`
{% endhint %}

{% hint style="info" %}
You should run Docker in your system and let it pull the required image.
{% endhint %}

{% hint style="info" %}
Cargo.lock file should not be included to .gitignore
{% endhint %}

**Deploy**

```
concordium-client module deploy module.wasm.v1 --sender <YOUR_ACCOUNT_ADDRESS>
 --name <YOUR_CONTRACT_NAME> --grpc-port 20000 --grpc-ip node.testnet.concordium.com
```

<figure><img src="../../../.gitbook/assets/Screenshot 2023-12-28 at 15.03.46.png" alt=""><figcaption></figcaption></figure>

**Initialize**

```
concordium-client contract init <MODULE-REF> --sender <YOUR-ACCOUNT-ADDRESS> --energy 30000 --contract cis2_dynamic_nft --grpc-port 20000 --grpc-ip node.testnet.concordium.com
```

<figure><img src="../../../.gitbook/assets/Screenshot 2023-12-28 at 15.15.16.png" alt=""><figcaption></figcaption></figure>

**Mint**

There will be 3 different metadata for each state of our pet from puppy to strong dog. Each metadata and image is uploaded to IPFS therefore publicly available for you and can be used for demo purposes.&#x20;

Little Puppy CIS2 Compatible Metadata File&#x20;

```json
{
  "name": "Little Puppy",
  "unique": true,
  "description": "My baby",
  "display": {
    "url": "https://ipfs.io/ipfs/Qmedq3GPcMtePE24NGrVmbi4XZRFnPTQNM47XnwismcBGx"
  },
  "attributes": [
    {
      "type": "string",
      "name": "Fluffy",
      "value": "999"
    }
  ]
}
```

Cool Boy CIS2 Compatible Metadata File

```json
{
  "name": "Cool Boy",
  "unique": true,
  "description": "Good looking, funny and my best friend.",
  "display": {
    "url": "https://ipfs.io/ipfs/QmWyTzZZKcEfhuW1hArMFhFacp5PBYC1tWN4sEjoAVYhWh"
  },
  "attributes": [
    {
      "type": "string",
      "name": "Charisma",
      "value": "999"
    }
  ]
}
```

Strong Dog CIS2 Compatible Metadata File

```json
{
  "name": "Strong Dog",
  "unique": true,
  "description": "Powerful, strong and my hero.",
  "display": {
    "url": "https://ipfs.io/ipfs/QmdxwBZ2vLTeS8bRaNLXYLZ28KtJi1x3q6a1eKuSyQKErG"
  },
  "attributes": [
    {
      "type": "string",
      "name": "Power",
      "value": "999"
    }
  ]
}
```

Excellent, now we should create all necessary JSON files to mint our token. First, let's create a JSON file for `MintParams`. Remember the struct we created in the smart contract, it requires an `owner` and corresponding `tokens` that holds a list of `MetadataUrl` for a token ID and the amount of it. Please note that, we will create it with only the first 2 metadata, to be able to test the `addMetadata` and `upgrade` functions.&#x20;

{% hint style="info" %}
Don't forget to upload the metadata files to IPFS as well, it means when you mint a token, you should give a hyperlink to the metadata file and the SHA(256) of the content optionally.&#x20;
{% endhint %}

```json
{
    "owner": {
        "Account": [
            "<YOUR-ADDRESS>"
        ]
    },
    "tokens": [
        [
            "01",
            {
                "metadata_url": [
                    {
                        "url": "https://ipfs.io/ipfs/Qmc7HPBm18uLEGcA7FZztEUzjc5UAbsqkXPHupNPfMw1SN",
                        "hash": {
                            "Some": [
                                "7cf0a806783a71d25d03dacb94c80109c3d5e66986b987cc6eb68c7da75cb1a6"
                            ]
                        }
                    },
                    {
                        "url": "https://ipfs.io/ipfs/QmX2QxwLvJkLqnEXvca6MpmLjNAPygenJwQg1iP4EdNKDg",
                        "hash": {
                            "Some": [
                                "197738868672d271a5fa81353b4d97ba977c3002ca095b482bdd74b89be22fb7"
                            ]
                        }
                    }
                ],
                "token_amount": "1"
            }
        ]
    ]
}
```

Now let's mint our token with the following command.

```
concordium-client contract update <CONTRACT-INDEX> --entrypoint mint --parameter-json mint-params.json --sender <YOUR-ACCOUNT> --energy 30000 --grpc-port 20000 --grpc-ip node.testnet.concordium.com
```

<figure><img src="../../../.gitbook/assets/Screenshot 2023-12-29 at 14.09.32.png" alt=""><figcaption></figcaption></figure>

Nice! We have successfully minted our token, lets go and check it from our wallet.

<figure><img src="../../../.gitbook/assets/Screenshot 2023-12-29 at 14.26.21.png" alt=""><figcaption></figcaption></figure>

**Upgrade**

Now, its is the big moment. Lets test whether this NFT is dynamic or not! We should be able to upgrade the metadata and evolve this little puppy into a cool boy! The `upgrade` function requires the token ID as a string in the JSON file.&#x20;

```json
"01"
```

Run the following command and see the magic!

```
concordium-client contract update <YOUR-CONTRACT-INDEX> --entrypoint upgrade --parameter-json upgrade.json --sender <YOUR-ADDRESS> --energy 30000 --grpc-port 20000 --grpc-ip node.testnet.concordium.com
```

<figure><img src="../../../.gitbook/assets/Screenshot 2023-12-29 at 14.32.47.png" alt=""><figcaption></figcaption></figure>

The transaction is completed. So excited, let's go and check!

<figure><img src="../../../.gitbook/assets/Screenshot 2023-12-29 at 14.36.51.png" alt=""><figcaption></figcaption></figure>

Isn't he gorgeous? :heart\_eyes: Our little puppy is now a cool boy!&#x20;

{% hint style="info" %}
Please keep this in mind, this step can be triggered by an oracle or any other interaction.&#x20;
{% endhint %}

**Add Metadata**

Now we are about to finalize our tutorial, one last step which is the addition of new metadata. Our cool boy will become a strong and majestic star!

Let's go and check the `AddParams` struct to create the right form of JSON file. It expects `tokens`that holds both token ID and `MetadataUrl` which is essentially `url` and `hash`.

```json
{
    "tokens": [
        [
            "01",
            {
                "url": "https://ipfs.io/ipfs/Qmc7HPBm18uLEGcA7FZztEUzjc5UAbsqkXPHupNPfMw1SN",
                "hash": {
                    "Some": [
                        "7cf0a806783a71d25d03dacb94c80109c3d5e66986b987cc6eb68c7da75cb1a6"
                    ]
                }
            }
        ]
    ]
}
```

Wait for the transaction's finalization.&#x20;

<figure><img src="../../../.gitbook/assets/Screenshot 2023-12-29 at 14.49.49.png" alt=""><figcaption></figcaption></figure>

Let's upgrade it once again with the same command in the previous step!

<figure><img src="../../../.gitbook/assets/Screenshot 2023-12-29 at 14.55.02.png" alt=""><figcaption></figcaption></figure>

Are you ready to see our cool boy? Well, he may have changed a little...

<figure><img src="../../../.gitbook/assets/Screenshot 2023-12-29 at 14.58.56.png" alt=""><figcaption></figcaption></figure>

**Token Metadata History**

Wow, it has been a great journey so far. Since the very first moment, we embarked on this journey we knew that our little puppy would become a tremendous loyal hero. :100:

Now, let's see what he has been through in all this time. We will invoke `tokenMetadataList` function to be able to see it. It accepts a vector of token IDs -see the following JSON-, to be able to return all of the tokens in the contract and their history in the form of `TokenMetadataList` struct which we defined as `Vec<Vec<MetadataUrl>>`.

```json
[
    "01"
] 
```

```
concordium-client contract invoke <YOUR-CONTRACT-INDEX> --entrypoint tokenMetadataList --parameter-json history.json --energy 30000 --grpc-port 20000 --grpc-ip node.testnet.concordium.com
```

And we will be able to see all the previous forms of the token we have from the contract state.&#x20;

<figure><img src="../../../.gitbook/assets/Screenshot 2023-12-29 at 15.11.38.png" alt=""><figcaption></figcaption></figure>

