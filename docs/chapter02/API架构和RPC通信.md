# API 架构和 RPC 通信

> 第二章子文档：API Service 架构和 RPC 通信原理详解

---

## 2.6 架构核心：API Service 是唯一入口

### 2.6.1 核心架构理解

**关键结论：**

✅ **所有交互工具（CLI、Web界面、第三方客户端）都必须通过 API Service**

✅ **API Service 是节点唯一对外的"大门"**

✅ **所有用户功能都从这一个门进入，再分发到不同的 Handler，最终调用内核层模块**

### 2.6.2 架构流程图

```
┌─────────────────────────────────────────────────────────┐
│              用户交互层（多种入口）                        │
├─────────────────────────────────────────────────────────┤
│  bytomcli (CLI)  │  dashboard (Web)  │  第三方客户端    │
└─────────────────────────────────────────────────────────┘
                    ↓
        ┌───────────────────────┐
        │   API Service         │
        │   (统一入口/大门)      │
        │   http://localhost:9888│
        └───────────────────────┘
                    ↓
    ┌───────────────────────────────┐
    │        各个 Handler            │
    ├───────────────────────────────┤
    │  account-handler  │  账户管理  │
    │  asset-handler     │  资产管理  │
    │  wallet-handler    │  钱包管理  │
    │  tx-handler       │  交易管理  │
    │  block-handler    │  区块查询  │
    │  mining-handler   │  挖矿相关  │
    │  token-handler    │  令牌管理  │
    └───────────────────────────────┘
                    ↓
    ┌───────────────────────────────┐
    │         内核层模块             │
    ├───────────────────────────────┤
    │  账户模块  │  资产模块  │  钱包模块│
    │  交易池   │  区块链    │  共识层  │
    │  P2P网络  │  数据存储  │  虚拟机  │
    └───────────────────────────────┘
```

### 2.6.3 关键理解点

#### 1. CLI 必须走 API Service

**重要：`bytomcli` 本质就是一个 API 调用工具**

- ❌ CLI **不直接**连接内核层
- ❌ CLI **不直接**访问数据库
- ❌ CLI **不直接**连接 P2P 网络
- ✅ CLI **只负责**构建 JSON-RPC 请求并发送给 API Service

**示例流程：**
```bash
# 用户执行命令
bytomcli create-account

# bytomcli 内部操作：
# 1. 解析命令和参数
# 2. 构建 JSON-RPC 请求
# 3. 发送 HTTP POST 到 http://localhost:9888
# 4. 等待 API Service 响应
# 5. 格式化并显示结果
```

#### 2. API Service 的作用

**API Service 是统一入口：**
- 接收所有外部请求（CLI、Web、第三方）
- 路由请求到对应的 Handler
- 统一处理认证、错误、日志等

#### 3. Handler 的作用

**Handler 是中间层：**
- 接收来自 API Service 的请求
- 验证参数和权限
- 调用对应的内核层模块
- 返回处理结果

**Handler 与功能模块的对应关系：**

| Handler | 功能分类 | 调用的内核模块 |
|---------|---------|--------------|
| `account-handler` | 账户管理 | `account.Manager` |
| `asset-handler` | 资产管理 | `asset.Registry` |
| `wallet-handler` | 钱包管理 | `wallet.Wallet` |
| `tx-handler` | 交易管理 | `protocol.TxPool`, `protocol.Chain` |
| `block-handler` | 区块查询 | `protocol.Chain` |
| `mining-handler` | 挖矿相关 | `consensus.Miner`, `blockproposer.BlockProposer` |
| `token-handler` | 令牌管理 | `accesstoken.CredentialStore` |

#### 4. 内核层模块的作用

**内核层是真正干活的：**
- 执行实际的业务逻辑
- 操作数据库
- 处理区块链状态
- 管理 P2P 网络

### 2.6.4 完整请求流程示例

**场景：创建账户**

```
1. 用户执行命令
   bytomcli create-account --quorum 1
   
2. bytomcli 构建请求
   POST http://localhost:9888
   {
     "jsonrpc": "2.0",
     "method": "create-account",
     "params": {"quorum": 1}
   }
   
3. API Service 接收请求
   - 验证访问令牌（如果需要）
   - 路由到 account-handler
   
4. account-handler 处理
   - 验证参数
   - 调用 account.Manager.CreateAccount()
   
5. 内核层 account.Manager
   - 创建账户对象
   - 保存到数据库
   - 返回账户信息
   
6. 响应返回
   account-handler → API Service → bytomcli → 用户
```

---

## 2.7 RPC 通信原理

### 2.7.1 RPC 协议

**RPC（Remote Procedure Call）** 是一种远程过程调用协议，允许客户端调用远程服务器上的函数，就像调用本地函数一样。

### 2.7.2 JSON-RPC 2.0

比原链使用 **JSON-RPC 2.0** 协议进行通信。

#### 请求格式

```json
{
  "jsonrpc": "2.0",
  "id": "请求ID",
  "method": "方法名",
  "params": {
    "参数1": "值1",
    "参数2": "值2"
  }
}
```

#### 响应格式

**成功响应：**
```json
{
  "jsonrpc": "2.0",
  "id": "请求ID",
  "result": {
    "返回数据": "值"
  }
}
```

**错误响应：**
```json
{
  "jsonrpc": "2.0",
  "id": "请求ID",
  "error": {
    "code": -32000,
    "message": "错误信息",
    "data": "错误详情"
  }
}
```

### 2.7.3 通信流程

```
客户端（bytomcli/dashboard）
    ↓
构建 JSON-RPC 请求
    ↓
HTTP POST 请求
    ↓
bytomd API Server
    ↓
路由到对应的处理函数
    ↓
执行业务逻辑
    ↓
返回 JSON-RPC 响应
    ↓
客户端解析响应
    ↓
显示结果
```

### 2.7.4 API 端点

**默认配置：**
- **地址**：`http://localhost:9888`
- **协议**：HTTP/HTTPS
- **格式**：JSON-RPC 2.0

**配置位置**：`config.toml`
```toml
api_addr = "0.0.0.0:9888"
```

---

**返回**: [交互工具详解](./交互工具详解.md)

