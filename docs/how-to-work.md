# Sovereign SDK 的工作原理

在我们的公告博客文章中，我们勾画了未来的愿景，其中创建 zk-rollup 就像启动 dapp 一样简单。这篇博文旨在解释我们将如何实现这一愿景。在我们深入讨论之前，先快速警告一下——这篇文章会有些技术性，但并不完全严谨。如果您想要 Sovereign SDK 的非技术介绍，请参阅此处。如果您想要完整的技术严谨性，您可以在 GitHub 上找到 Sovereign SDK 的草案规范和实现，或者在 Discord 上联系。好了，让我们开始吧！

## 为什么选择 ZK-Rollups？

在本节中，我们将尝试从第一原理解释为什么我们认为 zk-rollups 是区块链的“最终游戏”。

### 原子可组合性的代价

我们从这样的论点开始：以太坊的杀手级功能不是 EVM 本身，而是原子可组合性。任何智能合约都可以与任何其他智能合约交互，并且系统保证调用者不会陷入某种中间状态 - 即使调用堆栈上层出现问题。这极大地简化了编程模型，使以太坊成为对开发人员最友好的环境之一。但原子可组合性是有代价的。为了原子地执行调用，全节点需要实时访问调用堆栈中所有合约的相关状态。这意味着他们必须将所有相关状态存储在本地，并且需要能够非常快速地加载它。换句话说，全节点需要购买SSD，并且需要购买足够的SSD来存储整个链状态。


为了最大限度地提高可扩展性，Near 等系统试图放宽这些要求。 Near 不支持原子可组合性，只允许异步跨合约调用。这种设置允许更高的可扩展性，因为每个合约的状态只能存储在完整节点的专用子集上。但这对程序员来说代价很大——他们必须通过代码路径的组合爆炸进行推理。

### 两全其美

我们的愿景是这两个极端的综合。我们相信，没有一个完全原子的系统能够满足整个世界的需求。但与此同时，我们认识到原子性在开发简单且可组合的智能合约中发挥的核心作用。因此，我们寻求实现两全其美：一个优化链的生态系统，每个链都提供原子可组合性，并通过整个生态系统中的快速（但异步）跨链桥连接。

在单个链中，开发人员保留原子可组合性。合约设计可以相对简单，用户可以近乎实时地了解交易结果。但与此同时，没有一组完整节点负责存储和更新整个世界的状态。

### 构建汇总，而不是区块链

我们当然不是第一个沿着这些思路思考的人。据我们所知，第一个尝试创建集成多链系统的是Cosmos。就像我们的愿景一样，Cosmos 是一个自下而上的异构链生态系统，具有快速可靠的桥接能力。 Cosmos 的不足之处在于创建新链的能力。在一条链变得可用之前，它需要一个“验证器集”来为其运行提供有意义的经济影响。考虑到资本成本，在链有足够的使用量来创造可观的收入之前，吸引验证者设置到链上是极其困难的。但当然，没有用户愿意在不安全的链上拿自己的钱冒险。这就导致了先有鸡还是先有蛋的情况，并使推出新连锁店成为一项艰巨的任务。

此外，Cosmos 链在实现桥接方面面临着根本性的困难。他们的轻客户端桥（IBC）的维护成本很高，因为标头必须在链上处理。这会导致桥图稀疏，从而导致糟糕的用户体验。更糟糕的是，如果一条链遭受长距离重组或其验证器集执行无效的状态转换，资金可能会丢失。

我们对这些问题的解决方案很简单：与许多其他链共享验证器集。换句话说，不要构建区块链；而是构建区块链。构建汇总。

最后，这引出了我们的产品：Sovereign SDK。

## 什么是主权SDK？

Sovereign SDK 是一个用于构建经过异步验证的主权 zk-rollups 的工具包。这实在是太拗口了，所以让我们一点一点地看一下。

首先，异步证明交易意味着什么？简单来说，它只是意味着原始交易数据由排序器实时发布到 L1，然后生成证明。这与 StarkNet 等 Rollup 形成鲜明对比，在 StarkNet 中，在将任何数据发布到链上之前，都会在链外生成证明。异步证明的优点是它允许实时完成交易，从而提供类似于 Optimistic Rollup 的 Rollup 响应能力。

其次，rollup 的主权意味着什么？与“智能合约”汇总相比，这是最容易理解的。在智能合约汇总中，L2 状态只有在被 L1 上的智能合约接受时才是最终的（在轻客户端看来）。由于当今的 L1 链的吞吐量如此有限，智能合约汇总被迫相对不频繁地更新其规范的链上状态。另一方面，主权汇总的轻客户端负责自行决定是否应接受特定区块。这使得主权汇总能够更快地完成，因为它们不必限制更新频率来适应拥塞的 L1。

## 它是如何工作的？

Sovereign SDK 的工作原理是将 zk-rollup 的功能抽象为接口层次结构。在层次结构的每个级别，开发人员都可以自由使用 SDK 提供的预打包实现之一或从头开始构建自己的功能。通过将逻辑封装在定义良好的接口后面，我们能够提供可插入组件，在不牺牲灵活性的情况下创建简单的开发人员体验。

### 核心 API

在最高抽象级别，每个 Sovereign SDK 链都结合了三个不同的元素：

1. L1区块链，提供DA和共识
2. 状态转换函数（用 Rust 编写），实现链的“业务逻辑”
3. 零知识证明系统能够 (1) 递归和 (2) 运行 Rust 的子集

```rust
interface DaLayer {
  // Gets all transactions from a particular da layer block that are relevant to the rollup
  // Used only by full-nodes of the rollup
  function get_relevant_txs(header: DaHeader) -> Array<DaTxWithSender>
  // Gets all transactions from a particular da layer block that are relevant to the rollup,
  // along with a merkle proof (or similar) showing that each transaction really was included in the da layer block.
  // Depending on the DA layer, may need to include auxiliary information to show that no
  // relevant transactions were omitted.
  // Used by provers
  function get_relevant_txs_with_proof(header: DaHeader) -> (Array<DaTxWithSender>, DaMultiProof, CompletenessProof>
  // Verifies that a list of Da layer transactions provided by an untrusted prover is both
  // complete and correct.
  // Used by the "verifier" circuit in the zkVM
  function verify_relevant_tx_list(txs: Array<DaTxWithSender>, header: DaHeader, witness: DaMultiProof, completenessproof: CompletenessProof)
}

// The interface to a state transition function, inspired by Tendermint's ABCI
interface StateTransitionFunction {
	// Called once at rollup Genesis to set up the chain
  function init_chain(config: Config)
  // A slot is a DA layer block, and may contain 0 or more rollup blocks
  function begin_slot(slotnumber: u64)
  // Applies a batch of transactions to the current rollup state, returning a list of the
  // events from each transaction, or slashing the sequencer if the batch was malformed
  function apply_batch(batch: Array<Transaction>, sequencer: Bytes): Array<Array<Event>> | ConsensusSetUpdate
  // Process a zero-knowledge proof, rewarding (or punishing) the prover
  function apply_proof(proof: RollupProof): Array<ConsensusSetUpdate>
  // Commit changes after processing all rollup messages
  function end_slot(): StateRoot
}

interface ZkVM {
  // Runs some code, creating a proof of correct execution
  function run(f: Function): Proof
  // Verifies a proof, returning its public outputs on success
  function verify(p: Proof): (Result<Array<byte>>)
}
```

### 广义全节点

使用我们刚刚描述的核心接口，Sovereign SDK 将提供通用的全节点实现，能够在任何数据可用性层上运行几乎任何状态转换函数。一般来说，全节点的工作原理是将汇总功能封装在我们刚才描述的核心接口后面，并将汇总数据视为可以存储和传输的不透明字节字符串。为了让您了解完整节点将如何运行，这里粗略地说明了核心事件循环的外观。

```rust
// A pseudocode illustration of block execution. Plays fast and loose
// with types for the sake of brevity.
function run_next_da_block(self, prev_proof: Proof, db: Database) {
  // Each proof commits to the previous state as a public output. Use that state
  // to initialize the state transition processor
  let prev_state = deserialize(prev_proof.verify());
  let current_da_header = db.get_da_header(prev_state.slot_number + 1);
  let processor = stf::new(db, current_da_header, prev_state);

  // Fetch the relevant transactions from the DA layer
  let (da_txs, inclusion_proof, completeness_proof) = da::get_relevant_txs_with_proof(current_header);
  da::verify_relevant_tx_list(da_txs, inclusion_proof, completeness_proof)

  // Process the relevant DA transactions
  processor.begin_slot(prev_state.slot_number + 1)
  for da_tx in da_txs {
    let msg = parse_msg(da_tx);
    if msg.is_batch() {
      processor.apply_batch(msg)
    } else {
      processor.apply_proof(msg)
    }
  }
  processor.end_slot();
}
```

### 可插拔组件

由于全节点实现是完全通用的，开发人员可以轻松更换模块来改变其链的特性。例如，非常关心 calldata 成本的开发人员可能会使用 [Jupiter](https://github.com/Sovereign-Labs/Jupiter)，即我们与 Celestia 区块链的集成。另一方面，主要关心流动性获取的开发人员可能会在以太坊或 EigenDA 之上构建（我们也将为此提供一个可插入模块，但直到原型阶段之后才会开发）。


同样，开发人员可以自由选择自己的 zkVM - 只要它支持 no_std Rust 和一些 shim API。喜欢使用标准编译器工具链和 VM 接口的开发人员可能会选择 [Risc0](https://github.com/risc0/risc0)，而那些重视小证明大小的开发人员可能会选择另一个 VM，例如 [=nil；基础](https://nil.foundation/)。

### 默认模块

跳过抽象层，Sovereign SDK 还旨在简化创建状态转换函数的过程。为此，我们将提供一个受 Cosmos SDK 启发的可组合模块系统。尽管我们将随着时间的推移继续构建功能，但我们计划至少推出以下模块：存储、桥接、（可替代）令牌和证明/排序。

截至撰写本文时，模块系统的设计仍处于早期阶段 - 因此您在本节中阅读的所有内容都可能会发生快速变化。但是，我们仍然认为勾勒出我们一直在研究的一些设计是有用的，只是为了让您体验一下即将发生的事情。

#### 桥接模块

首先，桥接。与单独链之间的桥接不同（众所周知，如果没有强信任假设，[这是不可能的](https://spiral.imperial.ac.uk/bitstream/10044/1/75810/6/2019-1128.pdf)），共享 DA 层上的汇总之间的桥接从根本上来说并不困难。在 Sovereign SDK 中，我们计划充分利用这一事实，默认提供低延迟信任最小化桥接。

它的工作原理如下：假设您在共享 DA 层上运行了 50 个汇总。要在标准范例中在每个汇总之间运行信任最小化的桥梁，您需要每个汇总都运行每个其他汇总的链上轻客户端。计算一下，即 50 * 49 = 2450 个链上轻客户端。由于实际执行起来成本高昂，因此您会转而使用多跳桥。您可以让每个链连接到（比如说）另外 3 个链，并且假设您的网络具有合理的拓扑，您将能够使用 3 或 4 个跃点在任意一对链之间路由消息。

借助 zk 的力量，我们可以做得更好。回想一下，任何两个零知识证明（在 VM 模型中）都可以聚合成第三个证明，并且这个新证明的验证成本并不比原始证明更高。利用这种洞察力，我们可以在链外递归聚合所有 50 个汇总证明，然后验证每个汇总上的汇总证明。因此，我们可以进行 1 次链下证明聚合和 50 次链上证明验证，而不是维护 2450 个轻客户端连接。换句话说，我们将所有桥接的通信复杂性从 O(n^2) 降低到 O(n)。

#### 存储模块

高效 zk-rollups 的一个关键组成部分是状态管理。在设计经过身份验证的数据存储时，开发人员必须兼顾几个相互竞争的问题。一方面，他们想要能够有效地表示电路内的东西，因为每次对状态的读取和写入都必须在零知识证明中进行身份验证。另一方面，他们希望在本机执行过程中能够快速且轻量级，这样就不会成为全节点和排序器的瓶颈。

在Sovereign SDK中，我们计划提供一个默认的存储模块来平衡这两个设计目标。为了提高 zkVM 之外的效率，我们将使用最初由 Diem 开发的 [Jellyfish Merkle Tree (JMT)](https://developers.diem.com/papers/jellyfish-merkle-tree/2021-01-14.pdf) 来存储经过身份验证的状态数据。为了快速访问，我们还将维护一个平面磁盘数据结构来保存原始状态。为了提高电路内效率，我们将使 JMT 成为通用的哈希函数，以便开发人员可以轻松插入适合他们选择的 zkVM 的哈希。

## 将它们放在一起：块生命周期

将区块添加到主权链需要三个步骤。首先，定序器将新的交易数据块发布到 L1 链上。一旦 Blob 在 L1 上最终确定，新的逻辑汇总状态也将最终确定。

当每个 L1 块最终确定时，rollup 的完整节点会扫描它并识别与 rollup 的状态转换函数相关的 blob。然后，他们按顺序将汇总的 STF 应用于每个 Blob，计算新的汇总状态根。至此，区块从全节点的角度主观最终确定。

接下来，证明者节点（在 zkVM 内运行的完整节点）执行与完整节点相同的过程 - 扫描 DA 块并按顺序处理所有数据 blob。然而，与完整节点不同的是，证明者相互竞争以生成证明并将其发布到链上。第一个创建有效证明的合格节点将获得与汇总汽油费（部分）相对应的代币奖励。一旦给定批次的证明被发布到链上，该批次对于轻客户端来说就主观上是最终的。

最后，中继者可以选择向另一个链上的桥接智能合约提交最近的汇总证明 - 无论是底层 DA 层还是另一个汇总。为了提高效率，中继者可能会在提交更新之前使用证明聚合将许多汇总的有效性证明合并为单个证明。

## 结论

如果您已经完成了这一步，那么您现在应该对 Sovereign SDK 有一个很好的高级理解。如果您有兴趣了解更多信息，请深入我们的 github（欢迎贡献！）、在 Twitter 上联系或关注 Discord！另外，我们正在招聘！
