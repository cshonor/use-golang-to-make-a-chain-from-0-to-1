# bytomcli 详解

> 第二章子文档：bytomcli 命令行客户端详解

---

## 2.2 bytomcli 命令行客户端

### 2.2.1 概述

`bytomcli` 是比原链的命令行客户端，通过命令行界面与 `bytomd` 进程进行交互。

**特点：**
- ✅ 轻量级，资源占用少
- ✅ 适合脚本自动化和批量操作
- ✅ 适合开发者和高级用户
- ✅ 支持所有比原链功能

### 2.2.2 使用方法

#### 基本使用方式

`bytomcli` 有两种使用方式：

1. **直接使用 flags**：`bytomcli [flags]` - 查看帮助文档
2. **使用子命令**：`bytomcli [command]` - 执行具体操作

#### 查看帮助信息

**查看主帮助：**
```bash
bytomcli -h
# 或
bytomcli --help
```

**查看子命令帮助：**
```bash
bytomcli [command] -h
# 或
bytomcli [command] --help
```

**示例：**
```bash
# 查看创建账户命令的帮助
bytomcli create-account --help

# 查看构建交易命令的帮助
bytomcli build-transaction --help
```

**特点：**
- ✅ `bytomcli` 的帮助信息和示例非常详细
- ✅ 每个命令都有清晰的说明和示例
- ✅ 建议用户充分利用帮助信息

#### 命令格式

**基本格式：**
```bash
bytomcli [command] [flags] [arguments]
```

**示例：**
```bash
# 创建账户
bytomcli create-account --quorum 1

# 查询账户列表
bytomcli list-accounts

# 构建交易
bytomcli build-transaction --account-id <id> --asset-id <id> --amount 100
```

### 2.2.3 代码结构分析

#### 主入口文件

**文件位置**：`cmd/bytomcli/main.go`

```go
package main

import (
	"runtime"
	cmd "github.com/bytom/bytom/cmd/bytomcli/commands"
)

func main() {
	runtime.GOMAXPROCS(runtime.NumCPU())
	cmd.Execute()
}
```

**关键点：**
- 使用 `runtime.GOMAXPROCS(runtime.NumCPU())` 设置最大 CPU 核心数
- 调用 `cmd.Execute()` 执行命令

#### 命令结构

**文件位置**：`cmd/bytomcli/commands/bytomcli.go`

**根命令定义：**
```go
var BytomcliCmd = &cobra.Command{
	Use:   "bytomcli",
	Short: "Bytomcli is a commond line client for bytom core (a.k.a. bytomd)",
	Run: func(cmd *cobra.Command, args []string) {
		if len(args) < 1 {
			cmd.SetUsageTemplate(usageTemplate)
			cmd.Usage()
		}
	},
}
```

**命令注册：**
```go
func AddCommands() {
	// 访问令牌相关
	BytomcliCmd.AddCommand(createAccessTokenCmd)
	BytomcliCmd.AddCommand(listAccessTokenCmd)
	BytomcliCmd.AddCommand(deleteAccessTokenCmd)
	BytomcliCmd.AddCommand(checkAccessTokenCmd)

	// 账户相关
	BytomcliCmd.AddCommand(createAccountCmd)
	BytomcliCmd.AddCommand(deleteAccountCmd)
	BytomcliCmd.AddCommand(listAccountsCmd)
	BytomcliCmd.AddCommand(updateAccountAliasCmd)
	BytomcliCmd.AddCommand(createAccountReceiverCmd)
	BytomcliCmd.AddCommand(listAddressesCmd)
	BytomcliCmd.AddCommand(validateAddressCmd)
	BytomcliCmd.AddCommand(listPubKeysCmd)

	// 资产相关
	BytomcliCmd.AddCommand(createAssetCmd)
	BytomcliCmd.AddCommand(getAssetCmd)
	BytomcliCmd.AddCommand(listAssetsCmd)
	BytomcliCmd.AddCommand(updateAssetAliasCmd)

	// 交易相关
	BytomcliCmd.AddCommand(getTransactionCmd)
	BytomcliCmd.AddCommand(listTransactionsCmd)
	BytomcliCmd.AddCommand(getUnconfirmedTransactionCmd)
	BytomcliCmd.AddCommand(listUnconfirmedTransactionsCmd)
	BytomcliCmd.AddCommand(decodeRawTransactionCmd)

	// 钱包相关
	BytomcliCmd.AddCommand(listUnspentOutputsCmd)
	BytomcliCmd.AddCommand(listBalancesCmd)
	BytomcliCmd.AddCommand(rescanWalletCmd)
	BytomcliCmd.AddCommand(walletInfoCmd)

	// 交易构建相关
	BytomcliCmd.AddCommand(buildTransactionCmd)
	BytomcliCmd.AddCommand(signTransactionCmd)
	BytomcliCmd.AddCommand(submitTransactionCmd)
	BytomcliCmd.AddCommand(estimateTransactionGasCmd)

	// 区块相关
	BytomcliCmd.AddCommand(getBlockCountCmd)
	BytomcliCmd.AddCommand(getBlockHashCmd)
	BytomcliCmd.AddCommand(getBlockCmd)
	BytomcliCmd.AddCommand(getBlockHeaderCmd)

	// 密钥相关
	BytomcliCmd.AddCommand(createKeyCmd)
	BytomcliCmd.AddCommand(deleteKeyCmd)
	BytomcliCmd.AddCommand(listKeysCmd)
	BytomcliCmd.AddCommand(updateKeyAliasCmd)
	BytomcliCmd.AddCommand(resetKeyPwdCmd)
	BytomcliCmd.AddCommand(checkKeyPwdCmd)

	// 消息签名相关
	BytomcliCmd.AddCommand(signMsgCmd)
	BytomcliCmd.AddCommand(verifyMsgCmd)
	BytomcliCmd.AddCommand(decodeProgCmd)

	// 交易订阅相关
	BytomcliCmd.AddCommand(createTransactionFeedCmd)
	BytomcliCmd.AddCommand(listTransactionFeedsCmd)
	BytomcliCmd.AddCommand(deleteTransactionFeedCmd)
	BytomcliCmd.AddCommand(getTransactionFeedCmd)
	BytomcliCmd.AddCommand(updateTransactionFeedCmd)

	// 网络相关
	BytomcliCmd.AddCommand(netInfoCmd)
	BytomcliCmd.AddCommand(gasRateCmd)

	// 版本信息
	BytomcliCmd.AddCommand(versionCmd)
}
```

**技术栈：**
- 使用 **Cobra** 框架构建命令行工具
- 命令结构清晰，模块化设计
- 支持子命令和参数解析

### 2.2.4 命令分类

#### 1. 访问令牌管理（Access Token）

| 命令 | 功能 | 说明 |
|------|------|------|
| `create-access-token` | 创建访问令牌 | 创建新的访问令牌用于 API 认证 |
| `list-access-tokens` | 列出访问令牌 | 查看所有已创建的访问令牌 |
| `delete-access-token` | 删除访问令牌 | 删除指定的访问令牌 |
| `check-access-token` | 检查访问令牌 | 验证访问令牌的有效性 |

#### 2. 账户管理（Account）

| 命令 | 功能 | 说明 |
|------|------|------|
| `create-account` | 创建账户 | 创建新的账户，需要指定密钥和阈值 |
| `delete-account` | 删除账户 | 删除指定的账户 |
| `list-accounts` | 列出账户 | 查看所有账户列表 |
| `update-account-alias` | 更新账户别名 | 修改账户的别名 |
| `create-account-receiver` | 创建账户接收地址 | 为账户生成新的接收地址 |
| `list-addresses` | 列出地址 | 查看账户的所有地址 |
| `validate-address` | 验证地址 | 验证地址的有效性 |
| `list-pubkeys` | 列出公钥 | 查看账户的公钥列表 |

#### 3. 资产管理（Asset）

| 命令 | 功能 | 说明 |
|------|------|------|
| `create-asset` | 创建资产 | 创建新的资产类型 |
| `get-asset` | 获取资产信息 | 查询指定资产的详细信息 |
| `list-assets` | 列出资产 | 查看所有资产列表 |
| `update-asset-alias` | 更新资产别名 | 修改资产的别名 |

#### 4. 交易管理（Transaction）

| 命令 | 功能 | 说明 |
|------|------|------|
| `build-transaction` | 构建交易 | 构建交易模板 |
| `sign-transaction` | 签名交易 | 使用账户密码签名交易 |
| `submit-transaction` | 提交交易 | 提交已签名的交易到网络 |
| `estimate-transaction-gas` | 估算交易 Gas | 估算交易所需的 Gas 费用 |
| `get-transaction` | 获取交易信息 | 查询指定交易的详细信息 |
| `list-transactions` | 列出交易 | 查看交易列表 |
| `get-unconfirmed-transaction` | 获取未确认交易 | 查询未确认的交易 |
| `list-unconfirmed-transactions` | 列出未确认交易 | 查看未确认交易列表 |
| `decode-raw-transaction` | 解码原始交易 | 解码原始交易数据 |

#### 5. 钱包管理（Wallet）

| 命令 | 功能 | 说明 |
|------|------|------|
| `list-balances` | 列出余额 | 查看账户余额 |
| `list-unspent-outputs` | 列出未花费输出 | 查看 UTXO 列表 |
| `wallet-info` | 钱包信息 | 查看钱包详细信息 |
| `rescan-wallet` | 重新扫描钱包 | 重新扫描区块信息到钱包 |

#### 6. 密钥管理（Key）

| 命令 | 功能 | 说明 |
|------|------|------|
| `create-key` | 创建密钥 | 创建新的密钥对 |
| `delete-key` | 删除密钥 | 删除指定的密钥 |
| `list-keys` | 列出密钥 | 查看所有密钥列表 |
| `update-key-alias` | 更新密钥别名 | 修改密钥的别名 |
| `reset-key-password` | 重置密钥密码 | 重置密钥的密码 |
| `check-key-password` | 检查密钥密码 | 验证密钥密码是否正确 |

#### 7. 区块查询（Block）

| 命令 | 功能 | 说明 |
|------|------|------|
| `get-block-count` | 获取区块数量 | 查询当前区块高度 |
| `get-block-hash` | 获取区块哈希 | 查询最新区块的哈希 |
| `get-block` | 获取区块信息 | 查询指定区块的详细信息 |
| `get-block-header` | 获取区块头 | 查询区块头信息 |

#### 8. 网络信息（Network）

| 命令 | 功能 | 说明 |
|------|------|------|
| `net-info` | 网络信息 | 查看网络连接信息 |
| `gas-rate` | Gas 费率 | 查看当前的 Gas 费率 |

#### 9. 其他功能

| 命令 | 功能 | 说明 |
|------|------|------|
| `sign-message` | 签名消息 | 对消息进行签名 |
| `verify-message` | 验证消息 | 验证消息签名 |
| `decode-program` | 解码程序 | 解码程序为指令和数据 |
| `version` | 版本信息 | 查看 bytomcli 版本 |

### 2.2.5 RPC 通信机制

#### 请求流程

```
用户输入命令
    ↓
bytomcli 解析命令和参数
    ↓
构建 RPC 请求（JSON-RPC）
    ↓
通过 HTTP 发送到 bytomd
    ↓
bytomd 处理请求
    ↓
返回响应结果
    ↓
bytomcli 格式化输出
    ↓
显示给用户
```

#### 请求示例

**创建账户的请求结构：**
```json
{
  "jsonrpc": "2.0",
  "id": "1",
  "method": "create-account",
  "params": {
    "root_xpubs": ["..."],
    "quorum": 1,
    "alias": "my-account",
    "access_token": "..."
  }
}
```

**响应结构：**
```json
{
  "jsonrpc": "2.0",
  "id": "1",
  "result": {
    "id": "account-id",
    "alias": "my-account",
    ...
  }
}
```

### 2.2.6 使用示例

#### 基本操作流程

**1. 创建密钥**
```bash
bytomcli create-key
```

**2. 创建账户**
```bash
bytomcli create-account --quorum 1
```

**3. 创建资产**
```bash
bytomcli create-asset --alias "MyToken"
```

**4. 构建交易**
```bash
bytomcli build-transaction --account-id <account-id> --asset-id <asset-id> --amount 100
```

**5. 签名交易**
```bash
bytomcli sign-transaction --password <password>
```

**6. 提交交易**
```bash
bytomcli submit-transaction
```

#### 查询操作

**查询账户列表：**
```bash
bytomcli list-accounts
```

**查询余额：**
```bash
bytomcli list-balances
```

**查询区块信息：**
```bash
bytomcli get-block-count
bytomcli get-block --height 100
```

---

## 2.3 bytomcli 命令详细说明

### 2.3.1 完整命令列表

以下是 `bytomcli` 的所有可用命令及其功能说明：

#### 交易管理命令

| 命令 | 英文说明 | 中文说明 |
|------|----------|----------|
| `build-transaction` | Build one transaction template, default use account id and asset id | 创建一笔交易模板，默认使用账户ID和资产ID |
| `decode-raw-transaction` | decode the raw transaction | 解码原始交易 |
| `estimate-transaction-gas` | estimate gas for build transaction | 估算交易所需的 Gas 费用 |
| `get-transaction` | get the transaction by matching the given transaction hash | 根据交易哈希值获取交易信息 |
| `sign-transaction` | Sign transaction templates with account password | 使用账户密码对交易进行签名 |
| `submit-transaction` | Submit signed transaction | 提交已签名的交易 |
| `list-transactions` | List the transactions | 列出交易列表 |
| `get-unconfirmed-transaction` | get unconfirmed transaction by matching the given transaction hash | 根据交易哈希获取未确认的交易 |
| `list-unconfirmed-transactions` | list unconfirmed transactions hashes | 列出未确认交易的哈希列表 |

#### 账户和资产管理命令

| 命令 | 英文说明 | 中文说明 |
|------|----------|----------|
| `create-account` | Create an account | 创建账户 |
| `list-accounts` | List the existing accounts | 显示账户信息 |
| `delete-account` | Delete the existing account | 删除账户 |
| `update-account-alias` | update account alias | 更新账户别名 |
| `create-account-receiver` | Create an account receiver | 创建账户接收地址 |
| `list-addresses` | List the account addresses | 列出账户地址 |
| `validate-address` | validate the account addresses | 验证账户地址 |
| `list-pubkeys` | list the account pubkeys | 列出账户公钥 |
| `create-asset` | Create an asset | 创建资产 |
| `get-asset` | get asset by assetID | 根据资产ID获取资产信息 |
| `list-assets` | List the existing assets | 显示资产信息 |
| `update-asset-alias` | Update the asset alias | 更新资产别名 |

#### 密钥和访问令牌管理命令

| 命令 | 英文说明 | 中文说明 |
|------|----------|----------|
| `create-key` | Create a key | 创建密钥 |
| `delete-key` | Delete a key | 删除密钥 |
| `list-keys` | List the existing keys | 显示存在的密钥信息 |
| `update-key-alias` | Update key alias | 更新密钥别名 |
| `reset-key-password` | Reset key password | 重置密钥密码 |
| `check-key-password` | check key password | 检查密钥密码 |
| `create-access-token` | Create a new access token | 创建访问令牌 |
| `list-access-tokens` | List the existing access tokens | 显示访问令牌信息 |
| `delete-access-token` | Delete an access token | 删除访问令牌 |
| `check-access-token` | Check an access token | 检查访问令牌 |

#### 钱包管理命令

| 命令 | 英文说明 | 中文说明 |
|------|----------|----------|
| `list-balances` | List the accounts balances | 显示账户余额 |
| `list-unspent-outputs` | List the accounts unspent outputs | 列出账户的未花费输出（UTXO） |
| `wallet-info` | Print the information of wallet | 打印钱包信息 |
| `rescan-wallet` | Trigger to rescan block information into related wallet | 重新扫描钱包 |

#### 区块和网络信息命令

| 命令 | 英文说明 | 中文说明 |
|------|----------|----------|
| `get-block` | Get a whole block matching the given hash or height | 根据区块哈希值或高度获取完整区块信息 |
| `get-block-count` | Get the number of most recent block | 获取最新区块的数量 |
| `get-block-hash` | Get the hash of most recent block | 获取最新区块的哈希值 |
| `get-block-header` | Get the header of a block matching the given hash or height | 获取区块头信息 |
| `net-info` | Print the summary of network | 显示网络信息 |
| `gas-rate` | Print the current gas rate | 打印当前的 Gas 费率 |

#### 消息签名和验证命令

| 命令 | 英文说明 | 中文说明 |
|------|----------|----------|
| `sign-message` | sign message to generate signature | 签名消息以生成签名 |
| `verify-message` | verify signature for specified message | 验证指定消息的签名 |

#### 程序解码命令

| 命令 | 英文说明 | 中文说明 |
|------|----------|----------|
| `decode-program` | decode program to instruction and data | 解码程序为指令和数据 |

#### 交易订阅命令

| 命令 | 英文说明 | 中文说明 |
|------|----------|----------|
| `create-transaction-feed` | Create a transaction feed filter | 创建交易订阅过滤器 |
| `list-transaction-feeds` | list all of transaction feeds | 列出所有交易订阅 |
| `delete-transaction-feed` | Delete a transaction feed filter | 删除交易订阅过滤器 |
| `get-transaction-feed` | get a transaction feed by alias | 根据别名获取交易订阅 |
| `update-transaction-feed` | Update transaction feed | 更新交易订阅 |

#### 工具命令

| 命令 | 英文说明 | 中文说明 |
|------|----------|----------|
| `version` | Print the version number of Bytomcli | 打印 Bytomcli 的版本号 |
| `help` | Help about any command | 显示任何命令的帮助信息 |

### 2.3.2 命令使用技巧

#### 1. 查看命令帮助

每个命令都支持 `-h` 或 `--help` 参数查看详细帮助：

```bash
# 查看命令的详细帮助和示例
bytomcli create-account --help
```

#### 2. 命令参数说明

大多数命令都支持通过参数指定操作对象：

```bash
# 使用 ID 指定账户
bytomcli list-balances --id <account-id>

# 使用别名指定账户
bytomcli list-balances --alias <account-alias>
```

#### 3. 组合使用命令

可以组合多个命令完成复杂操作：

```bash
# 1. 创建账户
bytomcli create-account

# 2. 创建资产
bytomcli create-asset

# 3. 构建交易
bytomcli build-transaction --account-id <id> --asset-id <id> --amount 100

# 4. 签名交易
bytomcli sign-transaction --password <password>

# 5. 提交交易
bytomcli submit-transaction
```

---

**返回**: [交互工具详解](./交互工具详解.md)

