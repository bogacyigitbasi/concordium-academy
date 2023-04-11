# Smart Contract

We will organize the smart contract slightly differently than usual for this tutorial. First, there will be a _lib.rs_ which is basically the main function of our contract like the other programming languages. The compiler starts from this file to compile it. The second file will be the _contract.rs_ which will be our primary _CIS-2_ contract that includes all the logic provided for the requirements. I wanted to keep the _State.rs, Params.rs,_ and _Error.rs_ files separately just for demonstration purposes meaning you can keep them all in your _lib.rs_ file. Finally, we will have a _cis2\_client.rs_ which will enable the master contract to do some operations on the cis-2 token contract. One little reminder, there will be some additions to the cis2-multi contract, you can check the particular contract in this [link](https://github.com/Concordium/concordium-rust-smart-contracts/tree/main/examples/cis2-multi).

**State.rs**

```
/// The contract state,
///
/// Note: The specification does not specify how to structure the contract state
/// and this could be structured in a more space-efficient way.
#[derive(Serial, DeserialWithState, StateClone)]
#[concordium(state_parameter = "S")]
pub struct State<S> {
    /// The state of addresses.
    pub(crate) state: StateMap<Address, AddressState<S>, S>,
    /// All of the token IDs
    pub(crate) tokens: StateMap<ContractTokenId, MetadataUrl, S>,
    /// Map with tokenId and token amount for the supply
    pub(crate) token_supply: StateMap<ContractTokenId, ContractTokenAmount, S>,
    pub(crate) implementors: StateMap<StandardIdentifierOwned, Vec<ContractAddress>, S>,
    pub(crate) collaterals: StateMap<CollateralKey, CollateralState, S>,
}
```

Let’s start with the _state.rs_ file. As you already know from the previous tutorials, the state will keep our assets latest state. The contract has to have an initialization function to create an empty state in the beginning which includes four maps such as _state, tokens, token\_supply, implementors, and collaterals_. We have one addition which is the _collaterals_ practically, the tokens to be locked will be stored in this _StateMap\<CollateralKey, CollateralState, S>._

We need to create a new state variable for our collateralized token aka the tokens to be locked. We will need to keep the token’s contract address, its id and who locked it which all are provided in the _CollateralKey_ struct below. Then we need the token amounts for the total number of fractions and the new token ids.&#x20;

Basically, when someone sends the fraction token to this contract, we will assume that he/she wants to burn that asset and we will burn it and the _new()_ function will be invoked when someone wants to add new collateral to mint its fractions.



```
#[derive(Serial, Deserial, Clone, SchemaType, Copy)]
pub struct CollateralKey {
    pub contract: ContractAddress,
    pub token_id: ContractTokenId,
    pub owner: AccountAddress,
}

#[derive(Serial, Deserial, Clone, Copy, SchemaType)]
pub struct CollateralState {
    pub received_token_amount: ContractTokenAmount,
    pub minted_token_id: Option<ContractTokenId>,
}

impl CollateralState {
    fn new() -> Self {
        CollateralState {
            received_token_amount: ContractTokenAmount::from(0),
            minted_token_id: Option::None,
        }
    }
}
```

There are 5 new additions to the _cis2-multi_ contract state functions for handling the collateral state. The first one is the _add\_collateral()_ function, it expects the _token contract address, token\_id, owner address,_ and the _token amount_ to be locked.&#x20;

The second one is the _has\_collateral(), which_ similarly takes the _token contract address, token\_id,_ and _owner address_ as input to create a key in the form of _CollateralKey_ struct to look into the _StateMap_. If someone has already collateralized the token, this will return _true_ and we will use this to make sure that he will not be able to do it again.

The third one is _find\_collateral(),_ which takes _token\_id (fraction)_ as a parameter and checks its existence in the minted tokens. If there exists a token with that id, returns a clone of it.

The fourth one is _has\_fractions(),_ we will use this one to check whether a token is already fractionalized into new ones. We don't want to allow people to create more and more fractions when they lock their assets once.&#x20;

The last one is _update\_collateral\_token()_ we will use this one when we have locked the tokens while minting new fractions. Based on our amount of fractions, it will update the state with the new tokens.

One important note here you can lock a semi-fungible token technically with this example. If you would like to limit it you can adjust it by checking the amount simply.&#x20;



```
    pub(crate) fn add_collateral(
        &mut self,
        contract: ContractAddress,
        token_id: ContractTokenId,
        owner: AccountAddress,
        received_token_amount: ContractTokenAmount,
    ) {
        let key = CollateralKey {
            contract,
            token_id,
            owner,
        };

        let mut cs = match self.collaterals.get(&key) {
            Some(v) => *v,
            None => CollateralState::new(),
        };

        cs.received_token_amount += received_token_amount;

        self.collaterals.insert(key, cs);
    }

    pub(crate) fn has_collateral(
        &self,
        contract: &ContractAddress,
        token_id: &ContractTokenId,
        owner: &AccountAddress,
    ) -> bool {
        let key = CollateralKey {
            contract: *contract,
            token_id: *token_id,
            owner: *owner,
        };

        self.collaterals.get(&key).is_some()
    }

    pub(crate) fn find_collateral(
        &self,
        token_id: &ContractTokenId,
    ) -> Option<(CollateralKey, ContractTokenAmount)> {
        for c in self.collaterals.iter() {
            match c.1.minted_token_id {
                Some(t) => {
                    if t.eq(token_id) {
                        return Some((c.0.clone(), c.1.received_token_amount));
                    }
                }
                None => continue,
            };
        }

        None
    }

    pub(crate) fn has_fraction(
        &self,
        contract: &ContractAddress,
        token_id: &ContractTokenId,
        owner: &AccountAddress,
    ) -> Option<ContractTokenId> {
        let key = CollateralKey {
            contract: *contract,
            token_id: *token_id,
            owner: *owner,
        };

        self.collaterals.get(&key)?.minted_token_id
    }

    pub(crate) fn update_collateral_token(
        &mut self,
        contract: ContractAddress,
        token_id: ContractTokenId,
        owner: AccountAddress,
        minted_token_id: ContractTokenId,
    ) -> ContractResult<()> {
        let key = CollateralKey {
            contract,
            token_id,
            owner,
        };

        match self.collaterals.entry(key) {
            Entry::Vacant(_) => bail!(Cis2Error::Custom(CustomContractError::InvalidCollateral)),
            Entry::Occupied(mut e) => {
                e.modify(|s| s.minted_token_id = Some(minted_token_id));
                Ok(())
            }
        }
    }
```

**Token Supply Helpers**

```
  fn increase_supply(&mut self, token_id: ContractTokenId, amount: ContractTokenAmount) {
        let curr_supply = self.get_supply(&token_id);
        self.token_supply.insert(token_id, curr_supply + amount);
    }
  fn decrease_supply(&mut self, token_id: ContractTokenId, amount: ContractTokenAmount) {
        let curr_supply = self.get_supply(&token_id);
        let remaining_supply = curr_supply - amount;
        if remaining_supply.cmp(&ContractTokenAmount::from(0)).is_eq() {
            self.token_supply.remove(&token_id);
        } else {
            self.token_supply.insert(token_id, curr_supply - amount);
        }
    }
   pub(crate) fn get_supply(&self, token_id: &ContractTokenId) -> ContractTokenAmount {
        match self.token_supply.get(token_id) {
            Some(amount) => *amount,
            None => ContractTokenAmount::from(0),
        }
    }
```

**State Mint Function**

There is only one addition to the existing _mint()_ function in cis2-multi contract which is the _increase\_supply()_ when a token is minted.

```
   /// Mints an amount of tokens with a given address as the owner.
    pub(crate) fn mint(
        &mut self,
        token_id: &ContractTokenId,
        token_metadata: &TokenMetadata,
        amount: ContractTokenAmount,
        owner: &Address,
        state_builder: &mut StateBuilder<S>,
    ) {
        {
            self.tokens
                .insert(*token_id, token_metadata.to_metadata_url());
            let mut owner_state = self
                .state
                .entry(*owner)
                .or_insert_with(|| AddressState::empty(state_builder));
            let mut owner_balance = owner_state.balances.entry(*token_id).or_insert(0.into());
            *owner_balance += amount;
        }

        self.increase_supply(*token_id, amount);
    }
```

**State Burn Function**

Then we need to add a _burn()_ function to the state so that we will be able to burn the fractions which you can find in the following. We will use the _decrease\_supply()_ function to update the state when we burn something.

```
    pub(crate) fn burn(
        &mut self,
        token_id: &ContractTokenId,
        amount: ContractTokenAmount,
        owner: &Address,
    ) -> ContractResult<ContractTokenAmount> {
        let ret = {
            match self.state.get_mut(owner) {
                Some(address_state) => match address_state.balances.get_mut(token_id) {
                    Some(mut b) => {
                        ensure!(
                            b.cmp(&amount).is_ge(),
                            Cis2Error::Custom(CustomContractError::NoBalanceToBurn)
                        );

                        *b -= amount;
                        Ok(*b)
                    }
                    None => Err(Cis2Error::Custom(CustomContractError::NoBalanceToBurn)),
                },
                None => Err(Cis2Error::Custom(CustomContractError::NoBalanceToBurn)),
            }
        };

        self.decrease_supply(*token_id, amount);

        ret
    }
```







**Params.rs**

In this file, we will keep our parameter structs and some implementation for them in order to mint, metadata operations, and view. They are almost the same with _cis2-multi_ parameters with some additions for collaterals.

```
use concordium_cis2::*;
use concordium_std::*;
use core::convert::TryInto;

use crate::{
    state::{CollateralKey, CollateralState},
    ContractTokenAmount, ContractTokenId,
};

#[derive(Serial, Deserial, SchemaType)]
pub struct TokenMintParams {
    pub metadata: TokenMetadata,
    pub amount: ContractTokenAmount,
    pub contract: ContractAddress,
    pub token_id: ContractTokenId,
}

/// The parameter for the contract function `mint` which mints a number of
/// token types and/or amounts of tokens to a given address.
#[derive(Serial, Deserial, SchemaType)]
pub struct MintParams {
    /// Owner of the newly minted tokens.
    pub owner: Address,
    /// A collection of tokens to mint.
    pub tokens: collections::BTreeMap<ContractTokenId, TokenMintParams>,
}

/// The parameter type for the contract function `setImplementors`.
/// Takes a standard identifier and a list of contract addresses providing
/// implementations of this standard.
#[derive(Debug, Serialize, SchemaType)]
pub struct SetImplementorsParams {
    /// The identifier for the standard.
    pub id: StandardIdentifierOwned,
    /// The addresses of the implementors of the standard.
    pub implementors: Vec<ContractAddress>,
}

#[derive(Debug, Serialize, Clone, SchemaType)]
pub struct TokenMetadata {
    /// The URL following the specification RFC1738.
    #[concordium(size_length = 2)]
    pub url: String,
    /// A optional hash of the content.
    #[concordium(size_length = 2)]
    pub hash: String,
}

impl TokenMetadata {
    fn get_hash_bytes(&self) -> Option<[u8; 32]> {
        match hex::decode(self.hash.to_owned()) {
            Ok(v) => {
                let slice = v.as_slice();
                match slice.try_into() {
                    Ok(array) => Option::Some(array),
                    Err(_) => Option::None,
                }
            }
            Err(_) => Option::None,
        }
    }

    pub(crate) fn to_metadata_url(&self) -> MetadataUrl {
        MetadataUrl {
            url: self.url.to_string(),
            hash: self.get_hash_bytes(),
        }
    }
}

#[derive(Serialize, SchemaType)]
pub struct ViewAddressState {
    pub balances: Vec<(ContractTokenId, ContractTokenAmount)>,
    pub operators: Vec<Address>,
}

#[derive(Serialize, SchemaType)]
pub struct ViewState {
    pub state: Vec<(Address, ViewAddressState)>,
    pub tokens: Vec<ContractTokenId>,
    pub collaterals: Vec<(CollateralKey, CollateralState)>,
}

/// Parameter type for the CIS-2 function `balanceOf` specialized to the subset
/// of TokenIDs used by this contract.
pub type ContractBalanceOfQueryParams = BalanceOfQueryParams<ContractTokenId>;

/// Response type for the CIS-2 function `balanceOf` specialized to the subset
/// of TokenAmounts used by this contract.
pub type ContractBalanceOfQueryResponse = BalanceOfQueryResponse<ContractTokenAmount>;

pub type TransferParameter = TransferParams<ContractTokenId, ContractTokenAmount>;
```

\


**Error.rs**

We will implement custom errors for this project like the one below see the last 6 errors. For more information about custom errors in Concordium smart contracts check [this](https://developer.concordium.software/en/mainnet/smart-contracts/guides/custom-errors.html) link.&#x20;

```
pub enum CustomContractError {
    /// Failed parsing the parameter.
    #[from(ParseError)]
    ParseParams,
    /// Failed logging: Log is full.
    LogFull,
    /// Failed logging: Log is malformed.
    LogMalformed,
    /// Invalid contract name.
    InvalidContractName,
    /// Only a smart contract can call this function.
    ContractOnly,
    /// Failed to invoke a contract.
    InvokeContractError,
    TokenAlreadyMinted,
    InvalidCollateral,
    AlreadyCollateralized,
    NoBalanceToBurn,
    AccountsOnly,
    Cis2ClientError(Cis2ClientError),
}
```

**cis2\_client.rs**

In order to call a contract from another smart contract we need to implement a relay layer which is our _cis2\_client.rs_ it implements the transfer function. We will transfer the asset back to the original owner when all fractions are burned. In order to do that, we need to implement this client that will allow us to call the _transfer()_ function on the NFT contract. Please keep this in mind, you should transfer it using the contract that minted the original token in the first place.&#x20;

```
//! CIS2 client is the intermediatory layer between fractionalizer contract and CIS2 contract.
//!
//! # Description
//! It allows Fractionalizer contract to abstract away logic of calling CIS2 contract for the following methods
//! - `transfer` : Calls [`transfer`](https://proposals.concordium.software/CIS/cis-2.html#transfer)

use std::vec;

use concordium_cis2::*;
use concordium_std::*;

use crate::state::State;

pub const TRANSFER_ENTRYPOINT_NAME: &str = "transfer";

#[derive(Serialize, Debug, PartialEq, Eq, Reject, SchemaType)]
pub enum Cis2ClientError {
    InvokeContractError,
    ParseParams,
}

pub struct Cis2Client;

impl Cis2Client {
    pub(crate) fn transfer<
        S,
        T: IsTokenId + Clone + Copy,
        A: IsTokenAmount + Clone + Copy + ops::Sub<Output = A>,
    >(
        host: &mut impl HasHost<State<S>, StateApiType = S>,
        token_id: T,
        nft_contract_address: ContractAddress,
        amount: A,
        from: Address,
        to: Receiver,
    ) -> Result<(), Cis2ClientError>
    where
        S: HasStateApi,
        A: IsTokenAmount,
    {
        let params = TransferParams(vec![Transfer {
            token_id,
            amount,
            from,
            data: AdditionalData::empty(),
            to,
        }]);

        Cis2Client::invoke_contract_read_only(
            host,
            &nft_contract_address,
            TRANSFER_ENTRYPOINT_NAME,
            &params,
        )?;

        Ok(())
    }

    fn invoke_contract_read_only<S: HasStateApi, R: Deserial, P: Serial>(
        host: &mut impl HasHost<State<S>, StateApiType = S>,
        contract_address: &ContractAddress,
        entrypoint_name: &str,
        params: &P,
    ) -> Result<R, Cis2ClientError> {
        let invoke_contract_result = host
            .invoke_contract_read_only(
                contract_address,
                params,
                EntrypointName::new(entrypoint_name).unwrap_abort(),
                Amount::from_ccd(0),
            )
            .map_err(|_e| Cis2ClientError::InvokeContractError)?;
        let mut invoke_contract_res = match invoke_contract_result {
            Some(s) => s,
            None => return Result::Err(Cis2ClientError::InvokeContractError),
        };
        let parsed_res =
            R::deserial(&mut invoke_contract_res).map_err(|_e| Cis2ClientError::ParseParams)?;

        Ok(parsed_res)
    }
}
```

**Contract.rs**

Finally, we are going to discuss the modifications for the fractionalization of NFTs in the contract functions. There are two major changes in the _contract\_mint()_ and _contract\_transfer()_ functions which we are going to explain. The full code will be shared at the end of the tutorial.

**Mint Function**

In the _contract\_mint()_ function, there are 3 additions.&#x20;

First, we want to make sure that only accounts can lock and fractionalize the NFTs. You can see the _match_ statement below that does particular control.

Second, it should be impossible to mint new fractions if the collateral is not locked first. So, we need to ensure that the token exists in our collateral list. In the _ensure!()_ statement we check this, and if it violates throw an _InvalidCollateral_ custom error.

As a final addition to the _mint()_ function, we need to update our state when a token is minted. Basically, we are storing which token from which contract is locked, which token on this contract is minted and who is the owner. See the usage of the _update\_collateral\_token()_ function below.

```
#[receive(
    contract = "CIS2-Fractionalizer",
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
    let sender = match ctx.sender() {
        Address::Account(a) => a,
        Address::Contract(_) => bail!(CustomContractError::AccountsOnly.into()),
    };

    // Parse the parameter.
    let params: MintParams = ctx.parameter_cursor().get()?;

    let (state, builder) = host.state_and_builder();
    for (token_id, token_info) in params.tokens {
       ensure!(
            state.contains_token(&token_id),
            ContractError::Custom(CustomContractError::TokenAlreadyMinted)
        );    
    
        ensure!(
            state.has_collateral(&token_info.contract, &token_info.token_id, &sender),
            concordium_cis2::Cis2Error::Custom(CustomContractError::InvalidCollateral)
        );
        // create a fraction only for once for a token
        ensure!(
            state
                .has_fraction(&token_info.contract, &token_info.token_id, &sender)
                .is_none(),
            concordium_cis2::Cis2Error::Custom(CustomContractError::AlreadyCollateralized)
        );

        // Mint the token in the state.
        state.mint(
            &token_id,
            &token_info.metadata,
            token_info.amount,
            &params.owner,
            builder,
        );

        state.update_collateral_token(
            token_info.contract,
            token_info.token_id,
            sender,
            token_id,
        )?;

        // Event for minted token.
        logger.log(&Cis2Event::Mint(MintEvent {
            token_id,
            amount: token_info.amount,
            owner: params.owner,
        }))?;

        // Metadata URL for the token.
        logger.log(&Cis2Event::TokenMetadata::<_, ContractTokenAmount>(
            TokenMetadataEvent {
                token_id,
                metadata_url: token_info.metadata.to_metadata_url(),
            },
        ))?;
    }
    Ok(())
}
```

**Transfer Function**

We are about to finalize our contract development after one final step which is the _contract\_transfer()_ function. Basically, when you want to send your tokens to another address, you will invoke this function. In addition to that, we wanted to combine the burning process into this function.&#x20;

According to this logic, when you transfer the fractions (tokens minted on this contract) back to the contract, we assume you want to burn them. You need to be the owner of the asset when calling it. After we ensure whether you are authorized -meaning have some tokens-, then we check that you want to send those tokens to the contract itself. The next step is calling the state’s _burn()_ function which will reduce the token amount from either your balance or the state followed by emitting a _BurnEvent._ Note that, when you call the _burn()_ function, you need to emit the _BurnEvent._ For more detail check CIS-2 standard documentation from this [link](https://proposals.concordium.software/CIS/cis-2.html#cis-2-concordium-token-standard-2).

The state’s _burn()_ function will return the _remaining\_amount_, if this amount is 0 then we can say that this should be unlocked now as there is no need for the collateral. At this point, we will need a client in order to communicate with this CIS-2 token -the one that was locked as collateral in the beginning- smart contract to invoke the transfer function. Basically, our contract will be transferring the asset back to the owner by getting his/her address from the state’s _CollateralKey_ struct using the _token\_id_.

In the else statement, we are just transferring a token to someone else so this part is identical to the cis2-multi contract’s _transfer()_ function.&#x20;

```
#[receive(
    contract = "CIS2-Fractionalizer",
    name = "transfer",
    parameter = "TransferParameter",
    error = "ContractError",
    enable_logger,
    mutable
)]
fn contract_transfer<S: HasStateApi>(
    ctx: &impl HasReceiveContext,
    host: &mut impl HasHost<State<S>, StateApiType = S>,
    logger: &mut impl HasLogger,
) -> ContractResult<()> {
    // Parse the parameter.
    let TransferParams(transfers): TransferParameter = ctx.parameter_cursor().get()?;
    // Get the sender who invoked this contract function.
    let sender = ctx.sender();

    for Transfer {
        token_id,
        amount,
        from,
        to,
        data,
    } in transfers
    {
        let (state, builder) = host.state_and_builder();
        // Authenticate the sender for this transfer
        ensure!(
            from == sender || state.is_operator(&sender, &from),
            ContractError::Unauthorized
        );

        if to.address().matches_contract(&ctx.self_address()) {
            // tokens are being transferred to self
            // burn the tokens
            let remaining_amount: ContractTokenAmount = state.burn(&token_id, amount, &from)?;

            // log burn event
            logger.log(&Cis2Event::Burn(BurnEvent {
                token_id,
                amount,
                owner: from,
            }))?;

            // Check of there is any remaining amount
            if remaining_amount.eq(&ContractTokenAmount::from(0)) {
                // Everything has been burned
                // Transfer collateral back to the original owner
                let (collateral_key, collateral_amount) = state
                    .find_collateral(&token_id)
                    .ok_or(Cis2Error::Custom(CustomContractError::InvalidCollateral))?;

                // Return back the collateral
                Cis2Client::transfer(
                    host,
                    collateral_key.token_id,
                    collateral_key.contract,
                    collateral_amount,
                    concordium_std::Address::Contract(ctx.self_address()),
                    concordium_cis2::Receiver::Account(collateral_key.owner),
                )
                .map_err(CustomContractError::Cis2ClientError)?;
            }
        } else {
            let to_address = to.address();

            // Tokens are being transferred to another address
            // Update the contract state
            state.transfer(&token_id, amount, &from, &to_address, builder)?;

            // Log transfer event
            logger.log(&Cis2Event::Transfer(TransferEvent {
                token_id,
                amount,
                from,
                to: to_address,
            }))?;

            // If the receiver is a contract we invoke it.
            if let Receiver::Contract(address, entrypoint_name) = to {
                let parameter = OnReceivingCis2Params {
                    token_id,
                    amount,
                    from,
                    data,
                };
                host.invoke_contract(
                    &address,
                    &parameter,
                    entrypoint_name.as_entrypoint_name(),
                    Amount::zero(),
                )?;
            }
        }
    }

    Ok(())
}
```



