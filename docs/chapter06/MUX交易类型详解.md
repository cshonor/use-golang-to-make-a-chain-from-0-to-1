# MUX交易类型详解

> 比原链（Bytom）MUX（多路复用）交易类型的详细说明

---

## 概述

**MUX（Multi-plexer，多路复用）**是比原链中一种特殊的数据结构，专门用于处理**多资产链**（Multi-asset Chain）中的交易。与比特币等单资产链不同，比原链的交易输入和输出可以包含多种不同类型的资产，MUX结构就是为了简化这种多资产、多输入、多输出交易的验证和管理。

**核心概念：**
- MUX是比原链独有的数据结构
- 用于聚合和分发多种资产
- 简化多资产交易的验证
- 防止溢出攻击

---

## 为什么需要MUX？

### 单资产链 vs 多资产链

**比特币（单资产链）：**
- 所有输入和输出都是同一种资产（BTC）
- 验证简单：只需检查 `∑输入金额 = ∑输出金额 + 手续费`
- 输入和输出一一对应关系清晰

**比原链（多资产链）：**
- 输入和输出可以包含多种资产（BTM、GOLD、SILVER等）
- 验证复杂：需要按资产类型分别验证
- 输入和输出可能形成复杂的对应关系

### 多资产交易的复杂性

**场景1：不同资产类型的输入和输出**
```
输入：
  - Input1: 100 BTM
  - Input2: 50 GOLD

输出：
  - Output1: 80 BTM
  - Output2: 20 BTM (找零)
  - Output3: 30 GOLD
  - Output4: 20 GOLD (找零)
```

**场景2：同种资产类型的多个输入对应多个输出**
```
输入：
  - Input1: 100 BTM
  - Input2: 50 BTM

输出：
  - Output1: 80 BTM
  - Output2: 40 BTM
  - Output3: 30 BTM (找零)
```

### MUX的作用

MUX结构提供了一个**"交易池"**的概念：
1. **聚合所有输入**：按资产类型聚合所有输入
2. **统一验证**：与所有输出进行比较，防止不匹配
3. **简化验证**：验证单个输出时，无需重新累加输入，直接使用聚合的MUX输入
4. **防止溢出攻击**：检测恶意输入（如负数、超出上限的值）

---

## MUX数据结构

### Go语言定义

**文件位置：** `protocol/bc/bc.pb.go`

```go
type Mux struct {
    Sources             []*ValueSource      // 源值列表（聚合的输入）
    Program             *Program            // 控制程序
    WitnessDestinations []*ValueDestination // 见证目标列表（输出）
    WitnessArguments    [][]byte            // 见证参数（签名等）
}
```

### 字段详解

#### 1. Sources（源值列表）

**类型：** `[]*ValueSource`

**作用：** 聚合所有输入（Input）的资产信息

**ValueSource 结构：**
```go
type ValueSource struct {
    Ref      *Hash        // 引用的Entry ID（如Spend、Issuance等）
    Value    *AssetAmount // 资产金额（AssetId + Amount）
    Position uint64       // 位置索引
}
```

**说明：**
- 每个输入（Spend、Issuance、Coinbase等）都会创建一个ValueSource
- MUX将所有输入的ValueSource聚合在一起
- 按资产类型组织，便于验证

#### 2. Program（控制程序）

**类型：** `*Program`

**作用：** 控制MUX的行为和验证逻辑

**默认值：** 
```go
&bc.Program{
    VmVersion: 1,
    Code: []byte{byte(vm.OP_TRUE)}  // 总是返回true
}
```

**说明：**
- MUX的Program通常使用`OP_TRUE`，表示总是通过验证
- 实际的验证逻辑在输入和输出层面完成

#### 3. WitnessDestinations（见证目标列表）

**类型：** `[]*ValueDestination`

**作用：** 指向所有输出（Output）的目标

**ValueDestination 结构：**
```go
type ValueDestination struct {
    Ref      *Hash        // 引用的Entry ID（如OriginalOutput、Retirement等）
    Value    *AssetAmount // 资产金额（AssetId + Amount）
    Position uint64       // 位置索引
}
```

**说明：**
- 每个输出都会创建一个ValueDestination
- 连接MUX和输出，形成完整的资产流转路径

#### 4. WitnessArguments（见证参数）

**类型：** `[][]byte`

**作用：** 存储见证数据（如签名）

**说明：**
- 在MUX层面通常为空
- 签名验证在具体的输入（Spend、Issuance）层面完成

---

## MUX的工作原理

### 1. 交易映射过程（MapTx）

比原链使用`MapTx`函数将`TxData`转换为基于Entry的表示，MUX在这个过程中创建：

```go
func MapTx(txData *TxData) *bc.Tx {
    mh := newMapHelper(txData)
    mh.mapInputs()      // 1. 映射所有输入
    mh.initMux()        // 2. 初始化MUX
    mh.mapOutputs()     // 3. 映射所有输出
    return mh.generateTx()
}
```

### 2. 初始化MUX（initMux）

```go
func (mh *mapHelper) initMux() {
    // 创建MUX，聚合所有输入的ValueSource
    mh.mux = bc.NewMux(mh.muxSources, &bc.Program{
        VmVersion: 1,
        Code: []byte{byte(vm.OP_TRUE)},
    })
    muxID := mh.addEntry(mh.mux)

    // 将所有输入连接到MUX
    for _, spend := range mh.spends {
        spentOutput := mh.entryMap[*spend.SpentOutputId].(*bc.OriginalOutput)
        spend.SetDestination(&muxID, spentOutput.Source.Value, spend.Ordinal)
    }

    for _, issuance := range mh.issuances {
        issuance.SetDestination(&muxID, issuance.Value, issuance.Ordinal)
    }

    if mh.coinbase != nil {
        mh.coinbase.SetDestination(&muxID, mh.mux.Sources[0].Value, 0)
    }
}
```

**步骤说明：**
1. 创建MUX，包含所有输入的ValueSource
2. 获取MUX的ID
3. 将所有输入（Spend、Issuance、Coinbase）的目标设置为MUX

### 3. 映射输出（mapOutputs）

```go
func (mh *mapHelper) mapOutputs() {
    muxID := bc.EntryID(mh.mux)
    for i, out := range mh.txData.Outputs {
        // 创建从MUX到输出的ValueSource
        src := &bc.ValueSource{
            Ref:      &muxID,
            Value:    &out.AssetAmount,
            Position: uint64(i),
        }
        
        // 创建输出Entry（OriginalOutput、Retirement、VoteOutput）
        // ...
        
        // 将输出添加到MUX的WitnessDestinations
        mh.mux.WitnessDestinations = append(mh.mux.WitnessDestinations, &bc.ValueDestination{
            Value:    src.Value,
            Ref:      &resultID,
            Position: 0,
        })
    }
}
```

**步骤说明：**
1. 获取MUX的ID
2. 为每个输出创建从MUX引用的ValueSource
3. 创建输出Entry
4. 将输出添加到MUX的WitnessDestinations

---

## MUX的资产流转路径

### 完整流转图

```
输入层（Inputs）
    ↓
    ├── Spend ──────────┐
    ├── Issuance ───────┤
    ├── Coinbase ───────┤
    └── VetoInput ──────┤
                        ↓
                    MUX（聚合池）
                        ↓
    ├── OriginalOutput ←┘
    ├── Retirement ←────┘
    └── VoteOutput ←─────┘
输出层（Outputs）
```

### 详细说明

1. **输入阶段**：
   - 每个输入（Spend、Issuance等）创建一个Entry
   - 输入Entry的ValueSource被添加到MUX的Sources列表
   - 输入Entry的Destination指向MUX

2. **MUX聚合阶段**：
   - MUX聚合所有输入的资产
   - 按资产类型组织
   - 提供统一的验证接口

3. **输出阶段**：
   - 每个输出从MUX引用资产
   - 创建输出Entry（OriginalOutput、Retirement等）
   - 输出Entry的ValueSource引用MUX
   - 输出Entry被添加到MUX的WitnessDestinations

---

## MUX的验证机制

### 1. 防止溢出攻击

MUX在聚合输入时，会检查：
- **负数金额**：如果输入包含负数，聚合时会检测到
- **超出上限**：如果金额超出uint64上限，会溢出并被检测
- **资产类型匹配**：确保输入和输出的资产类型匹配

### 2. 资产守恒验证

对于每种资产类型，必须满足：
```
∑(MUX.Sources中该资产的金额) = ∑(MUX.WitnessDestinations中该资产的金额)
```

**示例：**
```
MUX.Sources:
  - Source1: 100 BTM
  - Source2: 50 BTM
  - Source3: 30 GOLD

MUX.WitnessDestinations:
  - Dest1: 80 BTM
  - Dest2: 70 BTM
  - Dest3: 30 GOLD

验证：
  BTM: 100 + 50 = 150 = 80 + 70 ✓
  GOLD: 30 = 30 ✓
```

### 3. 简化验证流程

**不使用MUX的验证（复杂）：**
```go
// 需要为每个输出重新累加输入
for each output {
    totalInput := 0
    for each input {
        if input.asset == output.asset {
            totalInput += input.amount
        }
    }
    validate(totalInput >= output.amount)
}
```

**使用MUX的验证（简单）：**
```go
// 直接使用MUX聚合的输入
muxInputs := mux.Sources  // 已按资产类型聚合
for each output {
    assetInput := findAssetInMux(muxInputs, output.asset)
    validate(assetInput >= output.amount)
}
```

---

## 实际应用示例

### 示例1：多资产转账

**交易场景：**
- 输入：100 BTM + 50 GOLD
- 输出：80 BTM（给Alice）+ 20 BTM（找零）+ 30 GOLD（给Bob）+ 20 GOLD（找零）

**MUX结构：**
```go
mux := &Mux{
    Sources: []*ValueSource{
        {Ref: &spend1ID, Value: &AssetAmount{AssetId: BTM, Amount: 100}},
        {Ref: &spend2ID, Value: &AssetAmount{AssetId: GOLD, Amount: 50}},
    },
    WitnessDestinations: []*ValueDestination{
        {Ref: &output1ID, Value: &AssetAmount{AssetId: BTM, Amount: 80}},
        {Ref: &output2ID, Value: &AssetAmount{AssetId: BTM, Amount: 20}},
        {Ref: &output3ID, Value: &AssetAmount{AssetId: GOLD, Amount: 30}},
        {Ref: &output4ID, Value: &AssetAmount{AssetId: GOLD, Amount: 20}},
    },
}
```

**验证：**
- BTM: 100 = 80 + 20 ✓
- GOLD: 50 = 30 + 20 ✓

### 示例2：资产发行和转账

**交易场景：**
- 输入：发行100 GOLD + 花费50 BTM
- 输出：80 BTM（给Alice）+ 20 BTM（找零）+ 100 GOLD（给Bob）

**MUX结构：**
```go
mux := &Mux{
    Sources: []*ValueSource{
        {Ref: &issuanceID, Value: &AssetAmount{AssetId: GOLD, Amount: 100}},
        {Ref: &spendID, Value: &AssetAmount{AssetId: BTM, Amount: 50}},
    },
    WitnessDestinations: []*ValueDestination{
        {Ref: &output1ID, Value: &AssetAmount{AssetId: BTM, Amount: 80}},
        {Ref: &output2ID, Value: &AssetAmount{AssetId: BTM, Amount: 20}},
        {Ref: &output3ID, Value: &AssetAmount{AssetId: GOLD, Amount: 100}},
    },
}
```

**注意：** 这个例子中BTM输出（80+20=100）大于输入（50），这是**错误的**！实际交易中需要额外的BTM输入来支付手续费和找零。

---

## MUX与其他Entry的关系

### Entry类型体系

```
Entry（接口）
├── Mux（多路复用）
├── Spend（花费输入）
├── Issuance（发行输入）
├── Coinbase（币基输入）
├── VetoInput（投票输入）
├── OriginalOutput（普通输出）
├── Retirement（销毁输出）
└── VoteOutput（投票输出）
```

### MUX在Entry图中的位置

```
                    ┌─────────┐
                    │  MUX    │
                    │ (聚合池) │
                    └────┬────┘
                         │
        ┌─────────────────┼─────────────────┐
        │                 │                 │
        ↓                 ↓                 ↓
   ┌────────┐       ┌──────────┐      ┌──────────┐
   │ Spend  │       │ Issuance │      │ Coinbase │
   └────────┘       └──────────┘      └──────────┘
        │                 │                 │
        └─────────────────┼─────────────────┘
                         │
                    ┌────┴────┐
                    │  MUX    │
                    └────┬────┘
                         │
        ┌─────────────────┼─────────────────┐
        │                 │                 │
        ↓                 ↓                 ↓
   ┌──────────┐    ┌──────────┐    ┌──────────┐
   │Original  │    │Retirement│    │VoteOutput│
   │ Output   │    │          │    │          │
   └──────────┘    └──────────┘    └──────────┘
```

---

## 关键特性总结

### 1. 多资产支持

- ✅ 支持交易中包含多种资产类型
- ✅ 按资产类型自动聚合和验证
- ✅ 简化多资产交易的复杂度

### 2. 验证简化

- ✅ 统一验证接口
- ✅ 避免重复计算
- ✅ 提高验证效率

### 3. 安全性

- ✅ 防止溢出攻击
- ✅ 确保资产守恒
- ✅ 检测恶意输入

### 4. 灵活性

- ✅ 支持任意数量的输入和输出
- ✅ 支持复杂的资产流转路径
- ✅ 适应各种交易场景

---

## 常见问题

### Q1: MUX是交易类型吗？

**A:** MUX不是一种交易类型，而是一种**数据结构**，用于在交易内部聚合和管理多资产。每笔交易都会创建一个MUX来管理其输入和输出。

### Q2: MUX在区块浏览器中能看到吗？

**A:** 通常看不到。MUX是底层数据结构，区块浏览器通常只显示用户友好的交易信息（输入、输出、金额等），不会直接显示MUX结构。

### Q3: MUX的Program为什么是OP_TRUE？

**A:** MUX的Program使用`OP_TRUE`表示总是通过验证，因为实际的验证逻辑在输入和输出层面完成。MUX主要起到聚合和路由的作用。

### Q4: 单资产交易也需要MUX吗？

**A:** 是的。即使交易只包含一种资产（如只有BTM），比原链也会创建MUX。这是为了保持架构的一致性，简化验证逻辑。

### Q5: MUX如何防止溢出攻击？

**A:** 
1. 在聚合输入时，检查每个输入的金额是否合法
2. 在验证时，确保输入总额等于输出总额
3. 使用uint64类型，防止整数溢出
4. 在验证层面检查资产守恒

---

## 相关资源

- [交易基础数据结构](./交易基础数据结构.md) - Tx、TxData 结构详解
- [交易Action数据结构详解](./交易Action数据结构详解.md) - Input/Output Action 类型详解
- [比原链交易类型对比](./比原链交易类型对比.md) - 交易类型对比
- [UTXO模型详解](./UTXO模型详解.md) - UTXO模型说明
- [比原链技术文档](https://github.com/Bytom/bytom)

---

**返回**: [第六章目录](./README.md)

