# Annotated 结构详解

> 比原链 AnnotatedInput 和 AnnotatedOutput 增强结构详解

---

## 概述

`AnnotatedInput` 和 `AnnotatedOutput` 是用于查询API和区块浏览器的增强型结构，在基础 `TxInput` 和 `TxOutput` 的基础上添加了用户友好的信息，如资产别名、账户别名、人类可读的地址等。

**文件位置：** `blockchain/query/annotated.go`

---

## AnnotatedInput（增强型交易输入）

### 结构定义

```go
// AnnotatedInput 表示一个带注释的交易输入（用于查询和显示）
type AnnotatedInput struct {
    Type             string               `json:"type"`                  // 输入类型（"spend", "coinbase", "issuance"等）
    AssetID          bc.AssetID           `json:"asset_id"`              // 资产ID
    AssetAlias       string               `json:"asset_alias"`           // 资产别名（用户友好）
    AssetDefinition  *json.RawMessage     `json:"asset_definition"`      // 资产定义（JSON格式）
    Amount           uint64               `json:"amount"`               // 金额
    IssuanceProgram  chainjson.HexBytes   `json:"issuance_program"`      // 发行程序（仅发行输入）
    ControlProgram   chainjson.HexBytes   `json:"control_program"`       // 控制脚本（解锁脚本）
    Address          string               `json:"address"`               // 地址（从ControlProgram派生）
    SpentOutputID    *bc.Hash             `json:"spent_output_id"`      // 被花费的输出ID（UTXO ID）
    AccountID        string               `json:"account_id"`            // 账户ID
    AccountAlias     string               `json:"account_alias"`          // 账户别名（用户友好）
    Arbitrary        chainjson.HexBytes   `json:"arbitrary"`             // 任意数据（仅Coinbase输入）
    InputID          bc.Hash              `json:"input_id"`              // 输入唯一标识符
    WitnessArguments []chainjson.HexBytes `json:"witness_arguments"`    // 见证参数（签名等）
    SignData         bc.Hash              `json:"sign_data"`             // 签名数据
    Vote             string               `json:"vote"`                 // 投票信息（仅投票输入）
    StateData        []string             `json:"state_data"`            // 状态数据
}
```

### 字段说明

| 字段 | 类型 | 说明 |
|------|------|------|
| `Type` | string | 输入类型：`"spend"`（花费）、`"coinbase"`（币基）、`"issuance"`（发行）、`"veto"`（否决） |
| `AssetID` | bc.AssetID | 资产ID（被花费的资产） |
| `AssetAlias` | string | 资产别名，用户友好的名称（如"BTM"） |
| `AssetDefinition` | *json.RawMessage | 资产的完整定义（JSON格式） |
| `Amount` | uint64 | 输入金额（最小单位） |
| `IssuanceProgram` | HexBytes | 发行程序（仅当输入类型为发行时） |
| `ControlProgram` | HexBytes | 控制脚本（解锁脚本），十六进制格式 |
| `Address` | string | 地址（从ControlProgram派生的人类可读地址） |
| `SpentOutputID` | *bc.Hash | 被花费的UTXO的ID（对应基础结构中的SourceID + SourcePosition） |
| `AccountID` | string | 账户ID（如果输入关联到账户） |
| `AccountAlias` | string | 账户别名，用户友好的名称 |
| `Arbitrary` | HexBytes | 任意数据（仅Coinbase输入，通常包含区块高度等信息） |
| `InputID` | bc.Hash | 输入的唯一标识符（哈希值） |
| `WitnessArguments` | []HexBytes | 见证参数数组（签名等），用于解锁UTXO |
| `SignData` | bc.Hash | 签名数据哈希 |
| `Vote` | string | 投票信息（仅当输入类型为投票时） |
| `StateData` | []string | 状态数据数组 |

### 交易输入的核心概念

1. **交易输入标记BUTXO为已消费**，并提供所有权证明（通过解锁脚本）
2. **用户搜索钱包中足够的BUTXO**来满足支付请求
3. **每个交易输入指向一个BUTXO**，并提供相应的解锁脚本来解锁它
4. **每一笔普通交易的输出都会引用一个输入**（通过SpentOutputID）

### TxInput vs AnnotatedInput 对比

| 维度 | TxInput（基础结构） | AnnotatedInput（增强结构） |
|------|---------------------|---------------------------|
| **用途** | 区块链底层存储 | 查询API、区块浏览器显示 |
| **包含信息** | 核心数据（引用、资产、脚本） | 核心数据 + 用户友好信息 |
| **UTXO引用** | SourceID + SourcePosition | SpentOutputID（直接引用） |
| **地址** | 只有ControlProgram（字节） | 包含Address（人类可读） |
| **资产信息** | 只有AssetID | AssetID + AssetAlias + AssetDefinition |
| **账户信息** | 无 | AccountID + AccountAlias |
| **见证数据** | Arguments（字节数组） | WitnessArguments（十六进制数组） |
| **使用场景** | 交易验证、UTXO管理 | 区块浏览器、钱包查询、API返回 |

### 示例

```json
{
  "type": "spend",
  "asset_id": "ffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffff",
  "asset_alias": "BTM",
  "amount": 3179800000000,
  "address": "bm1q5wtajhqBvxvu3pedct23qcvqqww44wa",
  "spent_output_id": "be08e806d5fa7ba234a586fb8ed68a94c0ff79f657a1c9064db124336acbfa55",
  "account_id": "account_456",
  "account_alias": "Bob",
  "control_program": "00148397d2cae03019667221cb70b514186010076876",
  "input_id": "cb299ef868d4f7de93c2dfb23b955283fc769179636f4dcfab344976056287",
  "witness_arguments": [
    "01a1b2c3d4e5f6...",
    "02f6e5d4c3b2a1..."
  ]
}
```

---

## AnnotatedOutput（增强型交易输出）

### 结构定义

```go
// AnnotatedOutput 表示一个带注释的交易输出（用于查询和显示）
type AnnotatedOutput struct {
    Type            string             `json:"type"`              // 输出类型（"control", "vote"等）
    OutputID        bc.Hash            `json:"id"`                 // 输出唯一标识符
    TransactionID   *bc.Hash           `json:"transaction_id"`    // 创建该输出的交易ID
    Position        int                `json:"position"`           // 输出在交易中的位置（索引）
    AssetID         bc.AssetID         `json:"asset_id"`           // 资产ID
    AssetAlias      string             `json:"asset_alias"`       // 资产别名（用户友好）
    AssetDefinition *json.RawMessage   `json:"asset_definition"`   // 资产定义（JSON格式）
    Amount          uint64             `json:"amount"`             // 金额
    AccountID       string             `json:"account_id"`         // 账户ID
    AccountAlias    string             `json:"account_alias"`      // 账户别名（用户友好）
    ControlProgram  chainjson.HexBytes `json:"control_program"`   // 控制脚本（锁定脚本）
    Address         string             `json:"address"`           // 地址（从ControlProgram派生）
    Vote            string             `json:"vote,omitempty"`     // 投票信息（仅投票输出）
    StateData       []string           `json:"state_data"`         // 状态数据
}
```

### 字段说明

| 字段 | 类型 | 说明 |
|------|------|------|
| `Type` | string | 输出类型：`"control"`（普通输出）或 `"vote"`（投票输出） |
| `OutputID` | bc.Hash | 输出的唯一标识符（哈希值） |
| `TransactionID` | *bc.Hash | 创建该输出的交易ID |
| `Position` | int | 输出在交易输出数组中的位置（从0开始） |
| `AssetID` | bc.AssetID | 资产ID（如BTM的资产ID） |
| `AssetAlias` | string | 资产别名，用户友好的名称（如"BTM"） |
| `AssetDefinition` | *json.RawMessage | 资产的完整定义（JSON格式） |
| `Amount` | uint64 | 输出金额（最小单位） |
| `AccountID` | string | 账户ID（如果输出关联到账户） |
| `AccountAlias` | string | 账户别名，用户友好的名称 |
| `ControlProgram` | HexBytes | 控制脚本（锁定脚本），十六进制格式 |
| `Address` | string | 地址（从ControlProgram派生的人类可读地址） |
| `Vote` | string | 投票信息（仅当输出类型为投票时） |
| `StateData` | []string | 状态数据数组 |

### 交易输出的核心概念

1. **每个普通交易都会创建输出**，这些输出被记录在BUTXO（比原链未花费交易输出）集合中
2. **输出包含两个重要信息：**
   - **资产信息**：资产类型和资产数量
   - **花费条件**：锁定脚本（ControlProgram），定义了花费该输出的条件
3. **当用户想要花费一个BUTXO时**，必须在新的交易中提供能够解锁锁定脚本的有效参数

### TxOutput vs AnnotatedOutput 对比

| 维度 | TxOutput（基础结构） | AnnotatedOutput（增强结构） |
|------|---------------------|---------------------------|
| **用途** | 区块链底层存储 | 查询API、区块浏览器显示 |
| **包含信息** | 核心数据（资产、金额、脚本） | 核心数据 + 用户友好信息 |
| **地址** | 只有ControlProgram（字节） | 包含Address（人类可读） |
| **资产信息** | 只有AssetID | AssetID + AssetAlias + AssetDefinition |
| **账户信息** | 无 | AccountID + AccountAlias |
| **使用场景** | 交易验证、UTXO管理 | 区块浏览器、钱包查询、API返回 |

### 示例

```json
{
  "type": "control",
  "id": "058c30247fa93cc30690799aa5a34f11df94m27562045094c4fd2d896125",
  "transaction_id": "7d769fb564e5dc4947dab944c33e1b3ec522b8celebdaf365d00edf576fbe7c1",
  "position": 0,
  "asset_id": "ffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffff",
  "asset_alias": "BTM",
  "amount": 999999500000,
  "account_id": "account_123",
  "account_alias": "Alice",
  "control_program": "0014666708df9c6c25701206eda04cflala659bcafe4",
  "address": "bm1qvr68qcfjw6lu7euqfr9gky6g9rxgjegrsjkd8d"
}
```

---

## 基础结构 vs 增强结构总结

### 为什么需要两种结构？

1. **基础结构（TxInput/TxOutput）**：
   - 用于区块链底层存储和验证
   - 只包含核心数据，体积小
   - 适合序列化和网络传输

2. **增强结构（AnnotatedInput/AnnotatedOutput）**：
   - 用于查询API和用户界面显示
   - 包含用户友好的信息（别名、地址等）
   - 适合区块浏览器、钱包等应用

### 转换关系

```go
// 基础结构 → 增强结构（需要查询额外信息）
func ConvertToAnnotatedInput(txInput *TxInput, assetInfo *AssetInfo, accountInfo *AccountInfo) *AnnotatedInput {
    return &AnnotatedInput{
        Type:            getInputType(txInput),
        AssetID:         txInput.AssetID(),
        AssetAlias:      assetInfo.Alias,      // 需要查询
        Address:         deriveAddress(txInput.ControlProgram),  // 需要计算
        SpentOutputID:   computeOutputID(txInput),  // 需要计算
        AccountAlias:    accountInfo.Alias,   // 需要查询
        // ...
    }
}
```

---

## 相关资源

- [比原链交易类型对比](./比原链交易类型对比.md)
- [交易基础数据结构](./交易基础数据结构.md)
- [交易Action数据结构详解](./交易Action数据结构详解.md) - Input/Output Action 类型详解
- [UTXO模型详解](./UTXO模型详解.md)
- [比原链技术文档](https://github.com/Bytom/bytom)

---

**返回**: [第六章目录](./README.md)

