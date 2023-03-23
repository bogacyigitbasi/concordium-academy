# Run a docker testnet node

## Run a node with Docker

In this guide, you learn how to run a node on your Linux computer that participates in the Concordium network. This means that you receive blocks and transactions from other nodes, as well as propagate information about blocks and transactions to the nodes in the Concordium network. After following this guide, you will be able to

* run a Concordium node
* observe it on the network dashboard
* query the state of the Concordium blockchain directly from your machine.

You do not need an account to run a node.

Note

Subscribe to the [Mainnet status page](https://status.mainnet.concordium.software/) or [Testnet status page](https://status.testnet.concordium.software/) and the [release information on Discourse](https://support.concordium.software/c/releases/9) to stay informed about updates and changes that may affect you as a node runner, including node software releases and protocol updates.

To subscribe to updates on the Mainnet/Testnet status page click **Subscribe** to get all updates or click **Get updates** to choose to get all updates or only updates for specific products.

### Before you begin

Before running a Concordium node you will need to

1. Install and run Docker.
   * On _Linux_, allow Docker to be run as a non-root user.
2. [Install the compose plugin](https://docs.docker.com/compose/install/).

### Running/upgrading a node

Concordium provides two Docker images, a [mainnet](https://hub.docker.com/r/concordium/mainnet-node) one and a [testnet](https://hub.docker.com/r/concordium/testnet-node) one. These images are designed to be used together with docker-compose, or a similar driver. This guide provides a sample configuration using `docker-compose`.

The node requires a database which must be stored on the host system so that it persists when the docker container is stopped. It is up to the user to select the location of the database on their host system. In the guide the location used is `/var/lib/concordium-mainnet` or `/var/lib/concordium-testent` but any location to which the user that runs the Docker command has access to will do.

Note

Node version 4.5.0 introduced the GRPC V2 interface which is enabled by the sample configurations listed below. If you have done special configuration of your node and want to re-use the configuration file and have the new API enabled, make sure to edit your configuration, adding `CONCORDIUM_NODE_GRPC2_LISTEN_PORT` and `CONCORDIUM_NODE_GRPC2_LISTEN_ADDRESS` as in the sample configurations.

Note

When upgrading, you can only upgrade one minor version at a time, or from the last release of major version X to major version X+1. You cannot skip versions. For patches, you can skip versions e.g. X.X.0 to X.X.3, or _X.1.1_ to _X.2.3_.

If you are running version 4.2.3 you can [migrate to the latest version](https://developer.concordium.software/en/mainnet/net/guides/run-node.html#migration-docker-distribution). If you are running any version older than 4.2.3 you will have to delete your database and start over using the instructions on this page.

### Run a testnet node

To run a node on testnet use the following configuration file and follow the steps below.

```
# This is an example configuration for running the testnet node
version: '3'
services:
  testnet-node:
    container_name: testnet-node
    image: concordium/testnet-node:latest
    pull_policy: always
    environment:
      # Environment specific configuration
      # The url where IPs of the bootstrap nodes can be found.
      - CONCORDIUM_NODE_CONNECTION_BOOTSTRAP_NODES=bootstrap.testnet.concordium.com:8888
      # Where the genesis is located
      - CONCORDIUM_NODE_CONSENSUS_GENESIS_DATA_FILE=/testnet-genesis.dat
      # The url of the catchup file. This speeds up the catchup process.
      - CONCORDIUM_NODE_CONSENSUS_DOWNLOAD_BLOCKS_FROM=https://catchup.testnet.concordium.com/blocks.idx
      # General node configuration Data and config directories (it's OK if they
      # are the same). This should match the volume mount below. If the location
      # of the mount inside the container is changed, then these should be
      # changed accordingly as well.
      - CONCORDIUM_NODE_DATA_DIR=/mnt/data
      - CONCORDIUM_NODE_CONFIG_DIR=/mnt/data
      # The port on which the node will listen for incoming connections. This is a
      # port inside the container. It is mapped to an external port by the port
      # mapping in the `ports` section below. If the internal and external ports
      # are going to be different then you should also set
      # `CONCORDIUM_NODE_EXTERNAL_PORT` variable to what the external port value is.
      - CONCORDIUM_NODE_LISTEN_PORT=8889
      # Desired number of nodes to be connected to.
      - CONCORDIUM_NODE_CONNECTION_DESIRED_NODES=5
      # Maximum number of __nodes__ the node will be connected to.
      - CONCORDIUM_NODE_CONNECTION_MAX_ALLOWED_NODES=10
      # Address of the GRPC server
      - CONCORDIUM_NODE_RPC_SERVER_ADDR=0.0.0.0
      # And its port
      - CONCORDIUM_NODE_RPC_SERVER_PORT=10001
      # Address of the V2 GRPC server
      - CONCORDIUM_NODE_GRPC2_LISTEN_PORT=20001
      # And its port
      - CONCORDIUM_NODE_GRPC2_LISTEN_ADDRESS=0.0.0.0
      # Maximum number of __connections__ the node can have. This can temporarily be more than
      # the number of peers when incoming connections are processed. This limit
      # ensures that there cannot be too many of those.
      - CONCORDIUM_NODE_CONNECTION_HARD_CONNECTION_LIMIT=20
      # Number of threads to use to process network events. This should be
      # adjusted based on the resources the node has (in combination with
      # `CONCORDIUM_NODE_RUNTIME_HASKELL_RTS_FLAGS`) below.
      - CONCORDIUM_NODE_CONNECTION_THREAD_POOL_SIZE=2
      # The bootstrapping interval in seconds. This makes the node contact the
      # specified bootstrappers at a given interval to discover new peers.
      - CONCORDIUM_NODE_CONNECTION_BOOTSTRAPPING_INTERVAL=1800
      # Haskell RTS flags to pass to consensus. `-N2` means to use two threads
      # for consensus operations. `-I0` disables the idle garbage collector
      # which reduces CPU load for non-baking nodes.
      - CONCORDIUM_NODE_RUNTIME_HASKELL_RTS_FLAGS=-N2,-I0
    entrypoint: ["/concordium-node"]
    # Exposed ports. The ports the node listens on inside the container (defined
    # by `CONCORDIUM_NODE_LISTEN_PORT` and `CONCORDIUM_NODE_RPC_SERVER_PORT`)
    # should match what is defined here. When running multiple nodes the
    # external ports should be changed so as not to conflict.
    # In the mapping below, the first port is the `host` port, and the second
    # port is the `container` port. When the `container` port is changed the
    # relevant environment variable listed above must be changed as well. For
    # example, changing `10001:10001` to `10001:13000` would mean that
    # `CONCORDIUM_NODE_RPC_SERVER_PORT` should be set to `13000`. Otherwise
    # the node's gRPC interface will not be available from the host.
    ports:
    - "8889:8889"
    - "10001:10001"
    - "20001:20001"
    volumes:
    # The node's database should be stored in a persistent volume so that it
    # survives container restart. In this case we map the **host** directory
    # /var/lib/concordium-testnet to be used as the node's database directory.
    - /var/lib/concordium-testnet:/mnt/data
  # The collector reports the state of the node to the network dashboard. A node
  # can run without reporting to the network dashboard. Remove this section if
  # that is desired.
  testnet-node-collector:
    container_name: testnet-node-collector
    image: concordium/testnet-node:latest
    pull_policy: always
    environment:
      # Settings that should be customized by the user.
      - CONCORDIUM_NODE_COLLECTOR_NODE_NAME=docker-test
      # Environment specific settings.
      - CONCORDIUM_NODE_COLLECTOR_URL=https://dashboard.testnet.concordium.com/nodes/post
      # Collection settings.
      # How often to collect the statistics from the node.
      - CONCORDIUM_NODE_COLLECTOR_COLLECT_INTERVAL=5000
      # The URL where the node can be reached. Note that this will use the
      # docker created network which maps `testnet-node` to the internal IP of
      # the `testnet-node`. If the name of the node service is changed from
      # `testnet-node` then the name here must also be changed.
      - CONCORDIUM_NODE_COLLECTOR_GRPC_HOST=http://testnet-node:10001
    entrypoint: ["/node-collector"]
```

1. Save the contents as `testnet-node.yaml`.
2.  Possibly modify the **volume mount** to map the database directory to a different location on the host system. The volume mount is the following section.

    ```
    volumes:
       # The node's database should be stored in a persistent volume so that it
       # survives container restart. In this case we map the **host** directory
       # /var/lib/concordium-testnet to be used as the node's database directory.
       - /var/lib/concordium-testnet:/mnt/data
    ```
3.  Modify the node name that appears on the network dashboard. This is set by the environment variable

    ```
    - CONCORDIUM_NODE_COLLECTOR_NODE_NAME=docker-test
    ```

    This name can be set to any non-empty string. If the name has spaces it should be quoted.
4.  Start the node and the collector.

    ```
    $docker-compose -f testnet-node.yaml up
    ```

The configuration starts two containers, one running the node, and another running the node collector that reports the node state to the network dashboard.

If you wish to have the node running in the background, then add a `-d` option to the above command.

Note

The sample configuration always downloads the latest node image. It is good practice to choose the version deliberately. To choose a specific version, find the correct version in [hub.docker.com/concordium/testnet-node](https://hub.docker.com/r/concordium/testnet-node) and change the `image` value from

> ```
> image: concordium/testnet-node:latest
> ```

to, e.g.,

> ```
> image: concordium/testnet-node:4.2.3-0
> ```

#### Enable inbound connections

If you are running your node behind a firewall, or behind your home router, then you will probably only be able to connect to other nodes, but other nodes will not be able to initiate connections to your node. This is perfectly fine, and your node will fully participate in the Concordium network. It will be able to send transactions and, [if so configured](https://developer.concordium.software/en/mainnet/net/guides/become-baker.html#become-a-baker), to bake and finalize.

However you can also make your node an even better network participant by enabling inbound connections. The sample configuration above makes the node listen on port `8889` for inbound connections. Depending on your network and platform configuration you will either need to forward an external port to `8889` on your router, open it in your firewall, or both. The details of how this is done will depend on your configuration.

**Retrieve node logs**

The sample configuration presented above logs data using Docker’s default logging infrastructure. To retrieve the logs for the node run:

```
$docker logs testnet-node
```

This outputs the logs to `stdout`.

#### Migration from the previous Docker distribution

In the past Concordium provided a `concordium-software` package which contained a `concordium-node` binary which orchestrated downloading a Docker image and running the node. To migrate from that setup:

1. Stop the running node (e.g., using `concordium-node-stop`)
2.  Either modify the relevant example configuration file above by mapping the existing node database directory for use by the new container, i.e., replacing

    ```
    - /var/lib/concordium-testnet:/mnt/data
    ```

    with

    ```
    - ~/.local/share/concordium:/mnt/data
    ```

    Or, alternatively, moving the contents of `~/.local/share/concordium` to, e.g., `/var/lib/concordium-testnet` and keeping the configuration files as they are.
3.  If your node is an existing baker node, update the configuration file above to include

    ```
    - CONCORDIUM_NODE_BAKER_CREDENTIALS_FILE=/mnt/data/baker-credentials.json
    ```

    into the `environment` section of the `node` service section of the file.
4.  Start the node and the collector.

    ```
    $docker-compose -f testnet-node.yaml up
    ```

The configuration starts two containers, one running the node, and another running the node collector that reports the node state to the network dashboard.

If you wish to have the node running in the background, then add a `-d` option to the above command.

Note

The sample configuration always downloads the latest node image. It is good practice to choose the version deliberately. To choose a specific version, find the correct version in [hub.docker.com/concordium/testnet-node](https://hub.docker.com/r/concordium/testnet-node) and change the `image` value from

> ```
> image: concordium/testnet-node:latest
> ```

to, e.g.,

> ```
> image: concordium/testnet-node:4.5.0-0
> ```
