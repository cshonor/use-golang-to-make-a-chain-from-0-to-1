# dashboard 详解

> 第二章子文档：dashboard Web 图形界面详解

---

## 2.4 dashboard Web 图形界面

### 2.4.1 概述

`dashboard` 是比原链的 Web 图形界面，提供更友好的用户交互体验。

**特点：**
- ✅ Web 页面交互，用户体验友好
- ✅ 图形化界面，操作直观
- ✅ 适合普通用户
- ✅ 默认集成，无需额外部署

### 2.4.2 部署和访问

#### 默认启用

`dashboard` 功能在 `bytomd` 中**默认启用**，无需手动配置。

**启动 bytomd 时自动启动：**
```bash
bytomd node
```

#### 配置选项

**禁用 dashboard：**
```bash
bytomd node --web.closed
```

**启用 dashboard（默认）：**
```bash
bytomd node
# 或
bytomd node --web.enable
```

#### 访问地址

- **默认地址**：`http://localhost:9888`
- **配置端口**：可通过 `config.toml` 中的 `api_addr` 配置

### 2.4.3 功能特性

#### 主要功能模块

1. **📊 区块链信息展示**
   - 区块高度
   - 网络状态
   - 节点信息

2. **💼 钱包管理**
   - 账户列表
   - 余额查询
   - 地址管理

3. **📝 交易管理**
   - 交易查询
   - 交易创建
   - 交易历史

4. **🔍 区块浏览器**
   - 区块详情
   - 交易详情
   - 地址查询

5. **⚙️ 系统配置**
   - 网络配置
   - 钱包设置

### 2.4.4 技术实现

#### 源码位置

- **GitHub 仓库**：https://github.com/Bytom/bytom-dashboard
- **集成位置**：`bytom/dashboard/`

#### 通信方式

与 `bytomcli` 相同，`dashboard` 也通过 **JSON-RPC** 与 `bytomd` 通信：

```
Web 浏览器
    ↓
JavaScript 前端代码
    ↓
HTTP/JSON-RPC 请求
    ↓
bytomd API Server
    ↓
处理并返回结果
    ↓
前端展示
```

---

**返回**: [交互工具详解](./交互工具详解.md)

