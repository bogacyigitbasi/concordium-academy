# The Verifier Backend

In this tutorial, we will show you how to create a minting platform that allows minting for people older than 18. In order to do that, the platform should ask for proof that the minter meets our requirement which is being older than 18 years old. But first, let’s shed a bit of light on what technology we will use, its terminology and what is the beauty of Concordium.

We decided to create two parts as the first part includes a lot of cryptographic implementation and explanation. In the scope of this part, we will implement a verifier backend server that signs a claim if it is verified.

**ZKP and ID Proofs**

Concordium Web wallet provides this feature and serves to dApps about if a particular user meets certain criteria(s) like age, gender, nationality, and even name. You can find more info in this link. But basically, when you have a Concordium account you have an ID object and the identity credential in your wallet. On-chain (in your account) there exists a (list of) commitment(s) to your attributes. No one can know who you are other than being able to see your public address. But with the brilliant _Zero-Knowledge Proof_ technology- proving a claim without revealing any information but the claim itself-, any dApp that wants to make sure that its users meet some criteria, the answer to the query which uses a ZK proof to show correctness. We call these criteria _statements._ These statements can be located in the dApp itself or stored in _the verifier_ which is a backend HTTP server. For this particular tutorial’s scenario it is being older than 18 is the statement that dApps asking for.

When the dApps wants to prove someone meets the criteria, it should first go to _the verifier._ The verifier is one of the key elements of this architecture, basically, dApp uses to verify the claims but it has a key function above all else. The verifier makes sure that a ZKP query cant be reused by someone else for example if it's stolen somehow. When the dApp communicates with the verifier, it asks for a _challenge_ simply a one-time or time-bound random string, that will be used while creating the proofs. The verifier, when doing verification checks whether the proof is created with the particular challenge issued for the query. If the proof is not created with the particular challenge, will not be verified.

When you have your Concordium account, meaning your ID object is created there are a set of _attributes_ that is inside of your encrypted data structure. A full list of the attributes can be found in this link but some of them are listed below.

* First name, Last name
* Sex
* Date of birth
* Country of nationality
* …

Using the Concordium Web Wallet, our dApp can request proofs for any of these attributes from its users. There is no possible way for us to _know_ anything beyond that _the statement_ doesn't include in the first place. When the user agrees to reveal these pieces of information, -not his all identity- he/she will start experiencing true Self-Sovereign Identity. That’s the beauty and its the future of Web 3. We will give the control back to the users!

**Minting with ID 2.0**

Let’s get started with technical implementations. It’s always good to define the requirements and the steps that will lead you to implement the solution.

* We want to use the existing NFT minting tool React application for the sake of time and implementation.
* We will implement a verifier backend and re-use as much as possible that is shared by the Concordium team.
* Our minting dApp will allow people only older than 18, but we can increase the set of attributes or add new combinations.

Nice, we have a very short requirements list. Now take a look at the flow from the architectural point of view in general.

* When the user wants to mint something, dApp goes to the verifier backend and asks for a challenge alongside the statement(s).
* The dApp sends a request for proof for the given challenge and statement to the wallet.
* The user accepts the requests, wallet sends back the proof.
* The dApp sends it to the verifier, it verifies the proof is correct according to the challenge and statement.
* It uses the private key of the owner, to sign a message.
* The smart contract’s _mint()_ function checks the signature created by the owner and allows for mint.

**Verifier Backend**

We will use the backend code shared in Concordium’s GitHub repo in this [link](https://github.com/Concordium/concordium-dapp-examples/tree/main/gallery/verifier). There will be some modifications based on our needs.

First, let’s create an empty project called the _backend_.

```
cargo new backend
```

<figure><img src="https://miro.medium.com/v2/resize:fit:1400/1*SUQ2I-lh3a48eYX6HBrAyw.png" alt=""><figcaption></figcaption></figure>

**Concordium Rust-SDK**

We will use a lot of features that Concordium Rust-SDK provides and we need to install it. Since it’s not published on crates.io yet, you have to clone your code and build it.

Create a folder called _deps_ and then clone the repo inside of it. Run the command below to clone the code and then you will update the submodules with the next one.

```
git clone https://github.com/Concordium/concordium-rust-sdk
git submodule update --init --recursive
```

<figure><img src="https://miro.medium.com/v2/resize:fit:1400/1*2qloaMMMO1QURB4I2unYZQ.png" alt=""><figcaption></figcaption></figure>

For the development, serialization, encryption, and an HTTP running server, we will need some dependencies to add the dependencies below to your _cargo.toml_ file. Let’s go over some of these dependencies.

**tokio**: A runtime for writing reliable, asynchronous, and slim applications with the Rust programming language.

**warp**: A super-easy, composable, web server framework for warp speeds.

**serde**: A very helpful framework for serializing/deserializing data structures generically.

**serde\_json**: A JSON serialization/deserialization file format.

**clap**: Command Line Argument Parser for Rust (CLAP)

**anyhow**: Easy error handling trait.

**ed25519-dalek:** To produce and consume Ed25519 signatures and for other key operations.

```
[dependencies]
tokio = { version = "1", features = ["full"] }
warp = "0.3"
serde = { version = "1.0", features = ["derive"] }
serde_json = "1.0"
log = "0.4.11"
env_logger = "0.9"
clap = { version = "4", features = ["derive"] }
anyhow = "1.0"
chrono = "0.4.19"
thiserror = "1"
rand = "0.8"
ed25519-dalek = "1.0.1"
hex = "0.4.3"

[dependencies.concordium-rust-sdk]
path = "../deps/concordium-rust-sdk/"
```

And now we can build it with the “_cargo build”_ command in the _concordium-rust-sdk_ folder_._ For Mac users, if you face a protobuf error in this step, you might need to install it manually. Then build it again.

```
brew install protobuf
```

<figure><img src="https://miro.medium.com/v2/resize:fit:1400/1*8zgfFsDaskUBNtqbHOSkZw.png" alt=""><figcaption></figcaption></figure>

Let’s start with the implementation and create a _types.rs_ file. We will use almost the same code in this [link](https://github.com/Concordium/concordium-dapp-examples/blob/main/gallery/verifier/src/types.rs). In this file, we will create our data structures, and responses, and manipulate error codes.

First, we have the _Challenge_ struct which is a u8 32 bytes array. This will be re-generated every time a new client connects or the backend gets a request.

_WithAccountAddress_ is used for storing the challenge created for a particular account.

_ChallengeStatus_ is used for storing the issued challenge, to whom its issued(address), and its creation time on the state.

The _Server_ is our state. When we run our verifier backend, we will create an empty state with an empty hashmap of challenges.

_The InjectStatementError_ enum will be used for handling rejections with error codes.

```
use concordium_rust_sdk::{
    common::{
        self as crypto_common,
        derive::{SerdeBase16Serialize, Serialize},
        Buffer, Deserial, ParseResult, ReadBytesExt, SerdeDeserialize, SerdeSerialize, Serial,
        Versioned,
    },
    endpoints::{QueryError, RPCError},
    id::{
        constants::{ArCurve, AttributeKind},
        id_proof_types::Proof,
        types::{AccountAddress, GlobalContext},
    },
    types::CredentialRegistrationID,
};
use std::{
    collections::HashMap,
    sync::{Arc, Mutex},
    time::SystemTime,
};

#[derive(
    Copy, Clone, Eq, PartialEq, Ord, PartialOrd, Hash, Debug, SerdeBase16Serialize, Serialize,
)]
pub struct Challenge(pub [u8; 32]);

#[derive(serde::Deserialize, Debug, Clone)]
pub struct WithAccountAddress {
    pub address: AccountAddress,
}

#[derive(Clone)]
pub struct ChallengeStatus {
    pub address: AccountAddress,
    pub created_at: SystemTime,
}

#[derive(Clone)]
pub struct Server {
    pub challenges: Arc<Mutex<HashMap<String, ChallengeStatus>>>,
    pub global_context: Arc<GlobalContext<ArCurve>>,
}

#[derive(Debug)]
/// An internal error type used by this server to manage error handling.
#[derive(thiserror::Error)]
pub enum InjectStatementError {
    #[error("Not allowed")]
    NotAllowed,
    #[error("Invalid proof")]
    InvalidProofs,
    #[error("Node access error: {0}")]
    NodeAccess(#[from] QueryError),
    #[error("Error acquiring internal lock.")]
    LockingError,
    #[error("Proof provided for an unknown session.")]
    UnknownSession,
    #[error("Issue with credential.")]
    Credential,
}

impl warp::reject::Reject for InjectStatementError {}

#[derive(serde::Serialize)]
/// Response in case of an error. This is going to be encoded as a JSON body
/// with fields 'code' and 'message'.
pub struct ErrorResponse {
    pub code: u16,
    pub message: String,
}

#[derive(serde::Deserialize, serde::Serialize, Debug)]
pub struct ChallengeResponse {
    pub challenge: Challenge,
}

#[derive(serde::Deserialize, serde::Serialize, Debug, Clone)]
pub struct ChallengedProof {
    pub challenge: Challenge,
    pub proof: ProofWithContext,
}

#[derive(serde::Deserialize, serde::Serialize, Debug, Clone)]
pub struct ProofWithContext {
    pub credential: CredentialRegistrationID,
    pub proof: Versioned<Proof<ArCurve, AttributeKind>>,
}
```

Finally, we have the _ChallengeResponse_ struct which will be used the API responses, _ChallengedProof_ and _ProofWithContext._ When the backend receives the proof, it will use these to validate it using the client object.

**Handlers.rs**

Nice, let’s create another file called _handlers.rs._ In this module, we will explain all functions one by one and add them to the file.

The _handle\_get\_challenge()_ function gets input as the _state_ and an _address._ It runs asynchronously when someone asks for a challenge using an endpoint. It invokes the _get\_challenge\_worker()_ which generates a random 32 bytes message aka _Challenge_ adds it to the _state’_s _challenges_ after encoding it the challenge as a key and the _address_ + issuing _time_ as value_(ChallengeStatus)._ As a result, it returns back the challenge as a response to send it back through the endpoint.

```
use crate::crypto_common::base16_encode_string;
use crate::types::*;
use concordium_rust_sdk::{
    common::{self as crypto_common, types::KeyPair},
    id::{
        constants::{ArCurve, AttributeKind},
        id_proof_types::Statement,
        types::{AccountAddress, AccountCredentialWithoutProofs},
    },
    v2::BlockIdentifier,
};
use log::warn;
use rand::Rng;
use std::convert::Infallible;
use std::time::SystemTime;
use warp::{http::StatusCode, Rejection};

static CHALLENGE_EXPIRY_SECONDS: u64 = 600;
static CLEAN_INTERVAL_SECONDS: u64 = 600;

pub async fn handle_get_challenge(
    state: Server,
    address: AccountAddress,
) -> Result<impl warp::Reply, Rejection> {
    let state = state.clone();
    log::debug!("Parsed statement. Generating challenge");
    match get_challenge_worker(state, address).await {
        Ok(r) => Ok(warp::reply::json(&r)),
        Err(e) => {
            warn!("Request is invalid {:#?}.", e);
            Err(warp::reject::custom(e))
        }
    }
}

/// A common function that produces a challenge and adds it to the state.
async fn get_challenge_worker(
    state: Server,
    address: AccountAddress,
) -> Result<ChallengeResponse, InjectStatementError> {
    let mut challenge = [0u8; 32];
    rand::thread_rng().fill(&mut challenge[..]);
    let mut sm = state
        .challenges
        .lock()
        .map_err(|_| InjectStatementError::LockingError)?;
    log::debug!("Generated challenge: {:?}", challenge);
    let challenge = Challenge(challenge);

    sm.insert(
        base16_encode_string(&challenge.0),
        ChallengeStatus {
            address,
            created_at: SystemTime::now(),
        },
    );
    Ok(ChallengeResponse { challenge })
}
```

The _handle\_provide\_proof()_ function, gets the _client_, _state_, _statement_, request, and _key\_pair_ as input. It serves through an API endpoint and is primarily used for verifying the proof by calling the _check\_proof\_worker()_ function.

The _check\_proof\_worker()_ function, validates the cryptographic proof. First, it locks the _state_ and gets the _status_ from the map using the _challenge’s_ base\_16 encoded _key_ of the map. Since the request is a _ChallengedProof_ type, we can access the _challenge_ and more than that, it holds the _ProofWithContext_ struct meaning both the credential and the proof is available to use in verification. Similarly, the _status_ is a _ChallengeStatus_ meaning we know the _address_ issued and the _time_ created. We will need these in the next step. And finally, the _Statement_ is a struct that holds a list of atomic statements.

When this function is invoked with a POST request, we will have the request object and we will use it to extract the _credential\_id._ Please note that every account has an account registration ID, which is the Credential ID of the first credential added to the account. Create a variable for the _cred\_id_ and let’s get the account information using the Concordium Rust-SDK. The function takes mutable _client: concordium\_rust\_sdk::v2::Clien_t as a parameter. We will get the account information using that. Use the _account address_ from the _status_, as the first parameter of the _client.get\_account\_info()_ and the _BlockIdentifier::LastFinal_ as the second param. Basically, this function provides you with the required information for a given account address in the given block. So we give it the _LastFinal_ block means the last finalized block at the time of the query.

Then we will find the credential by simply getting the initial element of the _account\_credentials_ map which is the map of all currently active credentials on the account. This includes public keys that can sign for the given credentials, as well as any revealed attributes. A credential contains commitments to these attributes. The map holds the _AccountCredentialWithoutProofs_ which has the _InitialCredentialDeploymentValues_ and _CredentialDeploymentCommitments._

```

pub async fn handle_provide_proof(
    client: concordium_rust_sdk::v2::Client,
    state: Server,
    statement: Statement<ArCurve, AttributeKind>,
    request: ChallengedProof,
    key_pair: KeyPair,
) -> Result<impl warp::Reply, Rejection> {
    let client = client.clone();
    let state = state.clone();
    let statement = statement.clone();
    match check_proof_worker(client, state, request, statement, key_pair).await {
        Ok(r) => Ok(warp::reply::json(&r)),
        Err(e) => {
            warn!("Request is invalid {:#?}.", e);
            Err(warp::reject::custom(e))
        }
    }
}

/// A common function that validates the cryptographic proofs in the request.
async fn check_proof_worker(
    mut client: concordium_rust_sdk::v2::Client,
    state: Server,
    request: ChallengedProof,
    statement: Statement<ArCurve, AttributeKind>,
    key_pair: KeyPair,
) -> Result<String, InjectStatementError> {
    let status = {
        let challenges = state
            .challenges
            .lock()
            .map_err(|_| InjectStatementError::LockingError)?;

        challenges
            .get(&base16_encode_string(&request.challenge.0))
            .ok_or(InjectStatementError::UnknownSession)?
            .clone()
    };

    let cred_id = request.proof.credential;
    let acc_info = client
        .get_account_info(&status.address.into(), BlockIdentifier::LastFinal)
        .await?;

    // TODO Check remaining credentials
    let credential = acc_info
        .response
        .account_credentials
        .get(&0.into())
        .ok_or(InjectStatementError::Credential)?;

    if crypto_common::to_bytes(credential.value.cred_id()) != crypto_common::to_bytes(&cred_id) {
        return Err(InjectStatementError::Credential);
    }

    let commitments = match &credential.value {
        AccountCredentialWithoutProofs::Initial { icdv: _, .. } => {
            return Err(InjectStatementError::NotAllowed);
        }
        AccountCredentialWithoutProofs::Normal { commitments, .. } => commitments,
    };

    let mut challenges = state
        .challenges
        .lock()
        .map_err(|_| InjectStatementError::LockingError)?;

    if statement.verify(
        &request.challenge.0,
        &state.global_context,
        cred_id.as_ref(),
        commitments,
        &request.proof.proof.value, // TODO: Check version.
    ) {
        challenges.remove(&base16_encode_string(&request.challenge.0));
        let sig = key_pair.sign(&acc_info.response.account_address.0);
        Ok(hex::encode_upper(sig.sig))
    } else {
        Err(InjectStatementError::InvalidProofs)
    }
}
```

The line below from the code snippet makes sure that the credential sent by the user is the same, as the one that the account has.

```
if crypto_common::to_bytes(credential.value.cred_id()) != crypto_common::to_bytes(&cred_id) {
        return Err(InjectStatementError::Credential);
    }
```

Then we will get the commitments, which are simply the protectors of the attribute credentials in a way. They are the attributes that the user doesn't want to reveal on the account while creating their wallet. So a user can decide to open certain commitments and reveal the attributes.

There is a great non-cryptographic analogy that explains commitments really well. Assume that you have data that you want to protect from others to see and even from you to change. You put that in an envelope, seal and send it to the public. No one can see it because it's sealed and you can not change it because it's out now.

```
let commitments = match &credential.value {
        AccountCredentialWithoutProofs::Initial { icdv: _, .. } => {
            return Err(InjectStatementError::NotAllowed);
        }
        AccountCredentialWithoutProofs::Normal { commitments, .. } => commitments,
    };
```

And finally, we verify the proof with this part and respond back with the result which is the signature. We need the _request_, _global\_context_, _cred\_id_, _commitments_, and the _proof_ itself to do that. If it's successful, we can remove the challenge from our map-since it's a one-time thing- and sign the account address (as string) with our private key. We used this approach to create and share the signature but it's also fine to sign any message. In the smart contract, while minting, we would like to verify that the claim is verified and signed with our private key. It may sound a bit complicated but you will understand it better while implementing the dApp.

```
if statement.verify(
        &request.challenge.0,
        &state.global_context,
        cred_id.as_ref(),
        commitments,
        &request.proof.proof.value, // TODO: Check version.
    ) {
        challenges.remove(&base16_encode_string(&request.challenge.0));
        let sig = key_pair.sign(&acc_info.response.account_address.0);
        Ok(hex::encode_upper(sig.sig))
    } else {
        Err(InjectStatementError::InvalidProofs)
    }
```

**Main.rs**

Dont worry! We are almost there, now we need to wrap it up and create our main program to run our HTTP server that listens to all endpoints required to create and send a challenge, share the statement, and verify the claim. Create a file called _main.rs._ We will use _warp_ to run an async HTTP server in a few easy steps as already mentioned and definitely need the _handlers.rs_ and _types.rs_ to invoke the helper functions and the data structures.

We need to create a struct called _IdVerifierConfig_ that accepts command line parameters while running the application. First, it should have a node _endpoint_ to build & configure HTTP/2 channels. (which gRPCv2 uses to stream). Second, we will need a port for our server to listen and a logger which we will use the _log_ crate. And finally, we will give the _statement_, _verify\_key_ and _sign\_key_ (the keys we get from our exported wallet file) as parameters in string form. Please note that for all parameters we specified some default values with the _clap._

```
mod handlers;
mod types;
use crate::handlers::*;
use crate::types::*;

use clap::Parser;
use concordium_rust_sdk::common::types::KeyPair;
use concordium_rust_sdk::{
    common::{self as crypto_common},
    id::{
        constants::{ArCurve, AttributeKind},
        id_proof_types::Statement,
    },
    v2::BlockIdentifier,
};
use log::info;
use std::{
    collections::HashMap,
    sync::{Arc, Mutex},
};
use warp::Filter;

/// Structure used to receive the correct command line arguments.
#[derive(clap::Parser, Debug)]
#[clap(arg_required_else_help(true))]
#[clap(version, author)]
struct IdVerifierConfig {
    #[clap(
        long = "node",
        help = "GRPC V2 interface of the node.",
        default_value = "http://localhost:20000"
    )]
    endpoint: concordium_rust_sdk::v2::Endpoint,
    #[clap(
        long = "port",
        default_value = "8100",
        help = "Port on which the server will listen on."
    )]
    port: u16,
    #[structopt(
        long = "log-level",
        default_value = "debug",
        help = "Maximum log level."
    )]
    log_level: log::LevelFilter,
    #[clap(
        long = "statement",
        help = "The statement that the server accepts proofs for."
    )]
    statement: String,
    #[structopt(
        long = "sign-key",
        help = "Sign key of the first credential of the signer"
    )]
    sign_key: String,
    #[structopt(
        long = "verify-key",
        help = "Verify key of the first credential of the signer"
    )]
    verify_key: String,
}
```

As the final step, we will add our _main()_ function. Let’s add _#\[tokio::main]_ macro just before main. It transforms the async _main()_ function into a synchronous _main()_ function that initializes a runtime instance and executes the async _main()_ function.

First we need to parse the params given as input while running the executable. After initializing the log file, we serialize the statement (see the _concordium-rust-sdk_ for more details), create a _client_ and get the latest cryptographic parameters which are public and our _global\_context_ until the finalized last block from Concordium. (or the request made) Create a state variable(initiate it) with empty challenges, and the global context.

**Get Challenge**

cors is a standard in HTTP related to the permissions to access and manage a website we set our server's settings, and then implement the first endpoint which is the getting challenge. So when someone wants to get a randomly generated challenge for his address, he/she must call this endpoint. We will get the address from the query payload and invoke _handle\_get\_challange()._ Since we don't need input this is a _GET_ function that is available at “_localhost:8000/api/challenge_”, and we will use the same channel when the challenge is generated, and stored on the state in a map with its key like (base16 encoded version, \<address, time>)

**Get Statement**

The second endpoint is the get statement, when our dApp wants to verify that a user meets some conditions, it needs to know what conditions they are, we will just answer back the statement from our input variables using the same channel. We don't need input so this is also a _GET_ endpoint that is available at “_localhost:8000/api/statement_”

**Prove**

The last endpoint is the prove. Basically, the request that dApp posts, (so this is a POST endpoint) include the challenge and the proof. We will send it to the _handle\_provide\_proof()_ helper function to proof and sign it. In order to sign it, we need to re-create our key pair which are created using our verify\_key and sign\_key. Basically, when the proof is verified, this endpoint sends back a signature (the public key of the user signed by the backend’s private key) that can be verifiable in the smart contract. Then the user will be able to mint the token because the signature will be verifiable by the smart contract using the public key of the backend address.

```

#[tokio::main]
async fn main() -> anyhow::Result<()> {
    let app = IdVerifierConfig::parse();
    let mut log_builder = env_logger::Builder::new();
    // only log the current module (main).
    log_builder.filter_level(app.log_level); // filter filter_module(module_path!(), app.log_level);
    log_builder.init();
    let statement: Statement<ArCurve, AttributeKind> = serde_json::from_str(&app.statement)?;

    let mut client = concordium_rust_sdk::v2::Client::new(app.endpoint).await?;
    let global_context = client
        .get_cryptographic_parameters(BlockIdentifier::LastFinal)
        .await?
        .response;

    log::debug!("Acquired data from the node.");

    let state = Server {
        challenges: Arc::new(Mutex::new(HashMap::new())),
        global_context: Arc::new(global_context),
    };
    let prove_state = state.clone();
    let challenge_state = state.clone();

    let cors = warp::cors()
        .allow_any_origin()
        .allow_header("Content-Type")
        .allow_method("POST");

    // 1a. get challenge
    let get_challenge = warp::get()
        .and(warp::path!("api" / "challenge"))
        .and(warp::query::<WithAccountAddress>())
        .and_then(move |query: WithAccountAddress| {
            handle_get_challenge(challenge_state.clone(), query.address)
        });

    // 1b. get statement
    // change it to check older than 18 only.
    let get_statement = warp::get()
        .and(warp::path!("api" / "statement"))
        .map(move || warp::reply::json(&app.statement));

    // 2. Provide proof
    let provide_proof = warp::post()
        .and(warp::filters::body::content_length_limit(50 * 1024))
        .and(warp::path!("api" / "prove"))
        .and(warp::body::json())
        .and_then(move |request: ChallengedProof| {
            let kp = KeyPair::from(ed25519_dalek::Keypair {
                public: ed25519_dalek::PublicKey::from_bytes(
                    hex::decode(&app.verify_key).unwrap().as_slice(),
                )
                .unwrap(),
                secret: ed25519_dalek::SecretKey::from_bytes(
                    hex::decode(&app.sign_key).unwrap().as_slice(),
                )
                .unwrap(),
            });
            handle_provide_proof(
                client.clone(),
                prove_state.clone(),
                statement.clone(),
                request,
                kp,
            )
        });

    info!(
        "Starting up HTTP serve
r. Listening on port {}.",
        app.port
    );

    tokio::spawn(handle_clean_state(state.clone()));

    let server = get_challenge
        .or(get_statement)
        .or(provide_proof)
        .recover(handle_rejection)
        .with(cors)
        .with(warp::trace::request());
    warp::serve(server).run(([0, 0, 0, 0], app.port)).await;
    Ok(())
}
```
