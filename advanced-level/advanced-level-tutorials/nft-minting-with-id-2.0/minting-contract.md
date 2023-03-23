# Minting Contract

We are going to update the _cis2-multi_ contract in this [link](https://github.com/Concordium/concordium-rust-smart-contracts/tree/main/examples/cis2-multi). First, remind us what we want to achieve. We would like to have a minting dApp that uses Concordium’s ID layer and according to our scenario, you should prove that you are older than 18 in order to mint a token. The most obvious solution to this is could be to ask for ID proofs from the smart contract but the proofs are not available to use like that from directly the contract. Instead, to create the same logic using _the_ _verifier_ backend server we will implement it in the following:

* We assume the owner of _the verifier_ and the smart contract instance owner (dApp owner) is the same person. When we run _the verifier_ backend server, we will use an accounts sign and verify keys.
* While creating a new instance of the contract, the owner has to send its verify key(public key) and the contract will keep it in the state. (To verify the signature)&#x20;
* When a user wants to mint a token, the dApp will ask for a _challenge_ and a _statement_ from _the verifier_.&#x20;
* Using that challenge, the dApp expects that the user accepts the information from his/her wallet that is going to be shared with it.&#x20;
* The proof will be shared with the verifier backend, and if they are verified then _the verifier_ will sign the public address of the user with its sign key which is given as an input parameter while starting the server.&#x20;
* This signature will be the input of the minting function and using the public key stored in the state, the smart contract will verify that it's coming from the verifier and is valid really.
* The _mint()_ function will be executed.

Create a project for your smart contract called cis2-multi.

```
cargo new cis2-multi
```

Add the following dependencies to _Cargo.toml_ file.

```
[features]
default = ["std"]
std = ["concordium-std/std", "concordium-cis2/std"]

[dependencies]
concordium-std = { version = "*", default-features = false }
concordium-cis2 = { version = "*", default-features = false }
hex = "*"

[lib]
crate-type=["cdylib", "rlib"]

[profile.release]
codegen-units = 1
opt-level = "s"
```

<figure><img src="https://cdn-images-1.medium.com/max/1600/1*xyK9FP7wRlHr0dqbsTk2Vg.png" alt=""><figcaption></figcaption></figure>

Create a _lib.rs_ file and copy\&paste the code from the example contract. Start modification with the _State_ struct and add a variable for the _verify\_key_ and add it to the state’s _empty()_ function. Please note that, unlike a regular _empty()_ function, this one takes the _verify\_key_ as a parameter.

```
/// The contract state,
///
/// Note: The specification does not specify how to structure the contract state
/// and this could be structured in a more space efficient way.
#[derive(Serial, DeserialWithState, StateClone)]
#[concordium(state_parameter = "S")]
struct State<S> {
    /// The state of addresses.
    state: StateMap<Address, AddressState<S>, S>,
    /// All of the token IDs
    tokens: StateMap<ContractTokenId, MetadataUrl, S>,
    /// Map with contract addresses providing implementations of additional
    /// standards.
    implementors: StateMap<StandardIdentifierOwned, Vec<ContractAddress>, S>,
    verify_key: PublicKeyEd25519,
}


impl<S: HasStateApi> State<S> {
    /// Construct a state with no tokens
    fn empty(state_builder: &mut StateBuilder<S>, verify_key: PublicKeyEd25519) -> Self {
        State {
            state: state_builder.new_map(),
            tokens: state_builder.new_map(),
            implementors: state_builder.new_map(),
            verify_key,
        }
    }
/// rest of the state functions below
}
```

While creating a new instance, we will need to store the _verify\_key_ or the public key of the owner. So create a struct for getting that input parameter called _InitParam._

```

#[derive(Serial, Deserial, SchemaType)]
struct InitParams {
    verify_key: PublicKeyEd25519,
}
```

And for the last part of the state, while minting we will require the signature in order to verify so our mint parameters should send it, add it like below.&#x20;

```
/// The parameter for the contract function `mint` which mints a number of
/// token types and/or amounts of tokens to a given address.
#[derive(Serial, Deserial, SchemaType)]
struct MintParams {
    /// Owner of the newly minted tokens.
    owner: Address,
    /// A collection of tokens to mint.
    tokens: collections::BTreeMap<ContractTokenId, (TokenMetadata, ContractTokenAmount)>,
    /// Signature from the owner of the contract
    signature: SignatureEd25519,
}
```

Finally, we will update the _mint()_ function. Basically add&#x20;

```
#[receive(
    contract = "CIS2-Multi",
    name = "mint",
    crypto_primitives,
    parameter = "MintParams",
    error = "ContractError",
    enable_logger,
    mutable
)]
fn contract_mint<S: HasStateApi>(
    ctx: &impl HasReceiveContext,
    host: &mut impl HasHost<State<S>, StateApiType = S>,
    logger: &mut impl HasLogger,
    crypto_primitives: &impl HasCryptoPrimitives,
) -> ContractResult<()> {
    // Get the sender of the transaction
    let sender: AccountAddress = match ctx.sender() {
        Address::Account(a) => a,
        Address::Contract(_) => bail!(ContractError::Custom(CustomContractError::AccountOnly)),
    };

    // Parse the parameter.
    let params: MintParams = ctx.parameter_cursor().get()?;

    let (state, builder) = host.state_and_builder();

    // Verifying that the signature belongs to the public key which was added at the time of init.
    ensure!(
        crypto_primitives.verify_ed25519_signature(state.verify_key, params.signature, &sender.0),
        ContractError::Unauthorized
    );

    for (token_id, token_info) in params.tokens {
        ensure!(
            state.contains_token(&token_id).eq(&false),
            ContractError::Custom(CustomContractError::TokenAlreadyMinted)
        );

        // Mint the token in the state.
        state.mint(
            &token_id,
            &token_info.0,
            token_info.1,
            &params.owner,
            builder,
        );

        // Event for minted token.
        logger.log(&Cis2Event::Mint(MintEvent {
            token_id,
            amount: token_info.1,
            owner: params.owner,
        }))?;

        // Metadata URL for the token.
        logger.log(&Cis2Event::TokenMetadata::<_, ContractTokenAmount>(
            TokenMetadataEvent {
                token_id,
                metadata_url: token_info.0.to_metadata_url(),
            },
        ))?;
    }
    Ok(())
}
```

Nice, we are done with the smart contract. Without slowing down, now the front-end development starts.
