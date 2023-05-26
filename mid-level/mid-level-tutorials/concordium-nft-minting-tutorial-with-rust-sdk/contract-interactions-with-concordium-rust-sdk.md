# Contract Interactions with Concordium-Rust-SDK

Let’s continue with the client development, first, we will create a new instance of our deployed contract with the module reference we have. For structuring things a bit, we will create another folder for our client with cargo.

```
cargo new cis2-rust-sdk-minting
```

As the beginning step, we need to add our dependencies to _Cargo.toml_ file. Add dependencies shown below including SDK, web client components, command line & error handlers, serialization, and time libraries.&#x20;

```
[dependencies]

concordium-rust-sdk="2"
tokio = { version = "1", features = ["full"] }
warp = "0.3"
log = "0.4.11"
env_logger = "0.9"
clap = { version = "4", features = ["derive"] }
anyhow = "1.0"
chrono = "0.4.19"
thiserror = "1"
structopt = "0.3.26"
serde = { version = "1.0", features = ["derive"] }
serde_json = "1.0"
strum = "0.24"
strum_macros = "0.24"
```

Then in _main.rs_ start adding the necessary libraries to our program, we need to command line argument parser (_clap_) for getting parameters and _structopt_ for parsing the input parameters as a struct. The _path_ library to manipulate file paths for example path of our wallet file, and many internal concordium-rust-sdk functions for making the transactions, serialization, accounts and more.&#x20;

```
use crate::clap::AppSettings;
use anyhow::Context;
use concordium_rust_sdk::{
    common::{self, types::TransactionTime, SerdeDeserialize, SerdeSerialize},
    smart_contracts::{
        common as concordium_std,
        common::Amount,
        types::{OwnedContractName, OwnedReceiveName},
    },
    types::{
        smart_contracts::{ModuleReference, OwnedParameter, WasmModule},
        transactions::{send, BlockItem, InitContractPayload, UpdateContractPayload},
        AccountInfo, ContractAddress, WalletAccount,
    },
    v2,
};
use std::path::PathBuf;
use structopt::*;

use strum_macros::EnumString;
```

For simplicity, we will create an enum that represents the actions that we would like to do. In that _Action_ enum, we specify the transaction types as following: _Deploy_, _Init_, _WithSchema_. In _Deploy Action_, we just need to specify our module's path and in _Init Action_, we need only the deployed module reference. In the _WithSchema Action,_ we need to specify the transaction parameter for minting, transferring, or viewing the contract state. Please note that all will be invokable with the schema file.

```
#[derive(StructOpt)]
enum Action {
    #[structopt(about = "Deploy the module")]
    Deploy {
        #[structopt(long = "module", help = "Path to the contract module.")]
        module_path: PathBuf,
    },
    #[structopt(about = "Initialize the CIS-2 NFT contract")]
    Init {
        #[structopt(
            long,
            help = "The module reference used for initializing the contract instance."
        )]
        module_ref: ModuleReference,
    },
    #[structopt(
        about = "Update the contract and set the provided  using JSON parameters and a \
                 schema."
    )]
    WithSchema {
        #[structopt(long, help = "Path of the JSON parameter.")]
        parameter: PathBuf,
        #[structopt(long, help = "Path to the schema.")]
        schema: PathBuf,
        #[structopt(long, help = "The contract to update.")]
        address: ContractAddress,
        #[structopt(long, help = "Transaction Type")]
        transaction_type_: TransactionType,
    },
}
```

We add an enum to distinguish all transactions that require a schema that comes with the _WithSchema_ parameter. We need the schema file for both state-changing and view functions (to print in a human-readable form).&#x20;

```
#[derive(StructOpt, EnumString)]
enum TransactionType {
    #[structopt(about = "Mint")]
    Mint,
    #[structopt(about = "Transfer")]
    Transfer,
    #[structopt(about = "TokenMetadata")]
    TokenMetadata,
    #[structopt(about = "View")]
    View,
}
```

We have a _TransactionResult_ enum that will be used for escaping an error for incompatible type error for returning different results from each match. Every state change after each invocation, including _init\_contract_, _deploy\_contract_ and _update\_contract,_ needs to be treated differently than the _tokenMetadata()_ and the _view()_ functions. In order to call these view functions -meaning won't cause any state changes- the _invoke\_instance_ function should be called which has a return type. So having a parent enum helps us to return the same types, but gives us the ability to manipulate each one individually.&#x20;

```

#[derive(Debug)]
enum TransactionResult {
    StateChanging(AccountTransaction<EncodedPayload>),
    None,
}
```

Now, we need to set the base configurations including node setup. Since we are going to deploy this contract to testnet, we use the testnet node gRPC endpoint as the default provided by Concordium which is _testnet.node.concordium.com._ We also need our key file path -the file exported from the wallet- and the _Action._ All these should be configurable from the terminal as an input parameter.

```
/// Node connection, key path and the action input struct
#[derive(StructOpt)]
struct App {
    #[structopt(
        long = "node",
        help = "GRPC interface of the node.",
        default_value = "http://node.testnet.concordium.com:20000"
    )]
    endpoint: v2::Endpoint,
    #[structopt(long = "account", help = "Path to the account key file.")]
    keys_path: PathBuf,
    #[structopt(subcommand, help = "The action you want to perform.")]
    action: Action,
}
```

Now we can create our _main()_ function. As you can see from the code below, it is a multi-threaded runtime that can handle multiple requests simultaneously. It reads the inputs from the terminal and creates a connection with Concordium by creating a client. We upload our key file by providing its path, and get the nonce of the last finalized block to have the full list of the accounts onboarded. Then we check the _action_ type, to decide whether this is going to be a _Deploy, Init_ or W_ithSchema_ transaction in a _match_ or switch case statement. (in rust there is no switch case statement). Let’s start coding with _Deploy_ and _Init_ first, then continue with _WithSchema._

```
#[tokio::main(flavor = "multi_thread")]
async fn main() -> anyhow::Result<()> {
    use base64::{engine::general_purpose, Engine as _};
    let app = {
        let app = App::clap().global_setting(AppSettings::ColoredHelp);
        let matches = app.get_matches();
        App::from_clap(&matches)
    };

    let mut client = v2::Client::new(app.endpoint)
        .await
        .context("Cannot connect.")?;

    // load account keys and sender address from a file
    let keys: WalletAccount =
        WalletAccount::from_json_file(app.keys_path).context("Could not read the keys file.")?;

    // Get the initial nonce at the last finalized block.
    let acc_info: AccountInfo = client
        .get_account_info(&keys.address.into(), &v2::BlockIdentifier::Best)
        .await?
        .response;

    let nonce = acc_info.account_nonce;
    // set expiry to now + 5min
    let expiry: TransactionTime =
        TransactionTime::from_seconds((chrono::Utc::now().timestamp() + 300) as u64);
```

**Deploy Contract**

In order to deploy the contract and all other transactions, we use the _send()_ wrapper from the concordium-rust-sdk under the _transactions_ library. We read the wasm compiled smart contract module, and after deserializing it invoke the _deploy\_module()_ function from the same library. For structuring the our directory a bit better, we created a folder called _nft-params_ and copy and paste the exported wallet file and the module from “_concordium-out_” into it.

```
let tx = match app.action {
        Action::Deploy { module_path } => {
            let contents = std::fs::read(module_path).context("Could not read contract module.")?;
            let payload: WasmModule =
                common::Deserial::deserial(&mut std::io::Cursor::new(contents))?;
            TransactionResult::StateChanging(send::deploy_module(
                &keys,
                keys.address,
                nonce,
                expiry,
                payload,
            ))
        }
```

<figure><img src="https://cdn-images-1.medium.com/max/1600/1*A66lNNWx3qcy8Pj2l9uWLg.png" alt=""><figcaption></figcaption></figure>

Let’s build our file first, then run the executable in the target/debug folder with the below command.&#x20;

```
cargo build
cd target/debug
./cis2-rust-sdk-minting --account ../../nft-params/wallet.export deploy --module ../../nft-params/module.wasm.v1
```

<figure><img src="https://cdn-images-1.medium.com/max/1600/1*mwpNyoR8K0Svm-ch0n9ljg.png" alt=""><figcaption></figcaption></figure>

Congrats! We have successfully deployed our smart contract!

**Init Contract**

Now we will create a new instance of the deployed contract. In the first _match,_ we check whether the _action_ is _Init_ and then we add an empty _OwnedParam_ this is because our smart contract init function doesn't require an input parameter and similarly there is no _Amount_ for this function as a payment. But the init function itself requires the _module_ _reference_ that we had in the previous step. Use that and call the _init\_contract()_ function from _send_ wrapper of the _transactions_ library.

```
Action::Init {
            module_ref: mod_ref,
        } => {
            let param = OwnedParameter::empty();
            //                 .expect("Known to not exceed parameter size limit.");
            let payload = InitContractPayload {
                amount: Amount::zero(),
                mod_ref,
                init_name: OwnedContractName::new_unchecked(
                    "init_rust_sdk_minting_tutorial".to_string(),
                ),
                param,
            };
            TransactionResult::StateChanging(send::init_contract(
                &keys,
                keys.address,
                nonce,
                expiry,
                payload,
                10000u64.into(),
            ))
        }
```

```
./cis2-rust-sdk-minting --account ../../nft-params/wallet.export init --module-ref <YOUR-MODULE-REFERENCE> 
```

<figure><img src="https://cdn-images-1.medium.com/max/1600/1*5-alH3wPbMg2_RdE0yFmGw.png" alt=""><figcaption></figcaption></figure>

In the following sections, we will use the schema file either while changing the state with _transfer(),_ _mint()_ functions or print return values in the form of JSON from the contract.&#x20;

**Using Schema in View and State Changing Functions**

We will need the schema file when calling _mint()_ and _transfer()_ functions and any view function’s printing including _tokenMetadata()_ and _view()_. First, we need to read and load schema from the _.bs64_ output file, for convenience copy and paste it from “_concordium-out_” to “_nft-params_” folder. Please note that base64 encoding is without padding, so we decode it accordingly. Then we have the _TransactionType_ enum which helps us to distinguish the transactions cause each one needs different parameters, invokes different functions, and uses different parts of the schema.&#x20;

For the sake of the _match_ statement’s return type mismatch error, after every transaction the return type is _TransactionResult._ Depending on the transaction it returns _TransactionResult::StateChanging (_If it’s a mint or transfer) or _TransactionResult::None (_If it’s a view function_)_

```
Action::WithSchema {
            parameter,
            schema,
            address,
            transaction_type_,
        } => {
            let parameter: serde_json::Value = serde_json::from_slice(
                &std::fs::read(parameter.unwrap()).context("Unable to read parameter file.")?,
            )
            .context("Unable to parse parameter JSON.")?;

            let schemab64 = std::fs::read(schema).context("Unable to read the schema file.")?;
            let schema_source = general_purpose::STANDARD_NO_PAD.decode(schemab64);

            let schema = concordium_std::from_bytes::<concordium_std::schema::VersionedModuleSchema>(
                &schema_source?,
            )?;
            // schema_global = schema;
            match transaction_type_ {
                TransactionType::Mint => {
                    let param_schema =
                        schema.get_receive_param_schema("rust_sdk_minting_tutorial", "mint")?;
                    let serialized_parameter = param_schema.serial_value(&parameter)?;
                    let message = OwnedParameter::try_from(serialized_parameter).unwrap();
                    let payload = UpdateContractPayload {
                        amount: Amount::zero(),
                        address,
                        receive_name: OwnedReceiveName::new_unchecked(
                            "rust_sdk_minting_tutorial.mint".to_string(),
                        ),
                        message,
                    };

                    TransactionResult::StateChanging(send::update_contract(
                        &keys,
                        keys.address,
                        nonce,
                        expiry,
                        payload,
                        10000u64.into(),
                    ))
                }
                //// Transfer Transaction which changes the state
                TransactionType::Transfer => {
                    let param_schema =
                        schema.get_receive_param_schema("rust_sdk_minting_tutorial", "transfer")?;
                    let serialized_parameter = param_schema.serial_value(&parameter)?;
                    let message = OwnedParameter::try_from(serialized_parameter).unwrap();
                    let payload = UpdateContractPayload {
                        amount: Amount::zero(),
                        address,
                        receive_name: OwnedReceiveName::new_unchecked(
                            "rust_sdk_minting_tutorial.transfer".to_string(),
                        ),
                        message,
                    };
                    //// call update contract with the payload
                    TransactionResult::StateChanging(send::update_contract(
                        &keys,
                        keys.address,
                        nonce,
                        expiry,
                        payload,
                        10000u64.into(),
                    ))
                }
                /// Token Metadata function with no state change
                TransactionType::TokenMetadata => {
                    let param_schema = schema
                        .get_receive_param_schema("rust_sdk_minting_tutorial", "tokenMetadata")?;
                    let rv_schema = schema.get_receive_return_value_schema(
                        "rust_sdk_minting_tutorial",
                        "tokenMetadata",
                    )?;

                    let serialized_parameter = param_schema.serial_value(&parameter)?;
                    let context = ContractContext {
                        invoker: None, //Account(AccountAddress),
                        contract: address,
                        amount: Amount::zero(),
                        method: OwnedReceiveName::new_unchecked(
                            "rust_sdk_minting_tutorial.tokenMetadata".to_string(),
                        ),
                        parameter: OwnedParameter::try_from(serialized_parameter).unwrap(), //Default::default(),
                        energy: 1000000.into(),
                    };
                    // invoke instance
                    let info = client
                        .invoke_instance(&BlockIdentifier::Best, &context)
                        .await?;

                    match info.response {
                            concordium_rust_sdk::types::smart_contracts::InvokeContractResult::Success { return_value, .. } => {
                                let bytes: concordium_rust_sdk::types::smart_contracts::ReturnValue = return_value.unwrap();
                                // deserialize and print return value
                                println!( "{}",rv_schema.to_json_string_pretty(&bytes.value)?);//jsonxf::pretty_print(&param_schema.to_json_string_pretty(&bytes.value)?).unwrap());
                            }
                            _ => {
                                println!("Could'nt succesfully invoke the instance. Check the parameters.")
                            }
                        }
                    TransactionResult::None

                    // info
                }
                TransactionType::View => {
                    let rv_schema = schema
                        .get_receive_return_value_schema("rust_sdk_minting_tutorial", "view")?;

                    let context = ContractContext {
                        invoker: None, //Account(AccountAddress),
                        contract: address,
                        amount: Amount::zero(),
                        method: OwnedReceiveName::new_unchecked(
                            "rust_sdk_minting_tutorial.view".to_string(),
                        ),
                        parameter: Default::default(),
                        energy: 1000000.into(),
                    };
                    // invoke instance
                    let info = client
                        .invoke_instance(&BlockIdentifier::Best, &context)
                        .await?;

                    match info.response {
                            concordium_rust_sdk::types::smart_contracts::InvokeContractResult::Success { return_value, .. } => {
                                let bytes: concordium_rust_sdk::types::smart_contracts::ReturnValue = return_value.unwrap();
                                // deserialize and print return value
                                println!( "{}",rv_schema.to_json_string_pretty(&bytes.value)?);//jsonxf::pretty_print(&param_schema.to_json_string_pretty(&bytes.value)?).unwrap());
                            }
                            _ => {
                                println!("Could'nt succesfully invoke the instance. Check the parameters.")
                            }
                        }
                    TransactionResult::None

                    // info
                }
            }
        } 
```

Finally, for the transaction output, we have one final _match_ statement with _TransactionResult,_ which will print the transaction details including module reference when deployed, contract address when initialized, and rejection reason if it's rejected by looking at the _BlockSummaryDetails_. The program will print the view functions’ returns in the previous section so in this final _match_ they are just gracefully exiting.

```
    match tx {
        TransactionResult::StateChanging(result) => {
            let item = BlockItem::AccountTransaction(result);
            // submit the transaction to the chain
            let transaction_hash = client.send_block_item(&item).await?;
            println!(
                "Transaction {} submitted (nonce = {}).",
                transaction_hash, nonce,
            );
            let (bh, bs) = client.wait_until_finalized(&transaction_hash).await?;
            println!("Transaction finalized in block {}.", bh);

            match bs.details {
                BlockItemSummaryDetails::AccountTransaction(ad) => {
                    match ad.effects {
                        AccountTransactionEffects::ModuleDeployed { module_ref } => {
                            println!("module ref is {}", module_ref);
                        }
                        AccountTransactionEffects::ContractInitialized { data } => {
                            println!("Contract address is {}", data.address);
                        }
                        AccountTransactionEffects::None {
                            transaction_type,
                            reject_reason,
                        } => {
                            println!("The Rejection Outcome is {:#?}", reject_reason);
                        }
                        _ => (),
                    };
                }
                BlockItemSummaryDetails::AccountCreation(_) => (),
                BlockItemSummaryDetails::Update(_) => {
                    println!("Transaction finalized in block {:?}.", bs.details);
                    ()
                }
            };
        }
         TransactionResult::None => {
            println!("No state changes, already printed, gracefully exiting.");
        }
    }
```

\


**Mint Function**

Now we can call the _mint()_ function from our new instance. For the complete minting tutorial you can follow [this](https://developer.concordium.software/en/mainnet/smart-contracts/tutorials/nft-minting/index.html) from our developer portal. Let’s create a file to mint our token in the _nft-params_ folder called _nft-params.json_ similar to the tutorial and add your address and token ID. And copy the schema file from _concordium-out_ folder to the _nft-params._

```
{
    "owner": {
        "Account": ["YOUR-ACCOUNT-ADDRESS"]
    },
    "tokens": ["TOKEN-ID"]
}
```

If you want to check the parameters you can always use _— help_ keyword.&#x20;

<figure><img src="https://cdn-images-1.medium.com/max/1600/1*K2ixoZ0QkoADNpo31ItYiQ.png" alt=""><figcaption></figcaption></figure>

We will call _with-schema_ which requires the contract address, parameters, schema, and transaction type. Since there could be multiple transaction types like mint, transfer, view, burn, etc. we have added another enum _TransactionType_ specification of the transaction type is necessary while starting the program using the command line. We are also expected to provide the JSON parameters and the schema file both will be read from the provided path. If you need more details use _— help_ again.&#x20;

<figure><img src="https://cdn-images-1.medium.com/max/1600/1*iC9yjMrxqB_SJs-vktYyhQ.png" alt=""><figcaption></figcaption></figure>

Use the command below to invoke the _mint()_ function.

```
./cis2-rust-sdk-minting --account ../../nft-params/wallet.export with-schema --address "<INDEX,SUBINDEX>" --parameter ../../nft-params/nft-params.json --schema ../../nft-params/module-schema.bs64 --transaction-type Mint
```

<figure><img src="https://cdn-images-1.medium.com/max/1600/1*DdmKs2xj_MdEt0JJwBTvEg.png" alt=""><figcaption></figcaption></figure>

Congrats! You have successfully minted your first token using Concordium Rust-SDK!

**TokenMetadata Function**

Let’s check our token’s metadata URL. We should invoke the _tokenMetada()_ function of _cis2-nft,_ it requires _token\_id._ Create a JSON file like below and add any _token\_id_s to send as a parameter.

```
[
    "TOKEN-ID",
    "TOKEN-ID"
]
```

```
./cis2-rust-sdk-minting --account ../../nft-params/wallet.export with-schema --address "<INDEX,SUBINDEX>" --parameter ../../nft-params/token-id.json --schema ../../nft-params/module-schema.bs64 --transaction-type TokenMetadata
```

<figure><img src="https://cdn-images-1.medium.com/max/1600/1*k2iHIfhi4cgz17guVOe7LQ.png" alt=""><figcaption></figcaption></figure>

**View Function**

Finally, we will invoke the _view()_ function, which simply returns the current state of the contract instance. It doesn't necessarily require a parameter to be invoked, but our program waits for a parameter so you can use the same _token\_id.json_ file to display the state.&#x20;

```
./cis2-rust-sdk-minting --account ../../nft-params/wallet.export with-schema --address "<INDEX,SUBINDEX>" --parameter ../../nft-params/token-id.json --schema ../../nft-params/module-schema.bs64 --transaction-type View
```

<figure><img src="https://cdn-images-1.medium.com/max/1600/1*8dC9EjoIz_LUxzhp3TQI2w.png" alt=""><figcaption></figcaption></figure>

Congrats! You have successfully completed Concordium NFT Minting Tutorial with Rust-SDK! The full code can be found [here](https://github.com/bogacyigitbasi/nft-rust-sdk).&#x20;
