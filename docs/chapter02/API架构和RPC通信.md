# API 架构和 RPC 通信

> 第二章子文档：API Service 架构和 RPC 通信原理详解

---

## 2.6 架构核心：API Service 是唯一入口

### 2.6.1 核心架构理解

**关键结论：**

✅ **所有交互工具（CLI、Web界面、第三方客户端）都必须通过 API Service**

✅ **API Service 是节点唯一对外的"大门"**

✅ **所有用户功能都从这一个门进入，再分发到不同的 Handler，最终调用内核层模块**

### 2.6.0 守护进程与 API Service 的关系

**核心理解：**

在比原链的架构里，**API Service 是守护进程（bytomd）对外提供服务的"窗口"**，而守护进程是所有核心功能的"容器"和"运行载体"。

#### 两者的角色定位

**守护进程（bytomd）：**
- 是整个节点的核心进程，负责加载并运行所有模块（钱包、P2P、区块链、挖矿等）
- 它是一个后台服务，持续运行，维护节点状态、同步区块、处理交易
- 可以把它理解成"节点本体"，所有底层逻辑都在这里
- **守护进程是"身体"**

**API Service：**
- 是守护进程内部的一个子模块（`Node` 结构体里的 `api` 字段）
- 它对外提供 HTTP/gRPC 接口，让 CLI（bytomcli）、Web 界面或第三方应用能和节点交互
- 可以把它理解成"节点的对外大门"，所有用户操作都必须通过它进入
- **API Service 是"嘴巴和耳朵"**

#### 它们的关系与交互流程

**1. 启动顺序：**
- 执行 `bytomd node` 启动守护进程
- 守护进程在初始化时，会根据配置创建并启动 API Service（除非用 `--web.closed` 关闭）

**2. 请求流转：**
- 用户通过 `bytomcli` 或其他客户端发送操作（如创建账户、查询余额、发送交易）
- 这些请求通过网络发送到 API Service
- API Service 解析请求，调用对应的 Handler
- Handler 再调用守护进程内部的核心模块（钱包、区块链等）执行实际操作
- 结果原路返回给客户端

**3. 生命周期绑定：**
- API Service 是守护进程的一部分，它的生命周期完全由守护进程管理
- 守护进程停止，API Service 也随之关闭
- 反之，关闭 API Service 不会停止守护进程，但会让节点无法被外部访问

**用一句话总结：**
**守护进程是"身体"，API Service 是"嘴巴和耳朵"。** 守护进程负责内部所有"干活"的逻辑，而 API Service 负责"对外沟通"，让用户能指挥这个身体。

### 2.6.2 架构流程图

**完整的守护进程与 API Service 交互流程：**

```
┌─────────────────────────────────────────────────────────┐
│              用户交互层（多种入口）                        │
├─────────────────────────────────────────────────────────┤
│  bytomcli (CLI)  │  dashboard (Web)  │  第三方客户端    │
└─────────────────────────────────────────────────────────┘
                    ↓
        ┌───────────────────────┐
        │   API Service         │ ← 守护进程的"嘴巴和耳朵"
        │   (统一入口/大门)      │   (对外沟通窗口)
        │   http://localhost:9888│   守护进程内部的一个模块
        └───────────────────────┘
                    ↓
    ┌───────────────────────────────┐
    │        各个 Handler            │
    │      (请求路由层)              │
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
┌─────────────────────────────────────────────────────────┐
│           守护进程（bytomd）- "身体"                      │
│   ┌─────────────────────────────────────────────────┐  │
│   │  Node 对象（节点本体）                           │  │
│   │  ├─ wallet（钱包模块）                           │  │
│   │  ├─ chain（区块链模块）                          │  │
│   │  ├─ txPool（交易池）                            │  │
│   │  ├─ syncManager（同步管理器）                    │  │
│   │  ├─ api（API Service）← 在这里                  │  │
│   │  └─ ...（其他核心模块）                          │  │
│   └─────────────────────────────────────────────────┘  │
│                                                          │
│   • 持续运行，维护节点状态                               │
│   • 同步区块和交易数据                                  │
│   • 处理 P2P 网络通信                                   │
│   • 执行所有核心业务逻辑                                │
└─────────────────────────────────────────────────────────┘
```

**关键点：**
- API Service 是守护进程内部的一个模块，不是独立进程
- 守护进程启动时，会创建并启动 API Service
- 所有外部请求都必须通过 API Service 进入守护进程
- Handler 调用守护进程内部的核心模块执行实际操作

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

