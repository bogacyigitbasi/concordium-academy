---
description: >-
  We will modify the CIS-2 Multi contract in this example from the Concordium
  example smart contracts repository. The full modified contract can be found at
  the end of this section.
---

# 📜 Smart Contract Implementation

{% hint style="info" %}
Please keep this in mind, since this is a mid-level tutorial, we assume that you have completed the other token tutorials in the developer [portal](https://developer.concordium.software/en/mainnet/smart-contracts/tutorials/sft-minting/index.html). Therefore, we will not explain everything in detail.&#x20;
{% endhint %}

Before we start implementation of the contract let's go over the example scenario. We will use our beloved, loyal friends in this tutorial. Assume that there is a little puppy, that will evolve into a cool little boy and then become a strong dog. Whenever we want to display or pet we will see its latest form and the transformation can be triggered by only the contract owner. &#x20;

* We want our d-NFT contract to support multiple metadata.&#x20;
* The contract owner can add new metadata to the token and upgrade the existing metadata.&#x20;
* We want to see the history of the metadata list.&#x20;

### Smart Contract Modification

Initially, we will start with modifying the state, this step includes adjusting the state variables, and creating structs for required logic, getter, and setter functions.

#### Contract's State Operations

Instead of keeping a `MetadataUrl` in `tokens` that represents the URL of the metadata for that token, we will keep another struct called `TokenMetadataState`. This struct holds a counter value and a list of  `MetadataUrl`which gives us to have more than one `MetadataUrl` for a token. The counter keeps the current index value of the present `MetadataUrl` in the list. When it is upgraded we increment it, and the latest one refers to the current `MetadataUrl` of the token.

```rust
/// The token state keeps a counter variable (as an index)
/// and a list of MetadataUrls. The counter points to the nth index
/// of the list of MetadataUrls. The idea is to return the value at that index
/// when it's queried by the `TokenMetadata` function.
/// When the owner of the contract upgrades this token, the counter will be
/// incremented by 1 so that the next MetadataUrl from the list becomes active.
#[derive(Serial, Deserial, Clone, SchemaType)]
pub struct TokenMetadataState {
    /// The counter is initially 0. When the owner of the contract
    /// upgrades this token, the counter will be incremented by 1.
    pub token_metadata_current_state_counter: u32,
    /// List of MetadataUrls for different stages of the token
    pub token_metadata_list:                  Vec<MetadataUrl>,
}
impl TokenMetadataState {
    fn add_metadata(&mut self, metadata: MetadataUrl) { self.token_metadata_list.push(metadata); }
}
/// The contract state,
/// Note: The specification does not specify how to structure the contract state
/// and this could be structured in a more space efficient way.
#[derive(Serial, DeserialWithState)]
#[concordium(state_parameter = "S")]
struct State<S> {
    /// The state of addresses.
    state:        StateMap<Address, AddressState<S>, S>,
    /// Token IDs and the MetadataStates which holds the counter and the list of
    /// MetadataURls
    tokens:       StateMap<ContractTokenId, TokenMetadataState, S>,
    /// Map with contract addresses providing implementations of additional
    /// standards.
    implementors: StateMap<StandardIdentifierOwned, Vec<ContractAddress>, S>,
}
```

First, we should modify the `mint` function in the state accordingly to be able to set initial values when its invoked. At the initial state of the token, `token_metadata_current_state_counter` has to be 0 and the `token_metadata_list` will be given by the `MintParams`.

```rust
/// Mints an amount of tokens with a given address as the owner.
    /// Sets the token's metadata list and the current metadata counter to 0.
    fn mint(
        &mut self,
        token_id: &ContractTokenId,
        mint_param: &MintParam,
        owner: &Address,
        state_builder: &mut StateBuilder<S>,
    ) {
        self.tokens.insert(*token_id, TokenMetadataState {
            token_metadata_current_state_counter: 0,
            token_metadata_list:                  mint_param.metadata_url.clone(),
        });
        let mut owner_state =
            self.state.entry(*owner).or_insert_with(|| AddressState::empty(state_builder));
        let mut owner_balance = owner_state.balances.entry(*token_id).or_insert(0.into());
        *owner_balance += mint_param.token_amount;
    }
```

Another addition required to the state is the function that lets the contract owner upgrade the token by incrementing the counter of the `MetadataUrl` index.

```rust
/// Picks the next MetadataUrl from the list when token is upgraded by
    /// the contract owner
    fn upgrade_token(&mut self, token_id: &ContractTokenId) -> ContractResult<()> {
        ensure!(self.contains_token(token_id), ContractError::InvalidTokenId);

        let counter = self.get_token_state_medata_counter(token_id);
        ensure!(
            counter < self.tokens.get(token_id).unwrap().token_metadata_list.len(),
            ContractError::Custom(CustomContractError::UpgradeTokenFailedError)
        );

        self.tokens.get_mut(token_id).unwrap().token_metadata_current_state_counter += 1;
        Ok(())
    }
```

Similarly, we need to create a function to be able to add new metadata.

```rust
 /// Adds a new Metadata URL to the MetdataList at a time
    fn add_metadata(
        &mut self,
        token_id: &ContractTokenId,
        mint_param: &MetadataUrl,
    ) -> ContractResult<()> {
        ensure!(self.contains_token(token_id), ContractError::InvalidTokenId);
        let update = MetadataUrl {
            url:  mint_param.url.clone(),
            hash: mint_param.hash,
        };

        let mut token = self.tokens.get_mut(token_id).unwrap();

        token.add_metadata(update);
        Ok(())
    }
```

And final three functions of the state are the getters of the token, `counter,` and `metadataUrl`.

```rust
    /// Returns the current counter value which is using for indexing the vector
    /// of MetadataUrls
    fn get_token_state_medata_counter(&self, token_id: &ContractTokenId) -> usize {
        let i = self
            .tokens
            .get(token_id)
            .map(|x| x.to_owned().token_metadata_current_state_counter)
            .unwrap();
        i as usize
    }

    /// Gets the current MetadataUrl of specified token in the current state.
    fn get_metadata(&self, token_id: &ContractTokenId) -> ContractResult<MetadataUrl> {
        ensure!(self.contains_token(token_id), ContractError::InvalidTokenId);
        let tk = self.tokens.get(token_id).unwrap();
        Ok(tk.token_metadata_list[tk.token_metadata_current_state_counter as usize].clone())
    }

    /// Gets token of given token id
    fn get_token(&self, token_id: &ContractTokenId) -> Option<MetadataUrl> {
        self.tokens.get(token_id).map(|x| {
            x.to_owned().token_metadata_list[self.get_token_state_medata_counter(token_id)].clone()
        })
    }
```

#### Mint

In this section, we will create the `mint` function and the required parameters. First of all, we need to modify the `MintParams` struct as below, as we want to be able to have a list of `MetadataUrl` while minting. So that the contract owner can mint a d-NFT with more than one metadata.&#x20;

```rust
/// MintParam expects the token amount as TokenAmountU64
/// and a vector of MetadataUrl
#[derive(Serial, Deserial, SchemaType)]
pub struct MintParam {
    /// Number of tokens
    pub token_amount: ContractTokenAmount,
    /// MetadataURLs for a token
    pub metadata_url: Vec<MetadataUrl>,
}

/// The parameter for the contract function `mint` which mints a number of
/// token types and/or amounts of tokens to a given address.
#[derive(Serial, Deserial, SchemaType)]
pub struct MintParams {
    /// Owner of the newly minted tokens.
    pub owner:  Address,
    /// A collection of tokens to mint.
    pub tokens: collections::BTreeMap<ContractTokenId, MintParam>,
}
```

`mint` entrypoint requires a small change from the example which is for the logged event.&#x20;

```rust

/// Mint new tokens with a given address as the owner of these tokens.
/// Can only be called by the contract owner.
/// Logs a `Mint` and a `TokenMetadata` event for each token.
///
/// It rejects if:
/// - The sender is not the contract instance owner.
/// - Fails to parse parameter.
/// - Any of the tokens fails to be minted, which could be if:
///     - Fails to log Mint event.
///     - Fails to log TokenMetadata event.
///
/// Note: Can at most mint 32 token types in one call due to the limit on the
/// number of logs a smart contract can produce on each function call.
#[receive(
    contract = "cis2_dynamic_nft",
    name = "mint",
    parameter = "MintParams",
    error = "ContractError",
    enable_logger,
    mutable
)]
fn contract_mint<S: HasStateApi>(
    ctx: &impl HasReceiveContext,
    host: &mut impl HasHost<State<S>, StateApiType = S>,
    logger: &mut impl HasLogger,
) -> ContractResult<()> {
    // Get the contract owner
    let owner = ctx.owner();
    // Get the sender of the transaction
    let sender = ctx.sender();

    ensure!(sender.matches_account(&owner), ContractError::Unauthorized);

    // Parse the parameter.
    let params: MintParams = ctx.parameter_cursor().get()?;

    let (state, builder) = host.state_and_builder();
    for (token_id, mint_param) in params.tokens {
        // Mint the token in the state.
        state.mint(&token_id, &mint_param, &params.owner, builder);

        // Event for minted token.
        logger.log(&Cis2Event::Mint(MintEvent {
            token_id,
            amount: mint_param.token_amount,
            owner: params.owner,
        }))?;

        // Metadata URL for the token.
        logger.log(&Cis2Event::TokenMetadata::<_, ContractTokenAmount>(TokenMetadataEvent {
            token_id,
            metadata_url: mint_param.metadata_url.first().unwrap().clone(),
        }))?;
    }
    Ok(())
}
```

#### Add

Based on our requirements, we want that the contract owner should be able to add a new `MetadataUrl`. Please be careful, this is different than upgrading it, which is covered in the next step.&#x20;

```rust
/// The parameter for the contract function `addMetadata` which adds a Metadata
/// URL to the list of Metadata URLs
#[derive(Serial, Deserial, SchemaType)]
pub struct AddParams {
    tokens: collections::BTreeMap<ContractTokenId, MetadataUrl>,
}

/// The `addMetadata` function adds new MetadataUrls to the state of different
/// token_ids.
#[receive(
    contract = "cis2_dynamic_nft",
    name = "addMetadata",
    parameter = "AddParams",
    error = "ContractError",
    enable_logger,
    mutable
)]
fn contract_add_metadata<S: HasStateApi>(
    ctx: &impl HasReceiveContext,
    host: &mut impl HasHost<State<S>, StateApiType = S>,
    logger: &mut impl HasLogger,
) -> ContractResult<()> {
    // Get the contract owner
    let owner = ctx.owner();
    // Get the sender of the transaction
    let sender = ctx.sender();

    ensure!(sender.matches_account(&owner), ContractError::Unauthorized);

    // Parse the parameter.
    let params: AddParams = ctx.parameter_cursor().get()?;

    let state = host.state_mut();
    for (token_id, metadata) in params.tokens {
        // Add metadata
        state.add_metadata(&token_id, &metadata)?;

        // Metadata URL for the token.
        logger.log(&Cis2Event::TokenMetadata::<_, ContractTokenAmount>(TokenMetadataEvent {
            token_id,
            metadata_url: metadata,
        }))?;
    }
    Ok(())
}
```

#### Upgrade

As per the requirements, the contract owner can call the `upgrade` function at a time. This entrypoint basically calls the state's `upgrade` function to increment the index.&#x20;

```rust
pub type TokenUpdateParams = ContractTokenId;

/// Only the contract owner can call the `upgrade` function to pint next item in
/// the MetadataUrl list. It requires only tokenId, callable once at a time for
/// a token.

#[receive(
    contract = "cis2_dynamic_nft",
    name = "upgrade",
    parameter = "TokenUpdateParams",
    error = "ContractError",
    enable_logger,
    mutable
)]
fn contract_upgrade<S: HasStateApi>(
    ctx: &impl HasReceiveContext,
    host: &mut impl HasHost<State<S>, StateApiType = S>,
    logger: &mut impl HasLogger,
) -> ContractResult<()> {
    // Get the contract owner
    let owner = ctx.owner();
    // Get the sender of the transaction
    let sender = ctx.sender();

    ensure!(sender.matches_account(&owner), ContractError::Unauthorized);

    // Parse the parameter.

    let token_id: TokenUpdateParams = ctx.parameter_cursor().get()?;

    let state = host.state_mut();

    // Upgrades the given token in the state
    state.upgrade_token(&token_id)?;

    logger.log(&Cis2Event::TokenMetadata::<_, ContractTokenAmount>(TokenMetadataEvent {
        token_id,
        metadata_url: state.get_metadata(&token_id)?,
    }))?;
    Ok(())
}
```



#### TokenMetadata

This function is the one when you are displaying a token or querying its `MetadataUrl` which needs some modifications to return the latest element of the list of `MetadataUrl`s for a given token ID.

```rust
/// Get the token metadata URL and checksums given a list of token IDs.
///
/// It rejects if:
/// - It fails to parse the parameter.
/// - Any of the queried `token_id` does not exist.
#[receive(
    contract = "cis2_dynamic_nft",
    name = "tokenMetadata",
    parameter = "ContractTokenMetadataQueryParams",
    return_value = "TokenMetadataQueryResponse",
    error = "ContractError"
)]
fn contract_token_metadata<S: HasStateApi>(
    ctx: &impl HasReceiveContext,
    host: &impl HasHost<State<S>, StateApiType = S>,
) -> ContractResult<TokenMetadataQueryResponse> {
    // Parse the parameter.
    let params: ContractTokenMetadataQueryParams = ctx.parameter_cursor().get()?;
    // Build the response.
    let mut response = Vec::with_capacity(params.queries.len());
    let state = host.state();

    for token_id in params.queries {
        let token_metadata_url = match state.tokens.get(&token_id) {
            Some(token_metadata_url) => token_metadata_url.token_metadata_list
                [token_metadata_url.token_metadata_current_state_counter as usize]
                .clone(),
            None => bail!(ContractError::InvalidTokenId),
        };
        response.push(token_metadata_url);
    }
    let result = TokenMetadataQueryResponse::from(response);
    Ok(result)
}
```

#### TokenMetadataList

As a final step of the requirements, our contract should return the list of the `MetadataUrl` that has been contained by any token from its initial state. Therefore, we created a helper struct that holds the `MetadataUrl`s of tokens' queried. The `tokenMetadataList` entrypoint iterates the given token IDs and adds the `MetadataUrl`s to the final vector to return.

```rust
#[derive(Serialize, SchemaType)]
struct TokenMetadataList {
    all_tokens_metadatas: Vec<Vec<MetadataUrl>>,
}
/// Get the complete Metadata URLs for given token_ids
/// It rejects if:
/// - It fails to parse the parameter.
/// - Any of the queried `token_id` does not exist.
#[receive(
    contract = "cis2_dynamic_nft",
    name = "tokenMetadataList",
    parameter = "ContractTokenMetadataQueryParams",
    return_value = "TokenMetadataList",
    error = "ContractError"
)]
fn contract_token_metadata_list<S: HasStateApi>(
    ctx: &impl HasReceiveContext,
    host: &impl HasHost<State<S>, StateApiType = S>,
) -> ContractResult<TokenMetadataList> {
    // Parse the parameter.
    let params: ContractTokenMetadataQueryParams = ctx.parameter_cursor().get()?;
    // Build the response.
    let mut response = Vec::with_capacity(params.queries.len());
    let state = host.state();

    for token_id in params.queries {
        let token_metadata_url = match state.tokens.get(&token_id) {
            Some(token_metadata_url) => token_metadata_url.token_metadata_list.clone(),
            None => bail!(ContractError::InvalidTokenId),
        };
        response.push(token_metadata_url);
    }
    Ok(TokenMetadataList {
        all_tokens_metadatas: response,
    })
}
```

