# Coinbase交易详解

> 比原链（Bytom）Coinbase交易（币基交易）的详细说明

---

## 概述

**Coinbase交易（币基交易）**是比原链中一种特殊的交易类型，由矿工/出块节点创建，用于发放区块奖励和收取交易手续费。它是每个区块的第一笔交易，也是唯一一种没有输入来源的交易。

---

## 核心特征

- **唯一发起方**：出块节点/矿工
- **位置**：每个区块的**第一笔交易**（索引为0）
- **作用**：发放区块奖励 + 收取交易手续费
- **特点**：**没有输入来源**，直接铸新币
- **唯一性**：一个区块只能有一笔 Coinbase 交易

---

## 数据结构（Go语言）

### CoinbaseInput 结构

```go
// Coinbase交易的Input结构
type CoinbaseInput struct {
    Arbitrary []byte  // 任意数据（通常包含区块高度等信息）
}
```

### 完整的Coinbase交易示例

```go
coinbaseTx := &Tx{
    Version: 1,
    Inputs: []*TxInput{
        {
            AssetVersion: 1,
            TypedInput: &CoinbaseInput{
                Arbitrary: []byte("block_height_12345"),  // 区块高度等任意信息
            },
        },
    },
    Outputs: []*TxOutput{
        {
            Amount: 41250000000,  // 区块奖励 + 手续费
            AssetID: BTMAssetID,  // BTM资产ID
            ControlProgram: minerAddress,  // 矿工地址
        },
    },
}
```

---

## 关键字段说明

### Arbitrary（任意数据）

- **类型**：`[]byte`（字节数组）
- **作用**：存储任意数据，通常包含：
  - 区块高度信息
  - 时间戳
  - 矿工自定义信息
- **特点**：不参与UTXO引用，仅用于存储元数据

### 与其他输入类型的区别

| 字段 | CoinbaseInput | SpendInput |
|------|--------------|------------|
| `SourceID` | ❌ **无** | ✅ **有**（引用的交易ID） |
| `SourcePosition` | ❌ **无** | ✅ **有**（引用的输出索引） |
| `Arguments` | ❌ **无**（不需要签名） | ✅ **有**（签名数据） |
| `AssetID()` | 返回空值 | 返回引用的资产ID |
| `Arbitrary` | ✅ **有**（存储元数据） | ❌ **无** |

---

## 工作原理

### 1. 创建时机

**矿工打包区块时**，自动创建Coinbase交易：
- 当矿工成功计算出符合难度要求的区块哈希时
- 在将交易打包进区块之前
- 作为区块的第一笔交易

### 2. 奖励计算

**计算总奖励** = 区块奖励 + 本区块所有交易的手续费

```go
func CalculateCoinbaseReward(blockHeight uint64, totalFee uint64) uint64 {
    blockReward := GetBlockReward(blockHeight)  // 根据区块高度计算区块奖励
    return blockReward + totalFee                // 区块奖励 + 手续费
}
```

### 3. 奖励发放

**将奖励发送给矿工地址**，作为挖矿激励：
- 奖励金额 = 固定区块奖励 + 本区块所有交易手续费
- 发送到矿工指定的地址（ControlProgram）
- 创建新的UTXO，成为矿工的资产

### 4. 验证机制

**无需验证输入**，因为这是系统直接发行的新币：
- 不需要引用之前的UTXO
- 不需要提供签名
- 系统自动验证Coinbase交易的有效性

---

## 实际应用场景

### 创建Coinbase交易

```go
// 矿工打包区块时创建Coinbase交易
func CreateCoinbaseTx(minerAddress []byte, blockHeight uint64, totalFee uint64) *Tx {
    blockReward := CalculateBlockReward(blockHeight)  // 计算区块奖励
    totalAmount := blockReward + totalFee              // 奖励 + 手续费
    
    return &Tx{
        Version: 1,
        Inputs: []*TxInput{
            NewCoinbaseInput([]byte(fmt.Sprintf("height_%d", blockHeight))),
        },
        Outputs: []*TxOutput{
            NewOriginalTxOutput(BTMAssetID, totalAmount, minerAddress, nil),
        },
    }
}
```

### 使用场景

- **矿工成功挖出新区块时**
  - 系统自动创建，无需用户操作
  - 奖励金额 = 固定区块奖励 + 本区块所有交易手续费

- **区块打包流程**
  1. 矿工收集待打包的交易
  2. 计算所有交易的手续费总和
  3. 创建Coinbase交易（包含区块奖励和手续费）
  4. 将Coinbase交易放在区块的第一位
  5. 打包其他交易

---

## 成熟期限制

### 为什么需要成熟期？

Coinbase交易的输出需要等待**成熟期（Maturity Period）**后才能被花费，这是为了防止51%攻击。

### 成熟期机制

```go
// Coinbase交易的输出需要等待成熟期
const CoinbasePendingBlockNumber = 100  // 需要等待100个区块

func IsCoinbaseMature(coinbaseHeight uint64, currentHeight uint64) bool {
    return currentHeight >= coinbaseHeight + CoinbasePendingBlockNumber
}
```

**规则：**
- Coinbase交易的输出需要等待 `CoinbasePendingBlockNumber` 个区块
- 在成熟期之前，Coinbase交易的输出不能被花费
- 成熟期之后，可以正常使用

**示例：**
```
区块高度 1000: Coinbase交易创建，输出 100 BTM
区块高度 1001-1099: 输出未成熟，不能花费
区块高度 1100: 输出成熟，可以花费
```

---

## 在区块浏览器中识别

### 识别方法

1. **查看交易的 `Inputs` 数组**
   - Coinbase交易只有一个输入

2. **检查Input类型**
   ```go
   if tx.Inputs[0].InputType() == CoinbaseInputType {
       // 这是 Coinbase 交易
   }
   ```

3. **查看Arbitrary字段**
   - 通常包含区块高度信息
   - 格式可能是：`"height_12345"` 或直接是区块高度的字节表示

4. **交易位置**
   - 必须是区块中的第一笔交易（索引为0）

5. **JSON格式**
   ```json
   {
     "inputs": [
       {
         "type": "coinbase",
         "arbitrary": "height_12345"
       }
     ]
   }
   ```

---

## 实际JSON结构示例

以下是一个真实的比原链Coinbase交易的完整JSON结构，来自区块浏览器：

```json
{
  "id": "360754aeed46658be88f5071af4bcae961e334f99018db1325e283aecbe9360f",
  "mux_id": "4c8f95edfb2fafbbf2209291e63e024306466d39d6cc0d7a59c77fc067280a5e",
  "size": 82,
  "status_fail": false,
  "time_range": 0,
  "version": 1,
  "inputs": [
    {
      "type": "coinbase",
      "amount": 0,
      "asset_id": "0000000000000000000000000000000000000000000000000000000000000000",
      "asset_definition": {},
      "arbitrary": "00313035343439",
      "input_id": "7cce7088ae01cd610fd7474f526292b760d38ad5e40ed82d37777f27882a2dc0",
      "witness_arguments": null
    }
  ],
  "outputs": [
    {
      "type": "control",
      "id": "500444582ea8dd42f7252380f5bd7b4f316762ae08180124ad4e05651376e567",
      "position": 0,
      "asset_id": "ffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffff",
      "asset_definition": {},
      "amount": 41250500000,
      "control_program": "00148916ad528556048b97437f05a8afa7482afe0b94",
      "address": "bm1q3yt265592czgh96r0uz63ta8fq40uzu5a8c2h0"
    }
  ]
}
```

### 关键字段解析

#### Input部分（Coinbase特征）
- **`type: "coinbase"`** - 明确标识这是Coinbase输入
- **`amount: 0`** - Coinbase输入金额为0，因为它不消耗现有UTXO
- **`asset_id: "0000...0000"`** - 全零资产ID，表示这是Coinbase输入
- **`arbitrary: "00313035343439"`** - 十六进制字符串，通常包含区块高度等信息
- **`witness_arguments: null`** - 无见证参数，因为不需要签名解锁

#### Output部分（奖励发放）
- **`type: "control"`** - 控制型输出，由脚本控制
- **`asset_id: "ffff...ffff"`** - 全F资产ID，表示比原链原生资产（BTM）
- **`amount: 41250500000`** - 奖励金额（单位：NEU，1 BTM = 10^8 NEU，即约412.505 BTM）
- **`control_program`** - 控制程序（脚本），定义花费条件
- **`address`** - 接收地址（矿工地址）

### 重要观察点

1. **Input金额为0**：这是Coinbase交易的核心特征，表示没有消耗任何现有UTXO
2. **Output金额**：`41250500000` NEU = 412.505 BTM，这是区块奖励加上本区块所有交易手续费的总和
3. **Arbitrary字段**：`"00313035343439"` 是十六进制编码，可能包含区块高度或其他元数据
4. **无见证参数**：Coinbase交易不需要签名，因为它是系统直接发行的新币

---

## 与其他链的对比

| 特性 | 比特币 | 比原链 |
|------|--------|--------|
| Coinbase交易 | ✅ 有 | ✅ 有 |
| 位置 | 区块第一笔 | 区块第一笔 |
| 输入来源 | 无 | 无 |
| 奖励内容 | 区块奖励 + 手续费 | 区块奖励 + 手续费 |
| 成熟期 | 100个区块 | 100个区块 |
| 特殊字段 | Coinbase数据 | Arbitrary字段 |

**比原链的特殊之处：**
- `Arbitrary` 字段可以存储更多元数据
- 支持多种资产类型（不仅限于BTM）

---

## 常见问题

### Q1: 为什么Coinbase交易没有输入？

**A:** Coinbase交易是系统直接发行的新币，不需要引用之前的UTXO。它是货币发行的唯一方式。

### Q2: Coinbase交易的手续费是多少？

**A:** Coinbase交易本身不支付手续费，手续费为0。它收取的是本区块其他交易的手续费。

### Q3: 可以创建多个Coinbase交易吗？

**A:** 不可以。每个区块只能有一笔Coinbase交易，且必须是第一笔交易。

### Q4: Coinbase交易的输出什么时候可以使用？

**A:** 需要等待 `CoinbasePendingBlockNumber`（通常为100）个区块确认后，输出才能被花费。

---

## 相关资源

- [比原链交易类型对比](./比原链交易类型对比.md)
- [交易基础数据结构](./交易基础数据结构.md)
- [交易Action数据结构详解](./交易Action数据结构详解.md) - Input/Output Action 类型详解
- [UTXO模型详解](./UTXO模型详解.md)
- [比原链技术文档](https://github.com/Bytom/bytom)

---

**返回**: [第六章目录](./README.md)

