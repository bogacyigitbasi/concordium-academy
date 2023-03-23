# Install Concordium Wallet

Now you need a Concordium wallet. Use the Concordium Wallet for Web. The Concordium Wallet for Web uses a 24 word secret recovery phrase to secure your wallet. Make sure to protect your 24 word secret recovery phrase and store it in a secure place. Anyone who knows the secret recovery phrase can access your wallet.

Use [this link](https://chrome.google.com/webstore/detail/concordium-wallet/mnnkpffndmickbiakofclnpoiajlegmg?hl=en-US) to install a Concordium Wallet for Web in a chromium web browser. Follow [these instructions](https://developer.concordium.software/en/mainnet/net/browser-wallet/setup-browser-wallet.html#setup-bw) to install the extension. Configure it to run on testnet with an identity created from the Concordium testnet IP (shown below) and an account based on that identity. You don’t have to provide an ID to create an identity on testnet when selecting `Concordium testnet IP`. Test identities are meant for testnet testing only.

<figure><img src="https://developer.concordium.software/en/mainnet/_images/bw-idp-selection.png" alt=""><figcaption></figcaption></figure>

Use the Testnet faucet in your account to claim 2000 CCDs for testing purposes.

One thing to note is that if you click [![button with paper airplane](https://developer.concordium.software/en/mainnet/\_images/send-ccd1.png)](https://developer.concordium.software/en/mainnet/\_images/send-ccd1.png), you enter transaction window. This allows you to transfer CCDs. You can type the amount of CCD and the recipient’s address in this section. As you can see just below those textboxes, there is a value highlighting the “Estimated transaction fee” in CCD terms. This allows you to estimate the costs beforehand and it allows helps you to calculate your business expenses in the future.

<figure><img src="https://developer.concordium.software/en/mainnet/_images/tx-fee-in-bw.png" alt=""><figcaption></figcaption></figure>

After that step, you need to [export the keys](https://developer.concordium.software/en/mainnet/net/guides/export-key.html#export-key) for your wallet. Save the file on your local machine in the same folder as the rest of the repository. It will have a name like this \<YOUR PUBLIC ADDRESS>.export. You can open it with a text editor and see your signKey, verifyKey in there. Copy signKey and your address. You will use them while deploying and interacting with your contract.

<figure><img src="https://developer.concordium.software/en/mainnet/_images/bw-export-key.png" alt=""><figcaption></figcaption></figure>

When you export the key it creates a file named `<YOUR PUBLIC ADDRESS>.export`. Open it with a text editor and find your `signKey`, `verifyKey` in there. Copy the `signKey` and your address. You will use it while deploying and interacting with your contract.

<figure><img src="https://developer.concordium.software/en/mainnet/_images/bw-exported-key.png" alt=""><figcaption></figcaption></figure>

#### Import the key

You are ready to import your key into the `concordium-client` configuration. Transfer your wallet key export file to the place where you are running your `concordium-client` tool. Navigate to the folder as well.

Import your key into the `concordium-client` configuration:

```
concordium-client config account import <Wallet.export> --name <Your-Wallet-Name>.json
```

