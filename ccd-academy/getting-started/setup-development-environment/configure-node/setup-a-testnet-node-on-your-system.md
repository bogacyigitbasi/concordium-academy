# Setup a testnet node on your system

{% hint style="info" %}
It is technically fine to run your testnet node locally on your machine instead of on a server in the cloud. Since blockchain nodes have to run 24/7 to be up-to-date with the blockchain, you have to run your local machine 24/7. Alternatively, if you don’t want to run your local machine 24/7, you can let your node catch up whenever you start your machine. Because this takes some time, the tutorials recommend a cloud provider setup for convenience.
{% endhint %}

You will need to run a node. You can run any node platform you wish. You can create an account on your favorite cloud provider to set up your instance unless you intend to run a testnet node locally on your machine. The following are the requirements to run a simple testnet node. See the [requirements](https://developer.concordium.software/en/mainnet/net/nodes/node-requirements.html#system-requirements-node-mainnet) for mainnet nodes.

| Hardware (Testnet node) | Recommended           |
| ----------------------- | --------------------- |
| CPU (Core)              | 2                     |
| RAM (Memory)            | 8 GB                  |
| Storage                 | 25 GB                 |
| Operating system        | Ubuntu 20.04 or 22.04 |

You can also run a Docker image of a node that can be [found here](https://developer.concordium.software/en/mainnet/net/guides/run-node.html#run-a-node). Docker file configurations can be found in the `docker-compose.yml` file as described [here](https://developer.concordium.software/en/mainnet/net/guides/run-node.html#run-a-node). Don’t forget the set a name for your node with the parameter `CONCORDIUM_COLLECTOR_NODE_NAME`.

#### Sync your node

Start the syncing process of the testnet node by following the guide for your platform [Ubuntu](https://developer.concordium.software/en/mainnet/net/nodes/ubuntu.html#ubuntu-node), [Docker](https://developer.concordium.software/en/mainnet/net/nodes/docker.html#docker-node), [Windows](https://developer.concordium.software/en/mainnet/net/nodes/windows.html#windows-node), or [MacOS](https://developer.concordium.software/en/mainnet/net/nodes/macos.html#macos-node). This step currently takes some time, potentially hours based on your device configuration, because your node is freshly started and needs to recover all the previous blocks.

You should find your node name on the [Concordium testnet dashboard](https://dashboard.testnet.concordium.com/). It will take less than a day until your testnet node is fully synced. You can observe the syncing process by watching the finalization length of your node. Wait until the `Fin Length` (finalization length) of your node is the same as the highest value used by the majority of nodes. Once the height value is the same as the height in [CCDScan](https://testnet.ccdscan.io/blocks), then you can continue with the development.

Note

To allow the network dashboard to display nodes in the network and their current status, a node must run a collector. The collector is enabled by the default start-up script but can be disabled.

<figure><img src="https://developer.concordium.software/en/mainnet/_images/pb_tutorial_13.png" alt=""><figcaption></figcaption></figure>

Alternatively, you can query the syncing state of your node with `concordium-client`. Wait until the `last finalized block height` of your node is the same as the highest value used by the majority of nodes.

```
./concordium-client consensus status --grpc-port 10001
```

<figure><img src="https://developer.concordium.software/en/mainnet/_images/pb_tutorial_19.png" alt=""><figcaption></figcaption></figure>

Note

It is a good practice to enable inbound connections on port 8889 (testnet) in your instance. You can allow inbound connections from any IPv4 and IPv6 address, by selecting `0.0.0.0/0` and `::/0` on the port 8889. This is not mandatory for the node to sync but it will make your node a good network participant. Feel free to skip this step if you are not feeling confident editing the inbound connection rules of your instance.

[![../../\_images/pb\_tutorial\_12.png](https://developer.concordium.software/en/mainnet/\_images/pb\_tutorial\_12.png)](https://developer.concordium.software/en/mainnet/\_images/pb\_tutorial\_12.png)

Remember you are working on the testnet. Check if your node collector is up and running in CCDScan. Look for the name of your node in the network section of the dashboard.

\


<figure><img src="https://developer.concordium.software/en/mainnet/_images/node-collector.png" alt=""><figcaption></figcaption></figure>
