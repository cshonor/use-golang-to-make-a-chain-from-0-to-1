# Chain 结构体详解

> 第五章子文档：比原链 Chain 结构体详解

---

## 5.3 区块链的数据结构

**区块链按时间顺序连接区块，形成按时间排序的链表结构。** 本节将详细讲解区块链的数据结构，以及区块上链、连接、重组等操作的原理。

### 5.3.1 Chain 结构体定义

**比原链的链上操作主要通过 `Chain` 结构体实现。**

**代码位置：** `protocol/protocol.go`

**Go 结构体定义：**
```go
type Chain struct {
    index           *state.BlockIndex
    orphanManage    *OrphanManage
    txPool          *TxPool
    store           Store
    processBlockCh  chan *processBlockMsg
    cond            sync.Cond
    bestNode        *state.BlockNode
}
```

### 5.3.2 Chain 结构体字段详解

**`Chain` 结构说明如下：**

| 字段 | 类型 | 作用 |
|------|------|------|
| `index` | `*state.BlockIndex` | 存储在内存中的区块索引，该索引是一个树形结构，可以快速查找区块 |
| `orphanManage` | `*OrphanManage` | 孤块管理器，处理暂时找不到父区块的合法区块 |
| `txPool` | `*TxPool` | 交易池，存放未打包的交易 |
| `store` | `Store` | 区块持久化存储接口（如 LevelDB） |
| `processBlockCh` | `chan *processBlockMsg` | 待验证区块的缓冲队列，用于异步处理区块 |
| `cond` | `sync.Cond` | Go 语言 `sync` 包中的 `Cond` 结构，用于 goroutine 之间的并发控制 |
| `bestNode` | `*state.BlockNode` | 记录当前主链的最高区块（即主链的"尾巴"） |

### 5.3.3 各字段的详细说明

#### 1. index（区块索引）

**作用：**
- 存储在内存中的区块索引
- 该索引是一个**树形结构**，可以快速查找区块

**特点：**
- 树形结构：支持快速查找和遍历
- 内存存储：访问速度快
- 支持按高度、按哈希查找区块

**使用场景：**
- 快速查找指定高度的区块
- 快速查找指定哈希的区块
- 判断区块是否在主链上

#### 2. orphanManage（孤块管理器）

**作用：**
- 管理暂时找不到父区块的合法区块
- 当父区块到达后，自动将孤块接入主链

**核心功能：**
- 接收暂时找不到父区块的区块，存入孤块池
- 当父区块同步到本地后，自动把孤块从池里取出，尝试接入主链
- 定期清理超时的无效孤块，避免资源浪费

**相关文档：** 详见 [难度计算和孤块管理](./难度计算和孤块管理.md)

#### 3. txPool（交易池）

**作用：**
- 存放未打包的交易
- 供矿工选择交易进行打包

**特点：**
- 每个节点都有自己的交易池
- 内容大致相同，但可能不完全一样
- 交易按手续费排序，优先打包手续费高的

**相关文档：** 详见 [区块大小和交易打包](./区块大小和交易打包.md)

#### 4. store（区块持久化存储）

**作用：**
- 区块持久化存储接口
- 通常使用 LevelDB 等键值数据库

**特点：**
- 持久化存储：区块数据写入磁盘
- 键值存储：用区块哈希作为 key，区块数据作为 value
- 支持快速查询和写入

#### 5. processBlockCh（区块处理通道）

**作用：**
- 待验证区块的缓冲队列
- 用于异步处理区块

**特点：**
- Go 语言的 channel，用于 goroutine 之间的通信
- 缓冲队列：可以暂存多个待处理的区块
- 异步处理：不阻塞主流程

#### 6. cond（并发控制）

**作用：**
- Go 语言 `sync` 包中的 `Cond` 结构
- 用于 goroutine 之间的并发控制

**特点：**
- 条件变量：用于协调多个 goroutine 的执行
- 线程安全：保证并发访问的安全性
- 支持等待和唤醒机制

#### 7. bestNode（主链最高区块）

**作用：**
- 记录当前主链的最高区块
- 即主链的"尾巴"

**特点：**
- 指向主链上高度最高的区块节点
- 用于快速获取主链信息
- 链重组时会更新这个指针

---

## 5.3.4 Chain 的核心方法

**链上的操作方法是整个比原系统中最重要的一部分，很多重要的方法都在这里实现了，例如交易的验证、区块连接、区块校验等。**

### 主要方法列表

#### 1. 获取主链信息的方法

**`func (c *Chain) BestBlockHeight()`**
- **作用：** 获取主链的高度
- **返回：** 当前主链的最高区块高度

**`func (c *Chain) BestBlockHash()`**
- **作用：** 获取主链上目前高度最高的区块哈希
- **返回：** 当前主链最高区块的哈希值

**`func (c *Chain) BestBlockHeader()`**
- **作用：** 获取主链上目前高度最高的区块头
- **返回：** 当前主链最高区块的区块头

#### 2. 区块查询方法

**`func (c *Chain) InMainChain()`**
- **作用：** 判断区块是否在主链上
- **参数：** 区块哈希或区块高度
- **返回：** 布尔值，表示区块是否在主链上

#### 3. 共识算法相关方法

**`func (c *Chain) CalcNextSeed()`**
- **作用：** 计算 Tensority 算法的 seed
- **说明：** Tensority 是比原链使用的共识算法，需要计算 seed 值

**`func (c *Chain) CalcNextBits()`**
- **作用：** 计算下一个区块的难度值（Bits）
- **说明：** 根据前一个区块的信息，按协议规则算出下一个区块的难度

**相关文档：** 详见 [难度计算和孤块管理](./难度计算和孤块管理.md)

#### 4. 区块处理核心方法

**`func (c *Chain) ProcessBlock(block *types.Block)`**
- **作用：** 处理区块，完成验证后尝试接入主链
- **说明：** 这是区块上链的核心方法，包含完整的验证流程

**相关文档：** 详见 [processBlock流程详解](./processBlock流程详解.md)

**`func (c *Chain) ValidateTx(tx *types.Tx)`**
- **作用：** 验证单笔交易的合法性
- **说明：** 检查交易的签名、余额、格式等

#### 5. 区块存储方法

**`func (c *Chain) SaveBlock(block *types.Block)`**
- **作用：** 保存区块到本地存储
- **说明：** 将验证通过的区块持久化到数据库

#### 6. 其他重要方法

**`func (c *Chain) GetTransactionStatus()`**
- **作用：** 获取区块中所有交易的状态
- **说明：** 返回区块内所有交易的执行状态

**`func (c *Chain) GetTransactionsUtxo()`**
- **作用：** 获取区块中所有交易引用的 UTXO
- **说明：** 返回区块内所有交易引用的未花费交易输出

---

## 5.3.8 区块上链（Block on Chain）

**区块在完成一系列的验证步骤后，就会被存储到链上。本节将介绍区块是如何存储到链上的。**

### 5.3.8.1 saveBlock 方法

**代码位置：** `protocol/block.go`

**方法签名：**
```go
func (c *Chain) saveBlock(block *types.Block) error
```

**作用：** 将验证通过的区块存储到链上

**`saveBlock` 方法主要通过以下几个步骤将区块上链：**

### 步骤 1：将区块存储到本地 LevelDB 数据库中

**在 `Chain` 的 `saveBlock` 方法中，在完成区块的校验后，会通过 LevelDB 提供的 `SaveBlock` 接口将区块存储到本地 LevelDB 数据库中。**

**代码位置：** `database/leveldb/store.go`

**实现代码：**
```go
func (s *Store) SaveBlock(block *types.Block, ts *bc.TransactionStatus) error {
    binaryBlock, err := block.MarshalText()
    // ... 错误处理
    
    binaryBlockHeader, err := block.BlockHeader.MarshalText()
    // ... 错误处理
    
    binaryTxStatus, err := proto.Marshal(ts)
    // ... 错误处理
    
    blockHash := block.Hash()
    batch := s.db.NewBatch()
    
    // 存储完整区块
    batch.Set(calcBlockKey(&blockHash), binaryBlock)
    
    // 存储区块头
    batch.Set(calcBlockHeaderKey(block.Height, &blockHash), binaryBlockHeader)
    
    // 存储交易状态
    batch.Set(calcTxStatusKey(&blockHash), binaryTxStatus)
    
    batch.Write()
    
    return nil
}
```

**存储的三种信息：**

| 存储内容 | Key 格式 | 说明 |
|---------|---------|------|
| **完整区块** | `"B:" + 区块哈希` | 区块的完整序列化信息 |
| **区块头** | `"BH" + 区块高度 + 区块哈希` | 区块头的序列化信息 |
| **交易状态** | `"BTS:" + 区块哈希` | 区块中所有交易状态的序列化信息 |

**关键理解：**
- 使用 LevelDB 的批量写入（Batch）提高性能
- 三种信息分别存储，便于快速查询
- Key 的设计支持按哈希和按高度查询

### 步骤 2：从孤块池中删除已存储的区块

**代码：**
```go
c.orphanManage.Delete(&bcBlock.ID)
```

**作用：**
- 如果这个区块之前在孤块池中，现在已经成功上链
- 需要从孤块池中删除，避免重复处理

**关键理解：**
- 区块上链后，就不再是孤块了
- 需要清理孤块池，释放内存

### 步骤 3：将区块更新到内存中的区块索引上

**作用：**
- 将区块信息更新到内存中的区块索引（`index`）
- 索引是树形结构，支持快速查找

**代码实现：**
```go
node, err := state.NewBlockNode(&block.BlockHeader, parent)
if err != nil {
    return err
}
c.index.AddNode(node)
```

**关键理解：**
- 先创建 `BlockNode` 节点对象
- 然后将节点添加到索引中
- 内存索引用于快速查询
- 数据库用于持久化存储
- 两者配合，提供高性能的区块访问

### 步骤 4：更新 bestNode（主链最高区块）

**作用：**
- 如果新块接在主链上，更新 `bestNode` 指针
- 指向新的主链最高区块

**关键理解：**
- `bestNode` 始终指向主链的最高区块
- 链重组时会更新这个指针

---

## 5.3.9 BlockIndex 结构体详解

**`BlockIndex` 是比原链中用于管理区块索引的核心数据结构。**

### 5.3.9.1 BlockIndex 结构体定义

**代码位置：** `state/blockindex.go`

**Go 结构体定义：**
```go
type BlockIndex struct {
    sync.RWMutex
    index     map[bc.Hash]*BlockNode
    mainChain []*BlockNode
}
```

### 5.3.9.2 BlockIndex 字段详解

**`BlockIndex` 结构说明如下：**

| 字段 | 类型 | 作用 |
|------|------|------|
| `sync.RWMutex` | `sync.RWMutex` | 读写锁，用于并发控制，保证线程安全 |
| `index` | `map[bc.Hash]*BlockNode` | 区块索引，用区块哈希作为 key，快速查找区块 |
| `mainChain` | `[]*BlockNode` | 主链数组，存储未分叉的主链数据，按区块高度排序 |

### 5.3.9.3 各字段的详细说明

#### 1. sync.RWMutex（读写锁）

**作用：**
- Go 语言的读写锁，用于并发控制
- 保证多个 goroutine 同时访问 `BlockIndex` 时的线程安全

**特点：**
- 支持多个读操作同时进行
- 写操作时独占锁
- 提高并发性能

#### 2. index（区块索引 Map）

**作用：**
- 用区块哈希作为 key，快速查找区块
- 支持 O(1) 时间复杂度的区块查找

**数据结构：**
```go
index map[bc.Hash]*BlockNode
```

**特点：**
- Key：区块哈希（`bc.Hash`）
- Value：指向 `BlockNode` 的指针
- 支持快速查找：通过哈希值直接定位区块

**使用场景：**
- 根据区块哈希查找区块
- 判断区块是否存在
- 快速访问区块信息

#### 3. mainChain（主链数组）

**作用：**
- 存储未分叉的主链数据
- 所有区块数据按区块高度顺序排列
- 支持按高度快速检索区块

**数据结构：**
```go
mainChain []*BlockNode
```

**特点：**
- 数组结构：按高度顺序存储
- 只存储主链上的区块
- 支持按高度索引（`mainChain[height]`）

**使用场景：**
- 根据区块高度查找主链上的区块
- 遍历主链区块
- 获取主链信息

### 5.3.9.4 BlockIndex 的核心方法

#### AddNode 方法

**方法签名：**
```go
func (bi *BlockIndex) AddNode(node *BlockNode) {
    bi.Lock()
    bi.index[node.Hash] = node
    bi.Unlock()
}
```

**作用：**
- 将区块节点添加到索引中
- 更新 `index` map

**关键理解：**
- **只更新 `index` map，不更新 `mainChain`**
- 因为区块保存到链上时，它成为了链的一部分，但不一定是主链
- `mainChain` 的信息在链重组时才会更新

**流程：**
```
1. 加锁（保证线程安全）
   ↓
2. 将节点添加到 index map
   - key: node.Hash（区块哈希）
   - value: node（区块节点指针）
   ↓
3. 解锁
```

### 5.3.9.5 BlockIndex 的设计原理

**为什么需要两个数据结构？**

**1. index（Map）的作用：**
- 快速查找：通过哈希值 O(1) 查找区块
- 存储所有区块：包括主链和侧链上的所有区块
- 支持按哈希查询

**2. mainChain（数组）的作用：**
- 快速按高度查询：通过高度索引 O(1) 查找主链区块
- 只存储主链：不包含侧链区块
- 支持按高度遍历主链

**两者配合：**
- `index`：存储所有区块，支持按哈希查找
- `mainChain`：只存储主链，支持按高度查找
- 两者配合，提供高效的区块查询能力

### 5.3.9.6 区块上链时的索引更新

**当区块保存到链上时：**

```
1. 创建 BlockNode 节点
   node, err := state.NewBlockNode(&block.BlockHeader, parent)
   ↓
2. 添加到 index map
   c.index.AddNode(node)
   - 更新 index[node.Hash] = node
   - 不更新 mainChain
   ↓
3. 判断是否在主链上
   - 如果在主链上 → 更新 mainChain
   - 如果在侧链上 → 不更新 mainChain
```

**关键理解：**
- 区块上链时，**只更新 `index` map**
- `mainChain` 的更新需要判断区块是否在主链上
- 如果区块在侧链上，不会更新 `mainChain`

**为什么只更新 index，不更新 mainChain？**

**核心原因：**
- 当区块保存到链上时，它成为了链的一部分，但**不一定是主链**
- 区块可能在主链上，也可能在侧链上
- 只有确定区块在主链上时，才会更新 `mainChain`

**具体说明：**
- `index` map：存储**所有区块**（主链 + 侧链），所以区块上链时立即更新
- `mainChain` 数组：只存储**主链上的区块**，需要判断区块是否在主链上才更新
- 如果区块在侧链上，更新 `mainChain` 会导致数据不一致

### 5.3.9.7 链重组时的索引更新

**当发生链重组时：**

```
1. 检测到侧链比主链更长
   ↓
2. 触发链重组
   ↓
3. 更新 mainChain
   - 清空旧的 mainChain
   - 重新构建新的 mainChain（从创世区块到新的最高区块）
   ↓
4. index map 保持不变
   - 因为所有区块（主链+侧链）都在 index 中
```

**关键理解：**
- 链重组时，`mainChain` 会重新构建
- `index` map 保持不变，因为所有区块都需要保留
- 这样可以支持快速查询历史区块

---

## 5.3.11 链重组（Chain Reorganization）

**链重组是区块链中处理分叉和侧链的核心机制。**

### 5.3.11.1 链重组的概念

**节点将接收到的区块连接到 Chain 链上。**

**链重组发生的场景：**

1. **新共识规则发布后**
   - 没有升级的节点会因不知道新共识规则，而生产不合法的区块
   - 这便会产生临时性分叉

2. **临时性分叉**
   - 这种临时性分叉可以称为软分叉或侧链（这里的侧链是临时性产生的）
   - 当主链与侧链冲突时，侧链也有可能成为主链

3. **工作量证明决定主链**
   - 侧链成为主链取决于哪个链累积了最多的工作量证明
   - 如果侧链的工作量比主链的工作量还要多，则需要对区块链进行重组，将侧链变为主链

### 5.3.11.2 processBlock 中的链重组判断

**在 `processBlock` 方法中：**

```
如果添加区块的父区块不是主链上的尾节点（bestNode）
    ↓
则将这个区块添加到侧链上
    ↓
此时，需要判断主链与侧链的工作量哪个更多
    ↓
如果侧链的工作量比主链的工作量还要多
    ↓
则需要对区块链进行重组，将侧链变为主链
```

**关键理解：**
- 不是所有侧链都会成为主链
- 只有工作量更多的侧链才会成为主链
- 链重组是自动的，由工作量证明决定

### 5.3.11.3 工作量计算（CalcWork）

**在判断区块链工作量时，需要计算区块链节点工作量总和。**

**代码位置：** `protocol/block.go` 或相关文件

**Go 函数实现：**
```go
func CalcWork(bits uint64) *big.Int {
    difficultyNum := CompactToBig(bits)
    if difficultyNum.Sign() <= 0 {
        return big.NewInt(0)
    }
    denominator := new(big.Int).Add(difficultyNum, bigOne)
    return new(big.Int).Div(oneLsh256, denominator)
}
```

### 5.3.11.4 CalcWork 函数详解

**函数签名：**
```go
func CalcWork(bits uint64) *big.Int
```

**参数：**
- `bits`：区块头中的难度值（uint64）

**返回值：**
- `*big.Int`：该区块的工作量值

**函数逻辑：**

**1. 转换难度值**
```go
difficultyNum := CompactToBig(bits)
```
- 将紧凑格式的难度值（bits）转换为大整数
- `CompactToBig` 函数将 bits 转换为目标难度值

**2. 检查难度值有效性**
```go
if difficultyNum.Sign() <= 0 {
    return big.NewInt(0)
}
```
- 如果难度值为 0 或负数，返回 0
- 处理边界情况，防止无效计算

**3. 计算分母**
```go
denominator := new(big.Int).Add(difficultyNum, bigOne)
```
- 分母 = 难度值 + 1
- `bigOne` 是预定义的大整数常量，值为 1

**4. 计算工作量**
```go
return new(big.Int).Div(oneLsh256, denominator)
```
- 工作量 = 2^256 / (难度值 + 1)
- `oneLsh256` 是 1 左移 256 位，即 2^256
- 这是区块链中常用的工作量计算公式

### 5.3.11.5 工作量计算的原理

**为什么这样计算工作量？**

**核心思想：**
- 难度值越小 → 工作量越大
- 难度值越大 → 工作量越小

**公式解释：**
```
工作量 = 2^256 / (难度值 + 1)
```

**举例：**
- 如果难度值 = 1，工作量 = 2^256 / 2 ≈ 2^255（很大）
- 如果难度值 = 2^255，工作量 = 2^256 / 2^255 = 2（很小）

**关键理解：**
- 挖矿难度越高，单个区块的工作量越小
- 但整个链的工作量是所有区块工作量的累加
- 链越长、难度越高，总工作量越大

### 5.3.11.6 链重组的完整流程

**链重组的完整流程：**

```
1. 收到新区块
   ↓
2. 判断父区块是否在主链上
   - 如果在主链上 → 直接接在主链上
   - 如果不在主链上 → 添加到侧链
   ↓
3. 如果添加到侧链，计算工作量
   - 计算主链的总工作量（WorkSum）
   - 计算侧链的总工作量（WorkSum）
   ↓
4. 比较工作量
   - 如果侧链工作量 > 主链工作量 → 触发链重组
   - 否则 → 侧链保留，不重组
   ↓
5. 执行链重组（如果触发）
   - 清空旧的 mainChain
   - 将侧链变为新的主链
   - 重新构建 mainChain（从创世区块到新的最高区块）
   - 更新 bestNode 指针
   ↓
6. 链重组完成
```

**关键理解：**
- 链重组是自动的，由工作量证明决定
- 工作量更多的链会自动成为主链
- 保证区块链的安全性和一致性

### 5.3.11.7 链重组的影响

**链重组对系统的影响：**

1. **区块状态变化**
   - 原主链上的区块可能变成侧链区块
   - 侧链上的区块可能变成主链区块

2. **交易状态变化**
   - 原主链上的交易可能被回滚
   - 侧链上的交易可能被确认

3. **索引更新**
   - `mainChain` 数组需要重新构建
   - `index` map 保持不变（所有区块都保留）

4. **bestNode 更新**
   - `bestNode` 指针指向新的主链最高区块

**关键理解：**
- 链重组是区块链的正常机制
- 保证最长链（工作量最多的链）成为主链
- 维护区块链的安全性和一致性

---

## 5.3.10 区块上链的完整流程总结

**完整流程：**
```
1. 区块验证通过
   ↓
2. 调用 saveBlock 方法
   ↓
3. 存储到 LevelDB
   - 完整区块（key: "B:" + 区块哈希）
   - 区块头（key: "BH" + 区块高度 + 区块哈希）
   - 交易状态（key: "BTS:" + 区块哈希）
   ↓
4. 从孤块池删除（如果之前是孤块）
   c.orphanManage.Delete(&bcBlock.ID)
   ↓
5. 创建 BlockNode 节点
   node, err := state.NewBlockNode(&block.BlockHeader, parent)
   ↓
6. 更新内存索引
   c.index.AddNode(node)
   - 更新 index map
   - 不更新 mainChain（需要判断是否在主链上）
   ↓
7. 判断是否在主链上
   - 如果在主链上 → 更新 mainChain 和 bestNode
   - 如果在侧链上 → 不更新 mainChain
   ↓
8. 区块成功上链
```

**关键理解：**
- 区块上链是持久化 + 索引更新的过程
- 需要同时更新数据库和内存索引
- `index` 和 `mainChain` 的更新时机不同
- 保证数据一致性和查询性能

---

## 5.3.5 Chain 结构体的作用

**`Chain` 是比原链内核层的核心，它整合了所有核心模块：**

### 1. 模块整合

**`Chain` 结构体将以下核心模块整合在一起：**
- **区块索引（index）**：快速查找区块
- **孤块管理（orphanManage）**：处理暂时没爹的区块
- **交易池（txPool）**：存放未打包的交易
- **存储接口（store）**：区块持久化
- **处理通道（processBlockCh）**：异步处理区块
- **并发控制（cond）**：协调 goroutine
- **主链指针（bestNode）**：记录主链最高区块

### 2. 核心功能

**`Chain` 提供了以下核心功能：**
- ✅ 区块上链操作
- ✅ 区块验证
- ✅ 交易验证
- ✅ 链重组
- ✅ 主链信息查询
- ✅ 难度计算

### 3. 在整个系统中的地位

**`Chain` 是比原链内核层的核心数据结构：**
- 所有区块操作都通过 `Chain` 进行
- 所有验证逻辑都在 `Chain` 的方法中实现
- 是连接各个模块的桥梁

---

## 5.3.6 与其他模块的关系

### Chain 与 Node 的关系

**在 `node.Node` 中：**
```go
type Node struct {
    chain *protocol.Chain
    // ... 其他字段
}
```

**`Chain` 是 `Node` 的一个子模块，`Node` 通过 `Chain` 来管理区块链。**

### Chain 与 API Service 的关系

**API Service 通过调用 `Chain` 的方法来：**
- 获取区块信息
- 验证交易
- 查询主链状态

**相关文档：** 详见 [API架构和RPC通信](../chapter02/API架构和RPC通信.md)

---

## 5.3.7 关键理解

**`Chain` 结构体的核心理解：**

1. **`Chain` 是区块链的核心容器**
   - 整合了所有核心模块
   - 提供了所有链上操作的方法

2. **`Chain` 的方法是最重要的**
   - 交易验证、区块连接、区块校验都在这里实现
   - 是整个比原系统中最重要的一部分

3. **`Chain` 的设计体现了模块化**
   - 每个字段负责一个功能模块
   - 通过方法协调各个模块的工作

---

**返回**: [区块与区块链详解](./区块与区块链详解.md)

