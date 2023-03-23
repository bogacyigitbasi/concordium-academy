# Rust Installation

First, you need to install `rustup` which installs Rust and Cargo to your computer. Go to [Rustup](https://rustup.rs/) to install `rustup` it for your platform.

Select **1** to continue the installation.

<figure><img src="https://developer.concordium.software/en/mainnet/_images/mint-install-rustup.png" alt=""><figcaption></figcaption></figure>

Finally, when Rust and Cargo are successfully installed in your system, you should see something similar to the one below.

<figure><img src="https://developer.concordium.software/en/mainnet/_images/mint-rust-install-done.png" alt=""><figcaption></figcaption></figure>

Copy and paste the commands in a terminal to install Wasm which will be used for building contracts.

```
rustup target add wasm32-unknown-unknown
```

During Wasm installation in your system you should see something similar to the one below.

<figure><img src="https://developer.concordium.software/en/mainnet/_images/mint-wasm-install.png" alt=""><figcaption></figcaption></figure>

Now you need to install the Concordium software package. [Click here](https://developer.concordium.software/en/mainnet/net/installation/downloads-testnet.html#cargo-concordium-testnet) and download the version 2.2.0 or greater of `cargo-concordium` for your operating system. The tool is the same for both testnet and mainnet.

First, rename the `cargo-congordium-v.x.x` file to `cargo-concordium`. Then go to the directory where the file is downloaded and run this command to make it executable. You also need to move the `cargo-concordium` executable to the cargo folder. [Follow the information here](https://developer.concordium.software/en/mainnet/smart-contracts/guides/setup-tools.html#setup-tools) to ensure that your cargo-concordium is configured correctly. The commands below are specifically for MacOS. Remember to adjust the commands based on your operating system.

```
sudo chmod +x cargo-concordium
```

```
mv cargo-concordium ~/.cargo/bin
```

If everything is correct, when you enter the command `cargo concordium --help` it shows something similar to the below.

<figure><img src="https://developer.concordium.software/en/mainnet/_images/cargo-help.png" alt=""><figcaption></figcaption></figure>

Note

If you have a warning on a Mac device that says “cargo-concordium cannot be opened because the developer cannot be verified” that means it requires permission to run and you should go to **System Preferences → Security** and unlock it with your password and click **Allow Anyway**.

\


<figure><img src="https://developer.concordium.software/en/mainnet/_images/mac-warning.png" alt=""><figcaption></figcaption></figure>
