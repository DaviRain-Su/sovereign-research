# Sovereign SDK Research

Sovereign SDK是一个免费且开源的工具包，用于构建Rollups（包括ZK和乐观型），目前正在开发中。Sovereign SDK由三个逻辑组件组成：

1. Rollup接口，一个定义了rollup的最小接口集合
2. 模块系统是一个为使用Rollup接口构建rollup的有见解的框架
3. 全节点是一个客户端实现，能够运行任何实现了Rollup接口的Rollup。

## Rollup 接口

在Sovereign SDK的核心是[Rollup接口](https://github.com/Sovereign-Labs/sovereign-sdk/blob/nightly/rollup-interface/specs/overview.md)，它定义了Rollup必须实现的接口。在Sovereign SDK中，我们将Rollup定义为三个组件的组合：

1. 一个[状态转换函数（"STF"）](https://github.com/Sovereign-Labs/sovereign-sdk/blob/nightly/rollup-interface/specs/interfaces/stf.md)，它定义了Rollup的"业务逻辑"

2. 一个[数据可用性层](https://github.com/Sovereign-Labs/sovereign-sdk/blob/nightly/rollup-interface/specs/interfaces/da.md)（"DA层"），确定要传送到状态转换函数的交易集合

3. 一个零知识证明系统（也称为“零知识虚拟机”或“zkVM”），它接收编译后的Rollup代码并生成简洁的证明，证明逻辑已经正确执行。

Sovereign SDK的主要目标之一是在这三个组件之间实现清晰的关注点分离。大多数Rollup开发者不需要实现DA层接口，他们可以使用SDK编写逻辑，并与任何DA层兼容，因此在新链上部署他们的Rollup就像从货架上选择一个特定DA层的[适配器](https://github.com/Sovereign-Labs/Jupiter)一样简单。

同样地，构建DA层的团队不需要担心使用他们的链将构建哪种状态转换。他们只需要实现DA层接口，就可以自动与所有状态转换函数兼容。

Rollup接口的代码位于rollup-interface文件夹中。对于技术描述，我们建议在这里查看概述。如果您想要一个不太技术性的介绍，请参阅这篇博客文章。

## 模块系统

虽然Rollup接口定义了一组强大的抽象，但它对于状态转换函数的实际工作方式没有明确的意见。就接口而言，您的状态机可能与经典的“区块链”金融应用无关，因此它没有内置的状态、账户、代币等概念。这意味着单独的Rollup接口包无法提供“一揽子”开发体验。但我们在Sovereign的目标之一是使开发Rollup与部署智能合约一样简单。因此，我们构建了一套额外的工具来定义您的状态转换函数，称为模块系统。

在模块系统的核心是包 sov-modules-api 。该包定义了一组核心特征，这些特征表达了如何将在不同模块中实现的功能组合成一个 Runtime ，能够处理事务并提供RPC请求的能力。它还定义了用于实现大多数这些特征的宏。对于许多应用程序来说，使用模块系统定义状态转换函数应该就像从货架上挑选一些模块并定义一个将它们粘合在一起的结构体一样简单。为了提供这种体验，模块系统依赖于一组常用的类型和特征，这些类型和特征在每个模块中都被使用。 sov-modules-api crate定义了这些特征（如 Context 和 MerkleTreeSpec ）以及 Address 等类型。

除了模块API之外，我们还提供了一个由Jellyfish Merkle树支持的状态存储层，以及一系列有助于处理有状态交易的实用工具。最后，我们还提供了一组实现常见区块链功能的模块，例如 Accounts 和可替代的 Tokens 。

有关模块系统的更多信息，请参阅其自述文件。您还可以在此处找到有关实施和部署自定义模块的教程。

## The Full Node 全节点

这个仓库的最后一个组件是全节点，它是一个客户端实现，能够运行任何实现了Rollup接口的Rollup。全节点提供了一种简单的方式来部署和运行您的Rollup。在默认配置下，它可以自动将链数据存储在其数据库中，为链数据和应用程序状态提供RPC请求服务，并与DA层交互以同步其状态和发送交易。虽然全节点实现应与自定义状态转换函数兼容，但目前仅针对使用模块系统构建的Rollup进行了测试。如果您在运行全节点时遇到任何困难，请联系我们或提出问题！所有核心开发人员都可以通过Discord联系到。
