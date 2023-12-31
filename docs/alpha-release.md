# 引入 Sovereign SDK Alpha 版本

ZK-rollups 正在彻底改变区块链，但直到现在，开发人员部署它们一直是一个真正的挑战。这就是为什么我们很高兴推出 Sovereign SDK 的 alpha 版本，这是一个开发工具包，允许开发人员将他们的应用程序作为 zk-rollups 启动。

## 什么是 Sovereign SDK Alpha 版本？

这是一个 MVP 版本，专注于一些核心功能，让您感受到 Sovereign SDK 的开发人员体验。具体来说，我们构建了：

- 演示汇总和证明
- Celestia 和 Avail 的数据可用性层适配器（感谢 Avail 团队实现了他们的适配器！）*
- Risc0 的 ZKVM 适配器**
- 一个模块系统，使您能够组合现成的模块和自定义逻辑来将您的应用程序构建为 ZK-rollup
- 即用型银行、存储、账户和集中排序模块
- 用于读取应用程序状态的简单且可扩展的 RPC 接口
- 将应用程序状态保存到磁盘的数据库层，以及使用 Jellyfish Merkle Tree 的经过身份验证的状态存储
- 以及大量文档，让您尽可能轻松地构建自己的 rollup

请务必注意，Sovereign SDK 尚未经过审核，在任何情况下都不应在生产中使用。在 alpha 期间，我们将尽最大努力保持 API 的稳定性并遵守语义版本控制。

## 下一步是什么？

在接下来的几个月里，我们将全力以赴生产我们的客户端：修复已知错误，优化，改善开发人员体验，添加桥接和分散排序等缺失功能，并彻底测试代码库。

我们的目标是在一年内达到测试版（API 稳定）。

## 为什么选择主权 SDK？

一旦准备就绪，SDK 将启用具有以下功能的应用程序：

- 有保证且可扩展的吞吐量：Sovereign SDK 将确保可扩展、可验证且抗审查的后端与您的应用程序的增长保持同步，从而使您的用户免于支付过高的费用。

- 无需引导验证器：开发您的应用程序已经足够具有挑战性了。 Sovereign SDK 将消除说服验证者的需要，也无需担心验证者未能出现时的活跃度问题。

- 快速、廉价和无需信任的桥接：用户不必担心资金会卡在应用程序中或向流动性提供者支付不必要的转账费用。使用 Sovereign SDK，您将告别长达一周的退出期，同时安全地扩展您的应用程序。

- 完全可定制性：使用您的代币作为燃料代币或添加自定义交易处理逻辑。 Sovereign SDK 将使您能够完全自定义您的堆栈，让您可以自由构建无限的应用程序。

- 降低平台风险：很难在 DA 和 zk-proof 系统中选出赢家。值得庆幸的是，我们的不可知论设计将允许您使用任何兼容的 DA 层（包括以太坊！）和 zk-proof 系统部署汇总，从而降低风险。

查看 [Sovereign SDK](https://github.com/Sovereign-Labs/sovereign-sdk) 并开始构建您的 [zk-rollup 演示应用程序](https://airtable.com/app9kbNZcvqdrOidH/shrPlG9rok3sVd8Mh)！如果您想与我们一起集思广益，请填写我们的早期合作伙伴意向表。

## 汇总互联网

我们想象一个未来，可验证和抗审查的区块链应用程序并行运行、扩展和无缝通信。

我们的使命是让任何人都能构建 zk-rollup，而不仅仅是密码学专家或协议工程师。

- 助您开始使用 Avail 适配器的教程即将推出。
- Risc0 适配器目前受限，因为 Risc0 尚不支持递归。
