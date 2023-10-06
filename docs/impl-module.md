# 如何使用模块系统创建新模块

## 了解模块系统

主权软件开发套件 (SDK) 包括一个模块系统，它充当汇总接口的具体且固定的实现的目录。这些模块是汇总的基本构建块，包括：

- 协议级逻辑：包括账户管理、状态管理逻辑、其他模块的API以及生成RPC的宏等元素。它为您的汇总提供了蓝图。
- 应用程序级逻辑：这类似于以太坊上的智能合约或 Polkadot 上的托盘。这些模块通常使用状态、模块 API 和宏模块来简化其开发和操作。

## 创建不可替代代币 (NFT) 模块

注意：本教程重点通过创建一个简单的 NFT 模块来说明 Sovereign SDK 的用法。这里的重点是模块系统而不是应用程序逻辑。更完整的NFT模块请参考[sov-nft-module](https://github.com/Sovereign-Labs/sovereign-sdk/tree/nightly/module-system/module-implementations/sov-nft-module)

在本教程中，我们将重点开发应用程序级模块。该模块的用户将能够铸造独特的代币、相互转让或销毁它们。用户还可以检查特定代币的所有权。为简单起见，每个令牌仅代表一个 ID，不会保存任何元数据。

##  入门

### 结构和依赖关系

Sovereign SDK 提供了一个[模块模板](https://github.com/Sovereign-Labs/sovereign-sdk/blob/nightly/module-system/module-implementations/module-template/README.md)，它是可以定制以轻松构建模块的样板。

```bash
├── Cargo.toml
├── README.md
└── src
    ├── call.rs
    ├── genesis.rs
    ├── lib.rs
    ├── query.rs
    └── tests.rs
```

以下是在 Cargo.toml 中定义模块启动所需的基本依赖项：

```toml
[dependencies]
anyhow = { anyhow = "1.0.62" }
sov-modules-api = { git = "https://github.com/Sovereign-Labs/sovereign-sdk.git", branch = "stable", features = ["macros"] }
```

### 建立根模块结构

模块是一个实现 sov_modules_api::Module 特征的独特包。每个模块都有私有状态，它会根据输入消息进行更新。

### 模块定义

NFT模块定义如下：定义的存储内容

```rust
#[derive(sov_modules_api::ModuleInfo, Clone)]
pub struct NonFungibleToken<C: sov_modules_api::Context> {
    #[address]
    address: C::Address,

    #[state]
    admin: sov_modules_api::StateValue<C::Address>,

    #[state]
    owners: sov_modules_api::StateMap<u64, C::Address>,

    // If the module needs to refer to another module
    // #[module]
    // bank: sov_bank::Bank<C>,
}
```

该模块包括：

1. 地址：每个模块都必须有一个地址，就像以太坊中的智能合约地址一样。这可以确保：
    - 模块地址是唯一的。
    - 生成该地址的私钥未知。
2. 状态属性：在本例中，状态属性是管理员的地址以及令牌 ID 到所有者地址的映射。为简单起见，令牌 ID 是 u64。
3. 可选模块引用：如果模块需要引用另一个模块，则使用此选项。

### State和Context

#### State

#[state] 在模块中声明的值并不物理存储在模块中。相反，模块定义只是声明它将访问的值的类型。这些值本身存在于一个名为 WorkingSet 的特殊结构中，它抽象了存储的实现细节。在默认实现中，实际状态值位于 Jellyfish Merkle Tree (JMT) 中。功能（由 Module 定义）和状态（由 WorkingSet 提供）之间的分离解释了为什么如此多的模块方法采用 WorkingSet 作为参数。

#### Context

Context 特征允许运行时在执行期间将经过验证的数据传递给模块。目前，Context 中唯一需要的方法是 sender()，它返回发起交易的个人（签名者）的地址。

Context 还继承了 Spec 特征，它定义了汇总用于散列、持久数据存储、数字签名和地址的具体类型。 Spec 特性允许 rollup 轻松地根据不同的 ZK VM 进行定制。通过在规范上通用，汇总可以确保任何可能对 SNARK 不友好的加密技术都可以轻松替换。

### 实现 sov_modules_api::Module 特征

#### 准备

在我们开始实现 Module 特征之前，需要执行几个准备步骤：

1. 在 Cargo.toml 中定义 native 功能并添加其他依赖项：

```toml
[dependencies]
anyhow = "1.0.62"
borsh = { version = "0.10.3", features = ["bytes"] }
serde = { version = "1", features = ["derive"] }
serde_json = "1"

sov-modules-api = { git = "https://github.com/Sovereign-Labs/sovereign-sdk.git", branch = "stable", default-features = false, features = ["macros"] }
sov-state = { git = "https://github.com/Sovereign-Labs/sovereign-sdk.git", branch = "stable", default-features = false }

[features]
default = ["native"]
serde = ["dep:serde", "dep:serde_json"]
native = ["serde", "sov-state/native", "sov-modules-api/native"]
```

此步骤对于优化在 ZK 模式下执行的模块是必要的，在 ZK 模式下不需要任何 RPC 相关的逻辑。零知识模式使用不同的序列化格式，因此不需要 serde。 sov-state 模块保持相同的逻辑，因此它的 native 标志仅在这种情况下启用。

2. 定义 Call 消息，用于更改模块的状态：

> 理解，定义的不同的调用指令方法

```rust
// in call.rs
#[cfg_attr(feature = "native", derive(serde::Serialize), derive(serde::Deserialize))]
#[derive(borsh::BorshDeserialize, borsh::BorshSerialize, Debug, PartialEq, Clone)]
pub enum CallMessage<C: sov_modules_api::Context> {
    Mint {
        /// The id of new token. Caller is an owner
        id: u64,
    },
    Transfer {
        /// The address to which the token will be transferred.
        to: C::Address,
        /// The token id to transfer.
        id: u64,
    },
    Burn {
        id: u64,
    }
}
```

正如您所看到的，我们为这些消息派生了 borsh 序列化格式。与大多数序列化库不同， borsh 保证所有消息都具有单个“规范”序列化，这使得可靠地散列和比较序列化消息变得很容易。

3. 为创世配置创建一个 Config 结构。在这种情况下，管理地址和初始令牌分配是可配置的：

```rust
// in lib.rs
pub struct NonFungibleTokenConfig<C: sov_modules_api::Context> {
    pub admin: C::Address,
    pub owners: Vec<(u64, C::Address)>,
}
```

### Module trait 的存根实现

将所有类型和功能组合在一起，我们在 lib.rs 中得到这个 Module 特征实现：

```rust
impl<C: sov_modules_api::Context> Module for NonFungibleToken<C> {
    type Context = C;
    type Config = NonFungibleTokenConfig<C>;
    type CallMessage = CallMessage<C>;

    fn genesis(
        &self,
        _config: &Self::Config,
        _working_set: &mut WorkingSet<C>,
    ) -> anyhow::Result<(), Error> {
        Ok(())
    }

    fn call(
        &self,
        _msg: Self::CallMessage,
        _context: &Self::Context,
        _working_set: &mut WorkingSet<C>,
    ) -> anyhow::Result<sov_modules_api::CallResponse, Error> {
        Ok(sov_modules_api::CallResponse::default())
    }
}
```

### 实现状态更改逻辑

#### 初始化

初始化由 genesis 方法执行，该方法采用指定要配置的初始状态的配置参数。由于它修改状态， genesis 也将工作集作为参数。 Genesis 在汇总部署期间仅调用一次。

```rust
use sov_modules_api::WorkingSet;

// in lib.rs
impl<C: sov_modules_api::Context> sov_modules_api::Module for NonFungibleToken<C> {
    type Context = C;
    type Config = NonFungibleTokenConfig<C>;
    type CallMessage = CallMessage<C>;

    fn genesis(
        &self,
        config: &Self::Config,
        working_set: &mut WorkingSet<C>,
    ) -> Result<(), Error> {
        Ok(self.init_module(config, working_set)?)
    }
}

// in genesis.rs
impl<C: sov_modules_api::Context> NonFungibleToken<C> {
    pub(crate) fn init_module(
        &self,
        config: &<Self as sov_modules_api::Module>::Config,
        working_set: &mut WorkingSet<C>,
    ) -> anyhow::Result<()> {
        self.admin.set(&config.admin, working_set);
        for (id, owner) in config.owners.iter() {
            if self.owners.get(id, working_set).is_some() {
                anyhow::bail!("Token id {} already exists", id);
            }
            self.owners.set(id, owner, working_set);
        }
        Ok(())
    }
}
```

### Call message

首先，我们需要实现处理不同情况的实际逻辑。让我们添加 mint 、 transfer 和 burn 方法：


```rust
use sov_modules_api::WorkingSet;

impl<C: sov_modules_api::Context> NonFungibleToken<C> {
    pub(crate) fn mint(
        &self,
        id: u64,
        context: &C,
        working_set: &mut WorkingSet<C>,
    ) -> anyhow::Result<sov_modules_api::CallResponse> {
        if self.owners.get(&id, working_set).is_some() {
            bail!("Token with id {} already exists", id);
        }

        self.owners.set(&id, context.sender(), working_set);

        working_set.add_event("NFT mint", &format!("A token with id {id} was minted"));
        Ok(sov_modules_api::CallResponse::default())
    }

    pub(crate) fn transfer(
        &self,
        id: u64,
        to: C::Address,
        context: &C,
        working_set: &mut WorkingSet<C>,
    ) -> anyhow::Result<sov_modules_api::CallResponse> {
        let token_owner = match self.owners.get(&id, working_set) {
            None => {
                anyhow::bail!("Token with id {} does not exist", id);
            }
            Some(owner) => owner,
        };
        if &token_owner != context.sender() {
            anyhow::bail!("Only token owner can transfer token");
        }
        self.owners.set(&id, &to, working_set);
        working_set.add_event(
            "NFT transfer",
            &format!("A token with id {id} was transferred"),
        );
        Ok(sov_modules_api::CallResponse::default())
    }

    pub(crate) fn burn(
        &self,
        id: u64,
        context: &C,
        working_set: &mut WorkingSet<C>,
    ) -> anyhow::Result<sov_modules_api::CallResponse> {
        let token_owner = match self.owners.get(&id, working_set) {
            None => {
                anyhow::bail!("Token with id {} does not exist", id);
            }
            Some(owner) => owner,
        };
        if &token_owner != context.sender() {
            anyhow::bail!("Only token owner can burn token");
        }
        self.owners.remove(&id, working_set);

        working_set.add_event("NFT burn", &format!("A token with id {id} was burned"));
        Ok(sov_modules_api::CallResponse::default())
    }
}
```

然后让用户通过 call 函数访问它们：

```rust
impl<C: sov_modules_api::Context> sov_modules_api::Module for NonFungibleToken<C> {
    type Context = C;
    type Config = NonFungibleTokenConfig<C>;

    fn call(
        &self,
        msg: Self::CallMessage,
        context: &Self::Context,
        working_set: &mut WorkingSet<C>,
    ) -> Result<sov_modules_api::CallResponse, Error> {
        let call_result = match msg {
            CallMessage::Mint { id } => self.mint(id, context, working_set),
            CallMessage::Transfer { to, id } => self.transfer(id, to, context, working_set),
            CallMessage::Burn { id } => self.burn(id, context, working_set),
        };
        Ok(call_result?)
    }
}
```

### 启用查询

我们还希望其他模块能够查询令牌的所有者，因此我们为此添加了一个公共方法。此方法仅适用于其他模块：当前不通过 RPC 公开。

```rust
use jsonrpsee::core::RpcResult;
use sov_modules_api::macros::rpc_gen;
use sov_modules_api::{Context, WorkingSet};

#[derive(Clone, Debug, Eq, PartialEq, serde::Deserialize, serde::Serialize)]
/// Response for `getOwner` method
pub struct OwnerResponse<C: Context> {
    /// Optional owner address
    pub owner: Option<C::Address>,
}

#[rpc_gen(client, server, namespace = "nft")]
impl<C: sov_modules_api::Context> NonFungibleToken<C> {
    #[rpc_method(name = "getOwner")]
    pub fn get_owner(
        &self,
        token_id: u64,
        working_set: &mut WorkingSet<C>,
    ) -> RpcResult<OwnerResponse<C>> {
        Ok(OwnerResponse {
            owner: self.owners.get(&token_id, working_set),
        })
    }
}
```

### 测试

建议进行集成测试以确保模块正确实现。这有助于确认所有公共 API 均按预期运行。

测试需要临时存储，因此我们将 sov-state 的 temp 功能启用为 dev-dependency

```toml
[dev-dependencies]
sov-state = { git = "https://github.com/Sovereign-Labs/sovereign-sdk.git", branch = "stable", features = ["temp"] }
```

以下是 NFT 模块集成测试的一些样板：

```rust
use simple_nft_module::{CallMessage, NonFungibleToken, NonFungibleTokenConfig, OwnerResponse};
use sov_modules_api::default_context::DefaultContext;
use sov_modules_api::{Address, Context, Module, WorkingSet};
use sov_rollup_interface::stf::Event;
use sov_state::{DefaultStorageSpec, ProverStorage};

pub type C = DefaultContext;
pub type Storage = ProverStorage<DefaultStorageSpec>;


#[test]
#[ignore = "Not implemented yet"]
fn genesis_and_mint() {}

#[test]
#[ignore = "Not implemented yet"]
fn transfer() {}

#[test]
#[ignore = "Not implemented yet"]
fn burn() {}
```

下面是设置模块并调用其方法的示例：

```rust
#[test]
fn transfer() {
    // Preparation
    let admin = generate_address::<C>("admin");
    let admin_context = C::new(admin.clone());
    let owner1 = generate_address::<C>("owner2");
    let owner1_context = C::new(owner1.clone());
    let owner2 = generate_address::<C>("owner2");
    let config: NonFungibleTokenConfig<C> = NonFungibleTokenConfig {
        admin: admin.clone(),
        owners: vec![(0, admin.clone()), (1, owner1.clone()), (2, owner2.clone())],
    };
    let mut working_set = WorkingSet::new(ProverStorage::temporary());
    let nft = NonFungibleToken::new();
    nft.genesis(&config, &mut working_set).unwrap();

    let transfer_message = CallMessage::Transfer {
        id: 1,
        to: owner2.clone(),
    };

    // admin cannot transfer token of the owner1
    let transfer_attempt = nft.call(transfer_message.clone(), &admin_context, &mut working_set);

    assert!(transfer_attempt.is_err());
    // ... rest of the tests
}
```

### 插入汇总

现在可以将此模块添加到汇总的 Runtime 中：

```rust
use sov_modules_api::{DispatchCall, Genesis, MessageCodec};

#[derive(Genesis, DispatchCall, MessageCodec)]
#[serialization(borsh::BorshDeserialize, borsh::BorshSerialize)]
pub struct Runtime<C: sov_modules_api::Context> {
    #[allow(unused)]
    sequencer: sov_sequencer_registry::Sequencer<C>,

    #[allow(unused)]
    bank: sov_bank::Bank<C>,

    #[allow(unused)]
    nft: simple_nft_module::NonFungibleToken<C>,
}
```
