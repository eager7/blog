# Tendermint 的区块构成

## Tendermint 的区块构成

Tendermint 定义了区块的格式，任何基于 Tendermint 所开发的区块链，都会遵循此格式：

1. Header：区块头（下文详述）
2. Data：交易列表
3. Evidence：Validator 作恶的证据列表
4. LastCommit：上一个区块所获得的若干 Validator 的签名

其中，交易列表在 Tendermint 看来，就是无意义的字节串，上层应用负责解释和执行这些交易。

Evidence 和 LastCommit 是在 PoS 链中才会出现的数据。不同于 Bitcoin 用 PoW 的 nonce 来确认一个区块，PoS 链用 Validator 的投票来确认一个区块，大多数 PoS 共识算法，包括 Tendermint，要求 2/3 以上的 Validator 投票才能确认一个区块。LastCommit 即为 Validator 们对上一个区块所作出的投票。

为何是 “上一个区块” 而不是 “当前区块”？因为 PoS 链要求先由某一个 Validator 提出（propose）下一个区块的候选（candidate），然后对它进行广播，请其他 Validator 对此节点进行投票，投票通过后区块得到确认，才能提交（commit）到链上。新生成的区块在被广播的时候，尚未得到投票确认，因此无法把“当前区块” 的投票信息（即各个 Validator 的签名）包括在区块中。

对恶意的 Validator 进行惩罚（slash）是保证 PoS 链安全的必要手段，目前，Tendermint 只惩罚一种恶意行为——双签（还有一种对可用性差的惩罚，但那并不属于恶意）。Validator 必须抵押一笔资金才能获得投票确认区块的权力，当它在一个区块高度上对两个不同的区块都进行了签名时，就犯了 “双签” 的错误，它抵押的资金会被罚掉很大一个比例。当某个节点发现某 Validator 进行双签的证据时，它把证据进行全网广播，最终由某个出块的 Validator 将证据包含在区块中，这就是区块中的 Evidence。

在介绍区块头之前，我们先介绍一下 Tendermint 是如何保存全节点状态的。全节点的状态分两部分，一部分是上层应用的状态，一般会表现为一系列键值对的集合，Tendermint 并不关心其细节，只要它们可以构成 Merkle 树，最终得到一个 Merkle Root 即可（下文中我们将看到，这一 Merkle Root 被命名为 AppHash）；另一部分是共识引擎的状态，即由 Tendermint 自己维护的状态。Tendermint 会确保全网所有的全节点的状态都是一致的。

共识引擎的状态定义如下：

```text
type State struct {
    Version Version


    // immutable
    ChainID string


    // LastBlockHeight=0 at genesis (ie. block(H=0) does not exist)
    LastBlockHeight int64
    LastBlockTotalTx int64
    LastBlockID types.BlockID
    LastBlockTime time.Time


    // LastValidators is used to validate block.LastCommit.
    // Validators are persisted to the database separately every time they change,
    // so we can query for historical validator sets.
    // Note that if s.LastBlockHeight causes a valset change,
    // we set s.LastHeightValidatorsChanged = s.LastBlockHeight + 1 + 1
    // Extra +1 due to nextValSet delay.
    NextValidators *types.ValidatorSet
    Validators *types.ValidatorSet
    LastValidators *types.ValidatorSet
    LastHeightValidatorsChanged int64


    // Consensus parameters used for validating blocks.
    // Changes returned by EndBlock and updated after Commit.
    ConsensusParams types.ConsensusParams
    LastHeightConsensusParamsChanged int64


    // Merkle root of the results from executing prev block
    LastResultsHash []byte


    // the latest AppHash we've received from calling abci.Commit()
    AppHash []byte
}
type ConsensusParams struct {
    Block BlockParams `json:"block"`
    Evidence EvidenceParams `json:"evidence"`
    Validator ValidatorParams `json:"validator"`
}
type BlockParams struct {
    MaxBytes int64 `json:"max_bytes"`
    MaxGas int64 `json:"max_gas"`
    // Minimum time increment between consecutive blocks (in milliseconds)
    // Not exposed to the application.
    TimeIotaMs int64 `json:"time_iota_ms"`
}
type EvidenceParams struct {
    MaxAge int64 `json:"max_age"` // only accept new evidence more recent than this
}
type ValidatorParams struct {
    PubKeyTypes []string `json:"pub_key_types"`
}
```

Version 包括两个部分，一个是底层共识的版本号，一个是上层应用的版本号，二者都是 int64 类型。

ChainID 用来唯一标志一条链，当链进行硬分叉升级时，ChainID 需要被更换。

LastBlockHeight、LastBlockTotalTx、LastBlockID、LastBlockTime 分别表示上一个区块的高度、累积交易总数（从创世块到上一个区块总共执行了多少交易），ID 和时间戳。其中，BlockID 的定义如下：

```text
type BlockID struct {
    Hash cmn.HexBytes `json:"hash"`
    PartsHeader PartSetHeader `json:"parts"`
}
type PartSetHeader struct {
    Total int `json:"total"`
    Hash cmn.HexBytes `json:"hash"`
}
```

其中包含两个部分，一个是 Block 整体形成的 Merkle 树的根 Hash，另一个是各个 Part 的 Hash。之所以要这样设计 BlockID，是因为 Tendermint 借鉴了 LibSwift，把区块拆分成很多个 Part，每个 Part 各自在 P2P 网络上进行广播，其它全节点需要逐个接收和验证这些 Part，然后再把它们拼合起来。这样可以达到更快的区块传播速度。

Tendermint 依赖上层应用来决定 Validator 集合应该如何变动，而且每个区块都可以更改 Validator 集合。但是，当前区块对 Validator 集合所作出的修改，要隔一个区块，到下下个区块才能生效。因此，Tendermint 使用了三个变量来跟踪 Validator 集合的变动：NextValidators、Validators、LastValidators，它们有效的时间分别是下一个区块，当前正在投票决策的区块和上一个区块。比如执行高度为 100 的区块时，修改了 Validator 集合，那么要在决策高度为 102 的区块时才会按照更新后的 Validator 集合进行投票。当高度为 100 的区块被执行完毕后，新的 Validator 集合被保存在 NextValidators 变量中；当高度为 101 的区块被执行完毕后，这一集合被保存在 Validator 中；当高度为 101 的区块被执行完毕，这一集合被保存在 LastValidator 中，而且，对高度为 102 的区块投票，将来自于这个集合中的 Validator。LastHeightValidatorsChanged 表示上一个发生了 Validator 集合变动的区块的高度，在上述的例子中，此变量取值为 102。

ConsensusParams 包含一些和共识相关的参数。这些参数同样可以被上层逻辑所修改。LastHeightConsensusParamsChanged 表示上一次修改这些参数的区块高度。

上层应用在执行完一个交易后，向 Tendermint 引擎返回一些 Results。LastResultsHash 是上个区块执行过程中所返回的所有 Results 所形成的 Merkle 树的根 Hash。

上层应用在执行完上一区块之后的内部状态所构成 Merkle 树的根 Hash，被保存在 AppHash 中。

以上，我们详细介绍 State 这个结构体所包含的各个成员，有了这些知识，理解区块头的定义就比较容易了。区块头的定义如下：

```text
type Header struct {
    // basic block info
    Version version.Consensus `json:"version"`
    ChainID string `json:"chain_id"`
    Height int64 `json:"height"`
    Time time.Time `json:"time"`
    NumTxs int64 `json:"num_txs"`
    TotalTxs int64 `json:"total_txs"`


    // prev block info
    LastBlockID BlockID `json:"last_block_id"`


    // hashes of block data
    LastCommitHash cmn.HexBytes `json:"last_commit_hash"` // commit from validators from the last block
    DataHash cmn.HexBytes `json:"data_hash"` // transactions


    // hashes from the app output from the prev block
    ValidatorsHash cmn.HexBytes `json:"validators_hash"` // validators for the current block
    NextValidatorsHash cmn.HexBytes `json:"next_validators_hash"` // validators for the next block
    ConsensusHash cmn.HexBytes `json:"consensus_hash"` // consensus params for current block
    AppHash cmn.HexBytes `json:"app_hash"` // state after txs from the previous block
    LastResultsHash cmn.HexBytes `json:"last_results_hash"` // root hash of all results from the txs from the previous block


    // consensus info
    EvidenceHash cmn.HexBytes `json:"evidence_hash"` // evidence included in the block
    ProposerAddress Address `json:"proposer_address"` // original proposer of the block
}
```

其中，Version、ChainID、LastBlockID、AppHash 都直接拷贝自 State 这个结构体的成员。

有 7 个 Hash 值，是通过计算得来的。DataHash、EvidenceHash 和 LastCommitHash，分别从区块的 Data、Evidence 和 LastCommit 三部分数据计算得来，即数据构成 Merkle Tree 后的根 Hash。而 ValidatorsHash、NextValidatorsHash、ConsensusHash 和 LastResultsHash 分别由 State 结构体的 Validators、NextValidators、ConsensusParams 和 LastResults 计算得来。

剩下的几个成员的含义非常容易理解：Height 是区块高度，Time 是区块的时间戳，NumTxs 是区块中的交易数，TotalTxs 是累积的交易总数（TotalTxs=LastBlockTotalTx+NumTxs），ProposerAddress 是打包并且广播此 Block 的 Validator 的地址。

```text
Height int64 `json:"height"`
Time time.Time `json:"time"`
NumTxs int64 `json:"num_txs"`
TotalTxs int64 `json:"total_txs"`
```

剩下的几个成员的含义非常容易理解：Height 是区块高度，Time 是区块的时间戳，NumTxs 是区块中的交易数，TotalTxs 是累积的交易总数（TotalTxs=LastBlockTotalTx+NumTxs），ProposerAddress 是打包并且广播此 Block 的 Validator 的地址。

