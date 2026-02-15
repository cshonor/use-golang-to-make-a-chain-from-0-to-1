# 交易 Action 数据结构详解

> 比原链（Bytom）交易 Action 数据结构的详细说明

---

## 概述

在比原链中，每一笔交易（Transaction）都由多个**交易动作（Transaction Action）**组成。交易动作是比原链交易系统的基础单元，用于描述交易中的输入和输出操作。

**核心概念：**
- 交易由多个 Action 组成
- Action 分为两大类：**Input Action**（输入动作）和 **Output Action**（输出动作）
- 每个 Action 对应交易中的一个 Input 或 Output

---

## Action 分类体系

```
交易 (Transaction)
├── Input Actions (输入动作)
│   ├── issue (发行资产)
│   ├── spend_account (账户模式花费)
│   ├── spend_account_unspent_output (直接花费指定UTXO)
│   └── coinbase (币基交易)
│
└── Output Actions (输出动作)
    ├── control_address (地址模式接收)
    ├── control_program (合约模式接收)
    ├── control (控制型输出)
    ├── retire (销毁资产)
    └── vote (投票型输出)
```

---

## Input Action 详解

### 1. issue（发行资产）

**定义：** 用于创建和发行新的资产。

**特点：**
- 创建新的资产类型
- 需要提供资产定义（AssetDefinition）
- 需要发行程序（IssuanceProgram）
- 需要签名证明发行权限

**Go 代码结构：**
```go
// 在 AnnotatedInput 中
in.Type = "issue"
in.IssuanceProgram = orig.ControlProgram()  // 发行程序
in.AssetDefinition = &assetDefinition       // 资产定义（JSON格式）
```

**JSON 示例：**
```json
{
  "type": "issue",
  "asset_id": "a1b2c3d4...",
  "amount": 1000000,
  "asset_definition": {
    "name": "GOLD",
    "decimals": 8,
    "description": "Gold Token"
  },
  "issuance_program": "0014...",
  "witness_arguments": ["signature1", "signature2"]
}
```

**使用场景：**
- 创建新的代币资产
- 发行自定义资产
- 资产首次发行

---

### 2. spend_account（账户模式花费）

**定义：** 以账户模式花费 UTXO，钱包会自动选择合适的 UTXO 进行花费。

**特点：**
- 指定账户ID，由钱包自动选择UTXO
- 不需要手动指定具体的 UTXO
- 钱包会根据金额自动组合多个 UTXO
- 需要账户签名

**Go 代码结构：**
```go
type SpendAccountAction struct {
    AccountID string `json:"account_id"`
    *bc.AssetAmount
}

func (s *SpendAccountAction) MarshalJSON() ([]byte, error) {
    return json.Marshal(&struct {
        Type      string `json:"type"`
        AccountID string `json:"account_id"`
        *bc.AssetAmount
    }{
        Type:        "spend_account",
        AccountID:   s.AccountID,
        AssetAmount: s.AssetAmount,
    })
}
```

**JSON 示例：**
```json
{
  "type": "spend_account",
  "account_id": "0ABC123...",
  "asset_id": "ffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffff",
  "amount": 10000000000
}
```

**使用场景：**
- 用户转账（钱包自动选择UTXO）
- 支付交易手续费
- 日常交易操作

**工作流程：**
1. 用户指定账户和金额
2. 钱包查询账户下的所有 UTXO
3. 钱包自动选择并组合合适的 UTXO
4. 创建 SpendInput 引用选中的 UTXO
5. 使用账户私钥签名

---

### 3. spend_account_unspent_output（直接花费指定UTXO）

**定义：** 直接花费指定的 UTXO，需要明确指定要花费的 UTXO。

**特点：**
- 必须明确指定要花费的 UTXO（SourceID + SourcePosition）
- 精确控制花费哪个输出
- 适用于需要精确控制 UTXO 的场景
- 需要提供签名解锁指定的 UTXO

**与 spend_account 的区别：**

| 特性 | spend_account | spend_account_unspent_output |
|------|--------------|------------------------------|
| UTXO选择 | 自动选择 | 手动指定 |
| 灵活性 | 高（钱包自动优化） | 低（必须精确指定） |
| 使用场景 | 日常转账 | 精确控制、批量操作 |
| 参数 | account_id + amount | SourceID + SourcePosition |

**使用场景：**
- 需要精确控制花费哪个 UTXO
- 批量交易优化
- 特殊业务逻辑
- 需要组合特定 UTXO 的场景

---

### 4. coinbase（币基交易）

**定义：** 矿工创建的特殊交易，用于发放区块奖励。

**特点：**
- 没有输入来源（不引用 UTXO）
- 直接创建新币
- 每个区块只有一笔
- 不需要签名

**详细说明：** 参见 [Coinbase交易详解](./Coinbase交易详解.md)

---

## Output Action 详解

### 1. control_address（地址模式接收）

**定义：** 接收方式为地址模式，使用标准的比原链地址。

**特点：**
- 使用地址字符串（如 `bm1q3yt265592czgh96r0uz63ta8fq40uzu5a8c2h0`）
- 地址会自动转换为 ControlProgram
- 最常用的接收方式
- 用户友好的接口

**Go 代码结构：**
```go
type ControlAddressAction struct {
    Address string `json:"address"`
    *bc.AssetAmount
}

func (c *ControlAddressAction) MarshalJSON() ([]byte, error) {
    return json.Marshal(&struct {
        Type    string `json:"type"`
        Address string `json:"address"`
        *bc.AssetAmount
    }{
        Type:        "control_address",
        Address:     c.Address,
        AssetAmount: c.AssetAmount,
    })
}
```

**JSON 示例：**
```json
{
  "type": "control_address",
  "address": "bm1q3yt265592czgh96r0uz63ta8fq40uzu5a8c2h0",
  "asset_id": "ffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffff",
  "amount": 10000000000
}
```

**使用场景：**
- 普通转账接收
- 用户间资产转移
- 最常用的输出方式

**工作流程：**
1. 用户提供接收地址
2. 系统将地址转换为 ControlProgram（脚本）
3. 创建 TxOutput，设置 ControlProgram
4. 接收方可以使用对应私钥解锁

---

### 2. control_program（合约模式接收）

**定义：** 接收方式为合约/程序模式，直接使用控制脚本（ControlProgram）。

**特点：**
- 直接使用十六进制编码的控制脚本
- 更灵活，支持复杂脚本
- 可以创建多签、时间锁等复杂条件
- 需要理解脚本编程

**JSON 示例：**
```json
{
  "type": "control",
  "control_program": "00148916ad528556048b97437f05a8afa7482afe0b94",
  "asset_id": "ffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffff",
  "amount": 10000000000
}
```

**使用场景：**
- 智能合约交互
- 多签钱包
- 时间锁交易
- 复杂条件解锁

**与 control_address 的区别：**

| 特性 | control_address | control_program |
|------|----------------|-----------------|
| 输入格式 | 地址字符串 | 十六进制脚本 |
| 用户友好性 | 高 | 低 |
| 灵活性 | 标准地址 | 任意脚本 |
| 使用难度 | 简单 | 需要脚本知识 |

---

### 3. control（控制型输出）

**定义：** 基础的控制型输出，是其他控制类型的基础。

**特点：**
- 最基础的输出类型
- 包含 ControlProgram 脚本
- 可以包含 StateData（状态数据）
- 支持智能合约状态

**在代码中的识别：**
```go
case orig.OutputType() == types.OriginalOutputType:
    out.Type = "control"
    if e, ok := tx.Entries[*outid]; ok {
        if output, ok := e.(*bc.OriginalOutput); ok {
            out.StateData = stateDataStrings(output.StateData)
        }
    }
```

**JSON 示例：**
```json
{
  "type": "control",
  "id": "500444582ea8dd42f7252380f5bd7b4f316762ae08180124ad4e05651376e567",
  "position": 0,
  "asset_id": "ffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffff",
  "amount": 41250500000,
  "control_program": "00148916ad528556048b97437f05a8afa7482afe0b94",
  "address": "bm1q3yt265592czgh96r0uz63ta8fq40uzu5a8c2h0",
  "state_data": []
}
```

---

### 4. retire（销毁资产）

**定义：** 销毁资产，将资产发送到不可花费的输出。

**特点：**
- 资产被永久销毁，无法恢复
- 使用不可花费的脚本（OP_FAIL）
- 可以包含任意数据（Arbitrary）
- 常用于资产销毁、消息存储

**Go 代码结构：**
```go
type RetireAction struct {
    *bc.AssetAmount
    Arbitrary []byte
}

func (r *RetireAction) MarshalJSON() ([]byte, error) {
    return json.Marshal(&struct {
        Type      string `json:"type"`
        Arbitrary string `json:"arbitrary"`
        *bc.AssetAmount
    }{
        Type:        "retire",
        Arbitrary:   hex.EncodeToString(r.Arbitrary),
        AssetAmount: r.AssetAmount,
    })
}
```

**识别方法：**
```go
// 检查 ControlProgram 是否为不可花费脚本
case vmutil.IsUnspendable(out.ControlProgram):
    out.Type = "retire"
```

**JSON 示例：**
```json
{
  "type": "retire",
  "asset_id": "a1b2c3d4...",
  "amount": 100,
  "arbitrary": "48656c6c6f20576f726c64",  // "Hello World" 的十六进制
  "control_program": "6a"  // OP_FAIL，不可花费
}
```

**使用场景：**
- 资产销毁（代币回购销毁）
- 链上消息存储（利用 Arbitrary 字段）
- 交易备注（将备注存储在销毁输出中）
- 减少资产供应量

**重要说明：**
- 销毁的资产**永远无法恢复**
- Arbitrary 字段可以存储任意数据（如消息、备注）
- 销毁操作需要支付手续费

---

### 5. vote（投票型输出）

**定义：** 用于投票和治理的输出类型。

**特点：**
- 包含投票信息（Vote）
- 用于链上治理
- 支持投票相关功能

**识别方法：**
```go
case orig.OutputType() == types.VoteOutputType:
    out.Type = "vote"
    if e, ok := tx.Entries[*outid]; ok {
        if output, ok := e.(*bc.VoteOutput); ok {
            // 处理投票输出
        }
    }
```

**使用场景：**
- 链上投票
- 治理提案
- 共识机制相关

---

## Action 与底层结构的关系

### Input Action → TxInput 映射

| Input Action | 对应的 TxInput 类型 | InputType 常量 |
|-------------|-------------------|---------------|
| `issue` | `IssuanceInput` | `IssuanceInputType` |
| `spend_account` | `SpendInput` | `SpendInputType` |
| `spend_account_unspent_output` | `SpendInput` | `SpendInputType` |
| `coinbase` | `CoinbaseInput` | `CoinbaseInputType` |

### Output Action → TxOutput 映射

| Output Action | 对应的 TxOutput 类型 | 识别方式 |
|--------------|---------------------|---------|
| `control_address` | `OriginalTxOutput` | 地址转换的 ControlProgram |
| `control_program` | `OriginalTxOutput` | 直接使用 ControlProgram |
| `control` | `OriginalTxOutput` | `OutputType() == OriginalOutputType` |
| `retire` | `OriginalTxOutput` | `IsUnspendable(ControlProgram)` |
| `vote` | `VoteOutput` | `OutputType() == VoteOutputType` |

---

## 实际交易示例

### 示例1：发行资产并转账

```json
{
  "actions": [
    {
      "type": "spend_account",
      "account_id": "alice",
      "asset_id": "ffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffff",
      "amount": 1000000000
    },
    {
      "type": "issue",
      "asset_definition": {
        "name": "GOLD",
        "decimals": 8
      },
      "amount": 100
    },
    {
      "type": "control_address",
      "address": "bm1q3yt265592czgh96r0uz63ta8fq40uzu5a8c2h0",
      "asset_id": "new_asset_id",
      "amount": 50
    }
  ]
}
```

**说明：**
- 使用 `spend_account` 花费 BTM 支付手续费
- 使用 `issue` 发行新资产 GOLD
- 使用 `control_address` 将部分 GOLD 发送到指定地址

### 示例2：销毁资产

```json
{
  "actions": [
    {
      "type": "spend_account",
      "account_id": "alice",
      "asset_id": "a1b2c3d4...",
      "amount": 100
    },
    {
      "type": "retire",
      "asset_id": "a1b2c3d4...",
      "amount": 100,
      "arbitrary": "4265726e"  // "Burn" 的十六进制
    }
  ]
}
```

**说明：**
- 使用 `spend_account` 花费要销毁的资产
- 使用 `retire` 将资产发送到不可花费的输出（销毁）

### 示例3：带备注的转账

```json
{
  "actions": [
    {
      "type": "spend_account",
      "account_id": "alice",
      "asset_id": "ffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffff",
      "amount": 10000000001
    },
    {
      "type": "control_address",
      "address": "bm1q3yt265592czgh96r0uz63ta8fq40uzu5a8c2h0",
      "asset_id": "ffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffff",
      "amount": 10000000000
    },
    {
      "type": "retire",
      "asset_id": "ffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffff",
      "amount": 1,
      "arbitrary": "48656c6c6f"  // 备注信息
    }
  ]
}
```

**说明：**
- 花费 10.00000001 BTM
- 发送 10 BTM 到目标地址
- 销毁 0.00000001 BTM 并附带备注（利用 retire 的 Arbitrary 字段存储消息）

---

## Action 类型总结表

### Input Actions

| Action 类型 | 中文名称 | 主要用途 | 是否需要签名 | 是否引用UTXO |
|------------|---------|---------|------------|------------|
| `issue` | 发行资产 | 创建新资产 | ✅ 是 | ❌ 否 |
| `spend_account` | 账户模式花费 | 自动选择UTXO花费 | ✅ 是 | ✅ 是（自动选择） |
| `spend_account_unspent_output` | 直接花费UTXO | 精确指定UTXO花费 | ✅ 是 | ✅ 是（手动指定） |
| `coinbase` | 币基交易 | 区块奖励 | ❌ 否 | ❌ 否 |

### Output Actions

| Action 类型 | 中文名称 | 主要用途 | 是否可花费 | 特殊说明 |
|------------|---------|---------|----------|---------|
| `control_address` | 地址模式接收 | 标准地址接收 | ✅ 是 | 最常用 |
| `control_program` | 合约模式接收 | 脚本控制接收 | ✅ 是 | 支持复杂脚本 |
| `control` | 控制型输出 | 基础控制输出 | ✅ 是 | 支持状态数据 |
| `retire` | 销毁资产 | 永久销毁资产 | ❌ 否 | 可存储消息 |
| `vote` | 投票型输出 | 链上投票 | ✅ 是 | 治理相关 |

---

## 常见问题

### Q1: spend_account 和 spend_account_unspent_output 如何选择？

**A:** 
- **日常使用**：选择 `spend_account`，让钱包自动优化 UTXO 选择
- **精确控制**：选择 `spend_account_unspent_output`，需要精确指定要花费的 UTXO
- **批量操作**：选择 `spend_account_unspent_output`，可以精确控制每个 UTXO

### Q2: control_address 和 control_program 的区别？

**A:**
- `control_address`：用户友好，使用地址字符串，系统自动转换为脚本
- `control_program`：更灵活，直接使用脚本，支持复杂条件

### Q3: retire 真的会销毁资产吗？

**A:** 是的。`retire` 将资产发送到使用 `OP_FAIL` 脚本的输出，这些输出**永远无法被花费**，资产被永久销毁。

### Q4: 如何识别一个 Output 的类型？

**A:** 
1. 检查 `ControlProgram` 是否为不可花费脚本 → `retire`
2. 检查 `OutputType()` → `control` 或 `vote`
3. 检查是否有地址字段 → `control_address`
4. 检查是否直接使用脚本 → `control_program`

### Q5: issue 和 coinbase 都能创建新资产吗？

**A:** 
- **issue**：创建**自定义资产**（如代币），需要提供资产定义
- **coinbase**：创建**原生资产**（BTM），系统自动发行，不需要资产定义

---

## 相关资源

- [交易基础数据结构](./交易基础数据结构.md)
- [Coinbase交易详解](./Coinbase交易详解.md)
- [普通交易详解](./普通交易详解.md)
- [Input和Output关系详解](./Input和Output关系详解.md)
- [比原链交易类型对比](./比原链交易类型对比.md)

---

**返回**: [第六章目录](./README.md)

