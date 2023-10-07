# Write A Sovereign Module Guide

1. use Soveregin SDK Module template

- [Module template](https://github.com/DaviRain-Su/sovereign/tree/research/module-system/module-implementations/module-template)

2. update you module crate Cargo.toml

This is a example of simple-coin module Cargo.toml

```toml
[package]
name = "simple-coin"
authors = { workspace = true }
edition = { workspace = true }
homepage = { workspace = true }
license = { workspace = true }
repository = { workspace = true }
rust-version = { workspace = true }
version = { workspace = true }
publish = false
resolver = "2"

[dependencies]
anyhow = { workspace = true }
borsh = { workspace = true, features = ["rc"] }
thiserror = { workspace = true }
schemars = { workspace = true, optional = true }
serde = { workspace = true }
serde_json = { workspace = true, optional = true }
jsonrpsee = { workspace = true, features = ["macros", "client-core", "server"], optional = true }
clap = { workspace = true, optional = true, features = ["derive"] }

sov-modules-api = { path = "../../module-system/sov-modules-api" }
sov-state = { path = "../../module-system/sov-state" }
sov-rollup-interface = { path = "../../rollup-interface" }

[dev-dependencies]
tempfile = { workspace = true }
simple-coin = { path = ".", version = "*", features = ["native"] }

[features]
default = []
native = ["serde_json", "jsonrpsee", "clap", "schemars", "sov-state/native", "sov-modules-api/native", ]
```

3. Add SimpleToken Storage and Config Struct

```rust
mod call;
mod genesis;
#[cfg(feature = "native")]
mod query;
// pub use call::CallMessage;
#[cfg(feature = "native")]
pub use query::*;
use serde::{Deserialize, Serialize};
use sov_modules_api::{Error, ModuleInfo, WorkingSet};

#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct SimpleTokenConfig<C: sov_modules_api::Context> {
    pub supply: u64,
    pub creator: C::Address,
}

/// A new module:
/// - Must derive `ModuleInfo`
/// - Must contain `[address]` field
/// - Can contain any number of ` #[state]` or `[module]` fields
/// - Should derive `ModuleCallJsonSchema` if the "native" feature is enabled.
///   This is optional, and is only used to generate a JSON Schema for your
///   module's call messages (which is useful to develop clients, CLI tooling
///   etc.).
#[cfg_attr(feature = "native", derive(sov_modules_api::ModuleCallJsonSchema))]
#[derive(ModuleInfo)]
pub struct SimpleToken<C: sov_modules_api::Context> {
    /// Address of the module.
    #[address]
    pub address: C::Address,

    /// Some value kept in the state.
    #[state]
    pub supply: sov_modules_api::StateValue<u64>,

    #[state]
    pub(crate) balances: sov_modules_api::StateMap<C::Address, u64>,
}

impl<C: sov_modules_api::Context> sov_modules_api::Module for SimpleToken<C> {
    type Context = C;

    type Config = SimpleTokenConfig<C>;

    type CallMessage = ();

    fn genesis(&self, config: &Self::Config, working_set: &mut WorkingSet<C>) -> Result<(), Error> {
        todo!()
    }

    fn call(
        &self,
        msg: Self::CallMessage,
        context: &Self::Context,
        working_set: &mut WorkingSet<C>,
    ) -> Result<sov_modules_api::CallResponse, Error> {
        todo!()
    }
}
```

4. Add SimpleToken Genesis

```rust
use anyhow::Result;
use sov_modules_api::WorkingSet;

use crate::SimpleToken;

impl<C: sov_modules_api::Context> SimpleToken<C> {
    pub(crate) fn init_module(
        &self,
        config: &<Self as sov_modules_api::Module>::Config,
        working_set: &mut WorkingSet<C>,
    ) -> Result<()> {
        self.supply.set(&config.supply, working_set);
        self.balances
            .set(&config.creator, &config.supply, working_set);
        Ok(())
    }
}
```

update lib.rs

```rust
impl<C: sov_modules_api::Context> sov_modules_api::Module for SimpleToken<C> {
    type Context = C;

    type Config = SimpleTokenConfig<C>;

    type CallMessage = ();

    fn genesis(&self, config: &Self::Config, working_set: &mut WorkingSet<C>) -> Result<(), Error> {
        Ok(self.init_module(config, working_set)?)
    }

    fn call(
        &self,
        msg: Self::CallMessage,
        context: &Self::Context,
        working_set: &mut WorkingSet<C>,
    ) -> Result<sov_modules_api::CallResponse, Error> {
        todo!()
    }
}
```

5. Add SimpleToken Call

```rust
use std::fmt::Debug;

use anyhow::Result;
use sov_modules_api::{CallResponse, WorkingSet};
use thiserror::Error;

use crate::SimpleToken;

/// This enumeration represents the available call messages for interacting with
/// the `SimpleToken` module.
/// The `derive` for [`schemars::JsonSchema`] is a requirement of
/// [`sov_modules_api::ModuleCallJsonSchema`].
#[cfg_attr(
    feature = "native",
    derive(serde::Serialize),
    derive(serde::Deserialize),
    derive(schemars::JsonSchema),
    schemars(bound = "C::Address: ::schemars::JsonSchema", rename = "CallMessage")
)]
#[derive(borsh::BorshDeserialize, borsh::BorshSerialize, Debug, PartialEq, Clone)]
pub enum CallMessage<C: sov_modules_api::Context> {
    /// Transfers a specified amount of tokens to the specified address.
    Transfer {
        /// The address to which the tokens will be transferred.
        to: C::Address,
        /// The amount of tokens to transfer.
        amount: u64,
    },
}

/// Example of a custom error.
#[derive(Debug, Error)]
enum SimpleTokenError {}

impl<C: sov_modules_api::Context> SimpleToken<C> {
    /// Sets `value` field to the `new_value`
    pub(crate) fn transfer(
        &self,
        to: C::Address,
        amount: u64,
        context: &C,
        working_set: &mut WorkingSet<C>,
    ) -> Result<sov_modules_api::CallResponse> {
        let caller = context.sender();
        let b = self
            .balances
            .get(caller, working_set)
            .ok_or(anyhow::anyhow!("sender have not token"))?;

        if b <= amount {
            return Err(anyhow::anyhow!("not enough founds"));
        }

        let old_balance = self.balances.get(&to, working_set).unwrap_or(0);

        let new_to_balance = old_balance + amount;
        let new_from_balance = b - amount;
        self.balances.set(caller, &new_from_balance, working_set);
        self.balances.set(&to, &new_to_balance, working_set);

        Ok(CallResponse::default())
    }
}
```

update lib.rs

```rust
impl<C: sov_modules_api::Context> sov_modules_api::Module for SimpleToken<C> {
    type Context = C;

    type Config = SimpleTokenConfig<C>;

    type CallMessage = ();

    fn genesis(&self, config: &Self::Config, working_set: &mut WorkingSet<C>) -> Result<(), Error> {
        Ok(self.init_module(config, working_set)?)
    }

    fn call(
        &self,
        msg: Self::CallMessage,
        context: &Self::Context,
        working_set: &mut WorkingSet<C>,
    ) -> Result<sov_modules_api::CallResponse, Error> {
        match msg {
            CallMessage::Transfer { to, amount } => {
                Ok(self.transfer(to, amount, context, working_set)?)
            }
        }
    }
}
```

6. Add SimpleToken rpc query

```rust
use jsonrpsee::core::RpcResult;
use sov_modules_api::macros::rpc_gen;
use sov_modules_api::WorkingSet;

use super::SimpleToken;

#[derive(serde::Serialize, serde::Deserialize, Debug, Eq, PartialEq, Clone)]
pub struct BalanceResponse {
    pub value: Option<u64>,
}

#[rpc_gen(client, server, namespace = "simpletoken")]
impl<C: sov_modules_api::Context> SimpleToken<C> {
    #[rpc_method(name = "GetBalance")]
    /// Queries accocunt balance.
    pub fn get_balance(
        &self,
        account: C::Address,
        working_set: &mut WorkingSet<C>,
    ) -> RpcResult<BalanceResponse> {
        let balance = self.balances.get(&account, working_set);
        Ok(BalanceResponse { value: balance })
    }
}
```

7. add simple-token module to demo-stf runtime

update demo-stf Cargo.toml

```toml
@ -31,6 +31,7 @@ sov-blob-storage = { path = "../../module-system/module-implementations/sov-blob
sov-bank = { path = "../../module-system/module-implementations/sov-bank" }
sov-nft-module = { path = "../../module-system/module-implementations/sov-nft-module" }
simple-nft-module = { path = "../simple-nft-module" }
# Add simple coin moduel
simple-coin = { path = "../simple-coin" }

sov-chain-state = { path = "../../module-system/module-implementations/sov-chain-state" }
sov-modules-stf-template = { path = "../../module-system/sov-modules-stf-template" }
@ -59,6 +60,7 @@ native = [
    "sov-bank/native",
    "sov-nft-module/native",
    "simple-nft-module/native",
    "simple-coin/native", # add this
    "sov-cli",
    "sov-accounts/native",
    "sov-sequencer-registry/native",
```

Add demo-stf/src/runtime.rs

```rust
// add this ---
#[cfg(feature = "native")]
use simple_coin::{SimpleTokenRpcImpl, SimpleTokenRpcServer};
// ---
#[cfg(feature = "native")]
use sov_accounts::{AccountsRpcImpl, AccountsRpcServer};
#[cfg(feature = "native")]
use sov_bank::{BankRpcImpl, BankRpcServer};
```

8. update demo-stf/src/gensis_config.rs

```rust
@ -3,6 +3,7 @@ use std::path::Path;

use anyhow::{bail, Context as AnyhowContext};
use serde::de::DeserializeOwned;
// add this ---
use simple_coin::SimpleTokenConfig;
// ---
use sov_accounts::AccountConfig;
use sov_bank::BankConfig;
use sov_chain_state::ChainStateConfig;
@ -98,6 +99,11 @@ fn create_genesis_config<C: Context, Da: DaSpec, P: AsRef<Path>>(
    let chain_state_config: ChainStateConfig =
        read_json_file(&genesis_paths.chain_state_genesis_path)?;

    // add this ----
    let simple_coin_config = SimpleTokenConfig {
        supply: 1000,
        creator: value_setter_config.admin.clone(),
    };
    // ---

    #[cfg(feature = "experimental")]
    let evm_config = get_evm_config(&genesis_paths.evm_genesis_path, eth_signers)?;

@ -111,6 +117,7 @@ fn create_genesis_config<C: Context, Da: DaSpec, P: AsRef<Path>>(
        #[cfg(feature = "experimental")]
        evm_config,
        nft_config,
        simple_coin_config, // add this
    ))
}

```

9. update demo-stf/src/runtime.rs

```rust
pub value_setter: sov_value_setter::ValueSetter<C>,
pub accounts: sov_accounts::Accounts<C>,
pub nft: sov_nft_module::NonFungibleToken<C>,
// add this ---
pub simple_token: simple_coin::SimpleToken<C>,
// ---
}

#[cfg(feature = "experimental")]
@ -101,6 +102,7 @@ pub struct Runtime<C: Context, Da: DaSpec> {
#[cfg_attr(feature = "native", cli_skip)]
pub evm: sov_evm::Evm<C>,
pub nft: sov_nft_module::NonFungibleToken<C>,
// add this  ---
pub simple_token: simple_coin::SimpleToken<C>,
// --
}

impl<C, Da> sov_modules_stf_template::Runtime<C, Da> for Runtime<C, Da>
```

10. add transaction json file to test-data/requests/simple_token_transfer.json

```json
{
    "Transfer": {
        "to": "sov1l6n2cku82yfqld30lanm2nfw43n2auc8clw7r5u5m6s7p8jrm4zqklh0qh",
        "amount": 200
    }
}
```


## Start demo-rollup

参照 demo-rollup的教程

https://github.com/DaviRain-Su/sovereign/tree/research/examples/demo-rollup#getting-started

1.  cd examples/demo-rollup/
2.  make clean && make start
3.  cargo run

## query simple-token balance

```bash
 curl -X POST -H "Content-Type: application/json" -d '{"jsonrpc":"2.0","method":"simpletoken_GetBalance","params":["sov1l6n2cku82yfqld30lanm2nfw43n2auc8clw7r5u5m6s7p8jrm4zqrr8r94"],"id":1}'
```

```bash
{"jsonrpc":"2.0","result":{"value":1000},"id":1}
```

## improt a wallet

```bash
make import-keys
```

## make a transaction

```bash
cargo run --bin sov-cli -- transactions import from-file simple-token --path ../test-data/requests/simple_token_transfer.json
```

## submit a transaction

```bash
cargo run --bin sov-cli rpc submit-batch by-address sov1l6n2cku82yfqld30lanm2nfw43n2auc8clw7r5u5m6s7p8jrm4zqrr8r94
```

## query simple-token balance

```bash
```bash
 curl -X POST -H "Content-Type: application/json" -d '{"jsonrpc":"2.0","method":"simpletoken_GetBalance","params":["sov1l6n2cku82yfqld30lanm2nfw43n2auc8clw7r5u5m6s7p8jrm4zqrr8r94"],"id":1}'
```

```bash
{"jsonrpc":"2.0","result":{"value":1200},"id":1}
```
