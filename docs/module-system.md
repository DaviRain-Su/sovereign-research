# 模块系统

该目录包含一个用于使用 Sovereign SDK 构建汇总的固定框架。它旨在提供“包含电池batteries included”的开发体验。使用模块系统仍然允许您自定义汇总的关键组件，例如哈希函数和签名方案，但它也迫使您依赖一些合理的默认值，例如序列化方案（Borsh）、地址格式（bech32）等。

通过使用模块系统进行开发，您可以访问一套预构建的模块，支持常见功能，例如生成帐户、铸造和转移代币以及激励排序器。您还可以使用用于生成 RPC 实现的强大工具，以及用于实现复杂状态转换的强大模板系统。


## 模块：基本构建块

模块系统的基本构建块是 module 。模块是 Rust 中的结构，需要实现 Module trait。您可以在此处找到展示如何实现自定义模块的[完整教程](./impl-module.md)。模块通常位于自己的板条箱中（您可以在此处找到[模板](https://github.com/Sovereign-Labs/sovereign-sdk/tree/nightly/module-system/module-implementations/module-template)），以便可以轻松地重复使用它们。模块的典型结构定义如下所示：

```rust
#[derive(ModuleInfo)]
pub struct Bank<C: sov_modules_api::Context> {
    /// The address of the bank module.
    #[address]
    pub(crate) address: C::Address,

    /// A mapping of addresses to tokens in the bank.
    #[state]
    pub(crate) tokens: sov_state::StateMap<C::Address, Token<C>>,
}
```

乍一看，由于通用 C ，这个定义可能看起来有点吓人。别担心，我们稍后会详细解释该泛型。现在，只需注意模块是一个带有地址和一些 #[state] 字段的结构体，这些字段指定该模块可以访问哪种数据。在幕后， ModuleInfo 派生宏将发挥一些作用，以确保任何 #[state] 字段都映射到唯一的存储键，以便只有该特定模块才能读取或写入其状态值。

在此阶段，请注意状态值位于模块外部，这一点也非常重要。此结构定义定义了将存储的值的形状，但值本身并不存在于模块结构内。换句话说，模块不会秘密地引用某些底层数据库。相反，模块定义了用于访问状态值的逻辑，并且这些值本身位于一个名为 WorkingSet 的特殊结构中。

这会产生几个后果。首先，这意味着模块的克隆成本总是很低。其次，这意味着调用 my_module.clone() 始终会产生与调用 MyModule::new() 相同的结果。最后，这意味着模块中读取或修改状态的每个方法都需要使用 WorkingSet 作为参数。

### 气体配置

该模块可能包含气体配置字段。如果在派生 ModuleInfo 的结构下使用 #[gas] 进行注释，它将尝试从项目的根目录读取 constants.json 文件，并将其注入到 < b3> 模块的实现。

以下是 constants.json 文件示例：

```json
{
    "gas": {
        "create_token": 4,
        "transfer": 5,
        "burn": 2,
        "mint": 2,
        "freeze": 1
    }
}
```

ModuleInfo 宏将在 JSON 中查找 gas 字段（该字段必须是一个对象），并将在 gas 中查找模块的名称目的。如果存在，它将将该对象解析为气体配置；否则，它将直接解析 gas 对象。在上面的示例中，它将尝试解析如下所示的结构：

```rust
#[derive(Debug, Clone, Copy, PartialEq, Eq, PartialOrd, Ord, Hash)]
pub struct BankGasConfig<GU: GasUnit> {
    pub create_token: GU,
    pub transfer: GU,
    pub burn: GU,
    pub mint: GU,
    pub freeze: GU,
}
```
GasUnit 泛型类型将由运行时 Context 定义。对于 DefaultContext ，我们使用 TupleGasUnit<2> - 即具有二维的气体单位。为 ZkDefaultContext 定义了相同的设置。以下是特定于 Bank 模块的 constants.json 文件示例：

```json
{
    "gas": {
        "comment": "this field will be ignored, as there is a matching module field",
        "Bank": {
            "create_token": [4, 19],
            "transfer": [5, 25],
            "burn": [2, 7],
            "mint": [2, 6],
            "freeze": [1, 4]
        }
    }
}
```
正如您在上面看到的，字段可以是数组、数字或布尔值。如果是布尔值，它将转换为 0 或 1 。如果是数组，则每个元素应为数字或布尔值。上面的例子将创建一个二维的气体单元。如果 Context 需要的维度少于可用维度，它将选择第一个相关的维度，并忽略其余维度。也就是说：对于一维的 Context ，有效配置将扩展为：


```
BankGasConfig {
    create_token: [4],
    transfer: [5],
    burn: [2],
    mint: [2],
    freeze: [1],
}
```

为了从工作组中充入气体，可以使用函数 charge_gas 。

```rust
fn call(
    &self,
    msg: Self::CallMessage,
    context: &Self::Context,
    working_set: &mut WorkingSet<C>,
) -> Result<sov_modules_api::CallResponse, Error> {
    match msg {
        call::CallMessage::CreateToken {
            salt,
            token_name,
            initial_balance,
            minter_address,
            authorized_minters,
        } => {
            self.charge_gas(working_set, &self.gas.create_token)?;
            // Implementation elided...
}
```

在上面的示例中，我们从工作集中对配置的单元进行收费。具体来说，我们将从 DefaultContext 和 ZkDefaultContext 中收取 [4, 19] 单位的费用。工作集将负责执行从维度到单个资金值的标量转换。它将执行加载价格与提供的单位的内积。

假设我们有一个加载价格 [3, 2] 的工作集。对于单维上下文，上述操作的充入气体将为 [3] · [4] = 3 × 4 = 12 ，对于 DefaultContext 和 ZkDefaultContext 均为 [3, 2] · [4, 19] = 3 × 4 + 2 × 19 = 50 。这种方法旨在解锁动态定价。

前面提到的 Bank 结构体，带有气体配置，将如下所示：

```rust
#[derive(ModuleInfo)]
pub struct Bank<C: sov_modules_api::Context> {
    /// The address of the bank module.
    #[address]
    pub(crate) address: C::Address,

    /// The gas configuration of the sov-bank module.
    #[gas]
    pub(crate) gas: BankGasConfig<C::GasUnit>,

    /// A mapping of addresses to tokens in the bank.
    #[state]
    pub(crate) tokens: sov_state::StateMap<C::Address, Token<C>>,
}
```

### 公共函数：模块到模块的接口

模块公开的第一个接口是由汇总的 impl 中的公共方法定义的。这些方法可以被其他模块访问，但不能被其他用户直接调用。 bank.transfer_from 方法就是一个很好的例子：

```rust
impl<C: Context> Bank<C> {
    pub fn transfer_from(&self, from: &C::Address, to: &C::Address, coins: Coins, working_set: &mut WorkingSet<C>) {
        // Implementation elided...
    }
}
```

此功能无需签名检查即可将代币从一个地址转移到另一个地址。如果它暴露给用户，就会导致资金被盗。但对于模块来说，能够在无需访问用户私钥的情况下发起资金转移是非常有用的。 （当然，模块在转账之前应该小心地获得用户的同意。通过使用transfer_from接口，模块声明它已经获得了这样的同意。）

这引出了关于模块系统的一个非常重要的观点。所有模块都是可信的。与以太坊上的智能合约不同，模块不能由用户动态部署 - 它们由汇总开发人员预先修复。这并不意味着 Sovereign SDK 不支持智能合约 - 只是它们位于堆栈的更高一层。如果您想在汇总上部署智能合约，则需要合并一个模块来实现安全虚拟机，用户可以调用该虚拟机来存储和运行智能合约。

### Call 功能：模块到用户界面

模块公开的第二个接口是来自 Module 特征的 call 函数。 call 函数定义用户可通过链上交易访问的接口，它通常采用枚举作为其第一个参数。该参数告诉 call 函数要调用模块的哪个内部方法。因此 call 的典型实现如下所示：

```rust
impl<C: sov_modules_api::Context> sov_modules_api::Module for Bank<C> {
	// Several definitions elided here ...
    fn call(&self, msg: Self::CallMessage, context: &Self::Context, working_set: &mut WorkingSet<C>) {
        match msg {
            CallMessage::CreateToken {
                token_name,
                minter_address,
            } => Ok(self.create_token(token_name, minter_address, context, working_set)?),
            CallMessage::Transfer { to, coins } => { Ok(self.transfer(to, coins, context, working_set)?) },
            CallMessage::Burn { coins } => Ok(self.burn(coins, context, working_set)?),
        }
    }
}
```

### RPC 宏：节点到用户界面

模块公开的第三个接口是 rpc 实现。要生成 RPC 实现，只需使用 sov_modules_api::macros 中的 #[rpc_gen] 宏注释您的 impl 块。

```rust
#[rpc_gen(client, server, namespace = "bank")]
impl<C: sov_modules_api::Context> Bank<C> {
    #[rpc_method(name = "balanceOf")]
    pub(crate) fn balance_of(
        &self,
        user_address: C::Address,
        token_address: C::Address,
        working_set: &mut WorkingSet<C>,
    ) -> RpcResult<BalanceResponse> {
        Ok(BalanceResponse {
            amount: self.get_balance_of(user_address, token_address, working_set),
        })
    }
}
```

这将在银行箱中生成一个名为 BankRpcImpl 的公共特征，它了解如何使用以下形式服务请求：

```rust
{
  "jsonrpc": "2.0",
  "id": 2,
  "method": "bank_balanceOf",
  "params": { "user_address": "SOME_ADDRESS", "token_address": "SOME_ADDRESS" }
}
```

有关如何将生成的特征实例化为绑定到特定端口的服务器的示例，请参阅 demo-rollup 包。

请注意，每个模块只能使用 rpc_gen 注释一个 impl 块，但该块可以包含任意数量的 rpc_method 注释。

有关显示如何使用模块系统实现 RPC 服务器的端到端演练，请参阅此处

### 背景和规格：如何使您的模块系统可移植

除了 Module 之外，模块系统中还有两个普遍存在的特征 - Context 和 Spec 。要理解这两个特征，请记住 Sovereign SDK rollup 的高级工作流程由两个阶段组成。首先，交易在本机代码中执行以生成“见证人”。然后，见证人被馈送到 zk 电路，该电路在（更昂贵的）zk 环境中重新执行交易以创建证明。因此，汇总工作流程的伪代码大致如下所示：

```rust
use sov_modules_api::DefaultContext;
fn main() {
    // First, execute transactions natively to generate a witness for the zkVM
    let native_rollup_instance = my_state_transition::<DefaultContext>::new(config);
    let witness = Default::default();
    native_rollup_instance.begin_slot(witness);
    for batch in batches.cloned() {
        native_rollup_instance.apply_batch(batch);
    }
    let (_new_state_root, populated_witness) = native_rollup_instance.end_batch();

    // Then, re-execute the state transitions in the zkVM using the witness
    let proof = MyZkvm::prove(|| {
        let zk_rollup_instance = my_state_transition::<ZkDefaultContext>::new(config);
        zk_rollup_instance.begin_slot(populated_witness);
        for batch in batches {
            zk_rollup_instance.apply(batch);
        }
        let (new_state_root, _) = zk_rollup_instance.end_batch();
        MyZkvm::commit(new_state_root)
    });
}
```

本机执行和零知识重新执行之间的区别已深深融入到模块系统中。我们的理念是，无论您使用哪种“模式”，您的业务逻辑都应该是相同的，因此我们抽象了一些特征背后的 zk 和本机模式之间的差异。

### 使用特征来定制不同模式下的行为

我们用来实现这种抽象的最重要的特征是 Spec 特征。 （简化的） Spec 定义如下：

```rust
pub trait Spec {
    type Address;
    type Storage;
    type PrivateKey;
    type PublicKey;
    type Hasher;
    type Signature;
    type Witness;
}
```

正如您所看到的，汇总的 Spec 指定将用于多种加密操作的具体类型。这样，您就可以根据抽象密码学定义业务逻辑，然后使用密码学对其进行实例化，这对于您选择的 zkVM 来说非常高效。

除了 Spec 特征之外，模块系统还提供了一个简单的 Context 特征，其定义如下：

```rust
pub trait Context: Spec + Clone + Debug + PartialEq {
    /// Sender of the transaction.
    fn sender(&self) -> &Self::Address;
    /// Constructor for the Context.
    fn new(sender: Self::Address) -> Self;
}
```

模块预计对于 Context 类型是通用的。如果模块在多个类型参数上是通用的，则绑定在 Context 上的类型始终位于这些类型参数中的第一个上。 Context 特征为它们提供了一个方便的句柄来访问 Spec 定义的所有加密操作，同时也使模块系统可以轻松传递经过身份验证的交易特定信息，这些信息否则将不可用于模块。目前， Context 只需要包含交易的 sender （签名者），但此特征将来可能会扩展。

综上所述，请记住 Bank 结构体是这样定义的。

```rust
pub struct Bank<C: sov_modules_api::Context> {
    /// The address of the bank module.
    pub(crate) address: C::Address,

    /// A mapping of addresses to tokens in the bank.
    pub(crate) tokens: sov_state::StateMap<C::Address, Token<C>>,
}
```

请注意，需要泛型类型 C 来实现 sov_modules_api::Context 特征。由于该泛型，Bank 结构可以从 Spec 访问 Address 字段 - 这意味着如果您更换底层地址架构，您的银行逻辑不会改变。

类似地，由于每个银行辅助函数在上下文中自动通用，因此很容易定义可以抽象出 zk 和 native 执行之间区别的逻辑。例如，当 rollup 在本机模式下运行时，其 Storage 类型几乎肯定是 ProverStorage ，它将其数据保存在 RocksDB 支持的 Merkle 树中。但如果您在 zk 模式下运行， Storage 类型将改为 ZkStorage ，它从证明者提供的一组“提示”中读取数据。因为所有汇总模块都是通用的，所以它们都不需要担心这种区别。

有关 Context 和 Spec 的更多信息，以及查看一些示例实现，请查看 sov_modules_api 文档。
