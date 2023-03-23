# Build & Deploy & Initialize

#### Deploy Fractionalizer Smart Contract

So far so good! We have implemented our smart contract and now we are ready to build, deploy, and create an instance of it.&#x20;

**Build Smart Contract**

First, we need to build the smart contract into wasm format. Use the following command to do it. You can extract the _schema_ and wasm compiled file into a new folder called “dist” as we do in this example.

```

cargo concordium build --out dist/module.wasm.v1 --schema-out dist/schema.bin
```

####

<figure><img src="https://cdn-images-1.medium.com/max/1600/1*jGE2i-KC3A57qoH9IEDr7w.png" alt=""><figcaption></figcaption></figure>

**Deploy Smart Contract**

Second, we need to deploy the wasm compiled smart contract to the testnet. Use the following command to do it. We have some good news for you! From now on you don’t have to run your own testnet node for deploying contracts. Essentially, using a public URL (_node.testnet.concordium.com_), you can connect to a testnet node and will be able to deploy/query nodes. For all operations in this tutorial, we are going to use the public gRPC endpoint that is provided by Concordium.&#x20;

Please keep this in mind, for some use cases you might need to run your own local node as there could be some limitations of this one. If you need more info either check this link or connect with us from our [support](http://support.concordium.software) channel.

```
concordium-client module deploy dist/module.wasm.v1 --sender <YOUR-ADDRESS> --name CIS2-Fractionalizer --grpc-port 10000 --grpc-ip node.testnet.concordium.com
```

<figure><img src="https://cdn-images-1.medium.com/max/1600/1*HKHI7MeCSqCmBQYiWQf5Bw.png" alt=""><figcaption></figcaption></figure>

**Initialize Smart Contract**

Third, we need to create a new instance of this smart contract which will initialize our empty state that holds the assets and allow us to invoke all methods. Run the command below to create an instance of your deployed contract using the module reference.

```
concordium-client contract init <YOUR-MODULE-HASH> --sender <YOUR-ADDRESS> --energy 30000 --contract CIS2-Fractionalizer --grpc-port 10000 --grpc-ip node.testnet.concordium.com
```

<figure><img src="https://cdn-images-1.medium.com/max/1600/1*1Wz0C2gtyxAAlX4q9uOZRg.png" alt=""><figcaption></figcaption></figure>

Excellent. As a next step, let’s move to the following section.
