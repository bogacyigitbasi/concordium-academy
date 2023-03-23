# Install concordium-client

You will use `concordium-client` as a command line tool to deploy, mint, and transfer. [Download it here](https://developer.concordium.software/en/mainnet/net/installation/downloads-testnet.html#concordium-node-and-client-download-testnet). Rename the package `concordium-client` in case it has some version annotation.



If you are not using Ubuntu/Linux as your operating system, the following screenshots will look different. Remember to adjust the following commands based on your operating system.

Go to the folder where you downloaded the `concordium-client`. You can check if you are in the correct folder when you see the output `concordium-client` from the command:

```
$ls | grep 'concordium-client'
```

<figure><img src="https://developer.concordium.software/en/mainnet/_images/pb_tutorial_10.png" alt=""><figcaption></figcaption></figure>

Alternatively, if you don’t want to navigate around in the folders, you can add the folder where the `concordium-client` tool is located to your PATH variable with the command: `export PATH="$HOME/path/to/your/concordium-client:$PATH"`. This allows you to use the following commands (such as `concordium-client --help`) without prepending them with `./`. Effectively, prepending with `./` searches for the executable package in the current directory while omitting `./` searches for the executable package in the PATH variable.

The package is not yet executable. You change this with the command:

```
$chmod +x concordium-client
```

<figure><img src="https://developer.concordium.software/en/mainnet/_images/pb_tutorial_8.png" alt=""><figcaption></figcaption></figure>

Check whether you can execute the `concordium-client` tool.

```
$./concordium-client --help
```

You should see some output that will help you in getting familiar with the `concordium-client` tool.

<figure><img src="https://developer.concordium.software/en/mainnet/_images/pb_tutorial_9.png" alt=""><figcaption></figcaption></figure>

The `concordium-client` the tool allows you to interact with your testnet node. You will find important commands that the `concordium-client` tool provides [here](https://developer.concordium.software/en/mainnet/net/references/concordium-client.html#concordium-client).
