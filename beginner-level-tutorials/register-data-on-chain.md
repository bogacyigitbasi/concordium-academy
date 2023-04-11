# 📠 Register Data On-Chain

Hello, in this tutorial we will implement a very basic react application that can connect to the Concordium blockchain. It will not have any smart contract development and/or interaction. (Even without running a node!) Concordium provides a very special transaction type to store data on-chain without even coding a smart contract. The simplest use case could be, for example, if you would like to make sure that a data’s integrity is not tampered with, what can you do you can store that data’s hash on-chain and verify it later!

## React Project <a href="#e5b4" id="e5b4"></a>

First, set up a working directory for our dApp and then create an empty react project from the template inside it. I assume you already have _yarn_ installed in your system and have a text editor. I have been using _vscode_ for many years now. To create a react project run the command below.

```
yarn create react-app register-data-dapp --template typescript
```

It will take some time to fetch & install all packages and dependencies, and when it’s all done you will have something like the below.

<figure><img src="https://miro.medium.com/max/1400/0*-UhfuUawNmLwpWb1.png" alt=""><figcaption></figcaption></figure>

And when you run it you will see the template application interface.

<figure><img src="https://miro.medium.com/max/1400/0*OPzLYj12SxJQtRgi.png" alt=""><figcaption></figcaption></figure>

We will start from scratch with an empty _React_ application that is bootstrapped from React template and will be using _material-ui._ We will add the dependencies for some react components from _material-ui_ and necessary libraries from _concordium-web-sdk_ and _concordium-web-wallet-helper_. To do that run the command below and _yarn_ will install all specified packages.

```
yarn add @mui/material @emotion/react @mui/icons-material @emotion/styled @concordium/web-sdk @concordium/browser-wallet-api-helpers
```

Once you complete that, it will create a _package.json_ file that includes all dependencies in it.

<figure><img src="https://miro.medium.com/max/1400/0*bmiVFBABlwxDm7qH.png" alt=""><figcaption></figcaption></figure>

**Header Component**

Let’s create a file called Header.tsx that will have a button and handle our _connect()_ to the web wallet and paste the code below.

```
import {
 detectConcordiumProvider,
 WalletApi,
} from "@concordium/browser-wallet-api-helpers";
import { AppBar, Toolbar, Typography, Button } from "@mui/material";
import { useState } from "react";
export default function Header(props: {
 onConnected: (provider: WalletApi, account: string) => void;
 onDisconnected: () => void;
}) {
 const [isConnected, setConnected] = useState(false);
 function connect() {
  detectConcordiumProvider()
   .then((provider) => {
    provider
     .connect()
     .then((account) => {
      setConnected(true);
      props.onConnected(provider, account!);
     })
     .catch((_) => {
      alert("Please allow wallet connection");
      setConnected(false);
     });
    provider.removeAllListeners();
    provider.on("accountDisconnected", () => {
     setConnected(false);
     props.onDisconnected();
    });
    provider.on("accountChanged", (account) => {
     props.onDisconnected();
     props.onConnected(provider, account);
     setConnected(true);
    });
    provider.on("chainChanged", () => {
     props.onDisconnected();
     setConnected(false);
    });
   })
   .catch((_) => {
    console.error(`could not find provider`);
    alert("Please download Concordium Wallet");
   });
 }
 return (
  <AppBar>
   <Toolbar>
    <Typography variant="h6" component="div" sx={{ flexGrow: 1 }}>
     Concordium Register Data
    </Typography>
    <Button color="inherit" onClick={connect} disabled={isConnected}>
     {isConnected ? "Connected" : "Connect"}
    </Button>
   </Toolbar>
  </AppBar>
 );
}
```

**Register Component**

Now create another file called Register.tsx, in this component, we will take input from the user with a text box, calculate its SHA-256 and store it onchain with a very simple and special type of transaction called _registerData._

```
import { detectConcordiumProvider } from "@concordium/browser-wallet-api-helpers";
import {
 AccountTransactionType,
 DataBlob,
 RegisterDataPayload,
 sha256,
} from "@concordium/web-sdk";
import { Button, Link, Stack, TextField, Typography } from "@mui/material";
import { FormEvent, useState } from "react";
import { Buffer } from "buffer/";

export default function RegisterData() {
 let [state, setState] = useState({
  checking: false,
  error: "",
  hash: "",
 });

 const submit = async (event: FormEvent<HTMLFormElement>) => {
  event.preventDefault();
  setState({ ...state, error: "", checking: true, hash: "" });
  const formData = new FormData(event.currentTarget);

  var formValues = {
   data: formData.get("data")?.toString() ?? "",
  };

  if (!formValues.data) {
   setState({ ...state, error: "Invalid Data" });
   return;
  }

  const provider = await detectConcordiumProvider();
  const account = await provider.connect();

  if (!account) {
   alert("Please connect");
  }

  try {
   const txnHash = await provider.sendTransaction(
    account!,
    AccountTransactionType.RegisterData,
    {
     data: new DataBlob(sha256([Buffer.from(formValues.data)])),
    } as RegisterDataPayload
   );

   setState({ checking: false, error: "", hash: txnHash });
  } catch (error: any) {
   setState({ checking: false, error: error.message || error, hash: "" });
  }
 };

 return (
  <Stack
   component={"form"}
   spacing={2}
   onSubmit={submit}
   autoComplete={"true"}
  >
   <TextField
    id="data"
    name="data"
    label="Data"
    variant="standard"
    disabled={state.checking}
   />
   {state.error && (
    <Typography component="div" color="error">
     {state.error}
    </Typography>
   )}
   {state.checking && <Typography component="div">Checking..</Typography>}
   {state.hash && (
    <Link
     href={`https://dashboard.testnet.concordium.com/lookup/${state.hash}`}
     target="_blank"
    >
     View Transaction <br />
     {state.hash}
    </Link>
   )}
   <Button
    type="submit"
    variant="contained"
    fullWidth
    size="large"
    disabled={state.checking}
   >
    Register Data
   </Button>
  </Stack>
 );
}
```

Specifically, let’s zoom in, on a part of the code below there are 2 important parts you need to be careful about, first the connection with the wallet and second the transaction parameters. To make sure these are provided, add a control that checks if the wallet is already connected successfully. Then, you need to give the data as a parameter to the _AccountTransactionType.RegisterData_ transaction. (convert it to sha256) Keep the returning transaction hash (_txnHash)_ to verify our transaction’s state and what it entails.

```
const provider = await detectConcordiumProvider();
  const account = await provider.connect();

  if (!account) {
   alert("Please connect");
  }

  try {
   const txnHash = await provider.sendTransaction(
    account!,
    AccountTransactionType.RegisterData,
    {
     data: new DataBlob(sha256([Buffer.from(formValues.data)])),
    } as RegisterDataPayload
   );

   setState({ checking: false, error: "", hash: txnHash });
  } catch (error: any) {
   setState({ checking: false, error: error.message || error, hash: "" });
  }
 };
```

Open App.tsx the file and paste the code below, basically you are calling these two components from the application’s main function.

```
import "./App.css";
import Header from "./Header";
import { useState } from "react";
import { Container } from "@mui/material";
import RegisterData from "./RegisterData";

export default function App() {
 const [isConnected, setConnected] = useState(false);

 return (
  <div className="App">
   <Header
    onConnected={() => setConnected(true)}
    onDisconnected={() => setConnected(false)}
   />
   <Container sx={{ mt: 15 }}>{isConnected && <RegisterData />}</Container>
  </div>
 );
}
```

If everything is correct, you should see something like the one below.

<figure><img src="https://miro.medium.com/max/1400/1*uuZypCFZqt4nP6ptNDrf6Q.png" alt=""><figcaption></figcaption></figure>

Connect it and you will have;

<figure><img src="https://miro.medium.com/max/1400/1*F_upUthy94tetN0tpGeR3A.png" alt=""><figcaption></figcaption></figure>

Let’s fill it with the string “concordium” and click the “REGISTER DATA” button.

<figure><img src="https://miro.medium.com/max/1400/1*ZPEU4_GbFo7bcQCqSOnaXg.png" alt=""><figcaption></figcaption></figure>

The application will print the _TxnHash_ value for us to track on the block explorer, click that to verify.

<figure><img src="https://miro.medium.com/max/1400/1*mBGvsK-USzhiVwYWJFGVNg.png" alt=""><figcaption></figcaption></figure>

It will redirect you to the dashboard, to that block in particular.

<figure><img src="https://miro.medium.com/max/1400/1*YUp5_CVXNOjIfk88C4FRGQ.png" alt=""><figcaption></figcaption></figure>

Let’s compare the data we stored in the Concordium SHA-256 version, right? You can use [this](https://emn178.github.io/online-tools/sha256.html) online tool to calculate any text’s SHA-256.

<figure><img src="https://miro.medium.com/max/1400/1*dmjts1A7uJLH8YRctHhYgg.png" alt=""><figcaption></figcaption></figure>

As expected, those two values are matched! Let’s step back for one second and think about what we implemented with this dApp. What we have done is basically, store proof of data on-chain, and allow everybody who wants to check that our data’s integrity is preserved or not can go and verify it by looking at this brilliant decentralized network. It is stored on-chain now and no one, even I can not change it. How cool is that? :) Full code can be found [here](https://github.com/bogacyigitbasi/registerdata).
