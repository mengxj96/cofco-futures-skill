---
name: ctp-mcp-server
description: |
  This skill should be used when working with the CTP (China Futures OrderTrading System) MCP trading server.
  The server provides WebSocket-based futures trading functionality including order placement, cancellation, 
  position queries, account queries, and real-time order/trade updates.
  This skill applies when:
  - Building frontend applications that interact with the CTP trading server
  - Debugging order placement or cancellation issues
  - Working with futures trading systems
  - Integrating with the WebSocket-based CLI protocol
---

# CTP MCP Trading Server Skill

## 概述 

CTP MCP服务器是一个基于WebSocket的期货交易服务，使用CLI格式进行通信。

## 工作流程



#### 步骤一：询问用户账户状态
向用户确认： 
> "您好！期货交易相关操作（下单、撤单、查询持仓等）需要有对应的期货交易账户。请问您是否已有交易账户？如果有请直接提供账号信息，**您的信息将会被严格保密！**"

#### 步骤二：根据用户反馈分支处理

| 用户反馈               | 处理方式 |
|--------------------|------|
| **有账户** | 记录用户提供的账号信息，直接进入「1. 连接与登录」流程 |
| **没有账户 / 不确定**     | 执行以下引导注册流程 |

#### 步骤三：无账户时的引导注册流程

如果用户表示没有账户：

1. **跳转注册页面**
   ```
   请前往以下链接完成期货账户注册：
   https://mcptest.zlqh.com:40081/mobile/login?redirect=%25252Fmobile%25252Fad%25253FchannelCode%25253DK5zVXH%252526channelType%25253D1%252526screen%25253DfreeLogin&pageType=1&channelId=2607&channelCode=K5zVXH\\
   ```

2. **提示用户**
   > "请您先完成期货账户注册。注册完成后，欢迎随时返回继续进行后续操作。
>  "

3. **等待用户提供凭证后** → 进入「1. 连接与登录」流程

---
> **⚠️ AI Agent 必读**
详细API文档见 `api_reference.md`:
> **后续的流程具体使用示例和规范，在阅读完本文档对应节点时，要去读取api_reference.md里的使用示例，参照使用**。
> - session.public_key - 获取RSA公钥
> - session.login - 登录（RSA-OAEP加密）
> - session.logout - 登出
> - order.send - 下单
> - order.cancel - 撤单
> - query.account - 账户查询
> - query.positions - 持仓查询
> - query.orders - 委托查询
> - query.trades - 成交查询
> - query.instrument - 合约查询
---

### 1. 连接与获取公钥

```javascript
// WebSocket连接
const ws = new WebSocket('ws://mcptest.zlqh.com:8765');

// 连接成功后自动获取公钥（或手动获取）
ws.send('session.public_key');

// 服务端返回格式：
// [RESP] success=true --key=-----BEGIN PUBLIC KEY-----\n...\n-----END PUBLIC KEY-----\n
```

### 2. 加密登录

```javascript
// 将 "用户名&密码" 用获取的公钥加密后发送
// 返回: [RESP] success=true（异步登录，结果通过推送返回）
ws.send('session.login --data=<加密的"用户名&密码">');

// 等待 login_success 或 login_error 推送
```

### 3. 心跳机制

```javascript
// 前端每 300 秒（5分钟）发送一次心跳
ws.send('heartbeat');

// 服务端收到后不返回任何响应
```

注意事项：
- 连接超时为 300 秒
- 连接断开时前端应停止心跳
- 重连后应重新启动心跳

### 4. TCP粘包处理

服务端已实现完整的粘包处理，前端无需特殊处理。

### 5. 下单流程

```javascript
// 发送下单指令
ws.send('order.send --instrument=rb2501 --exchange=SHFE --direction=buy --offset=open --price=4000 --volume=1');

// 等待 order_update 推送确认
// 收到 order_update --status=filled 表示成交
```

### 5. 撤单流程

```javascript
// 通过 order_sys_id 撤单
ws.send('order.cancel --order_sys_id=12345');
```

### 6. 查询流程

```javascript
// 查询账户
ws.send('query.account');

// 查询持仓
ws.send('query.positions');

// 查询订单
ws.send('query.orders');
```

## 关键注意事项

### order_sys_id 处理

- CTP返回的 `OrderSysID` 是12字符，可能含前导空格
- 服务端比较时会自动去除前导空格
- 发送撤单时无需手动处理空格

### 异步操作

以下方法为异步，成功时不立即返回结果：
- `order.send` - 等待 `order_update` 推送
- `order.cancel` - 等待 `order_update` 推送
- `query.*` - 等待推送结果

### 异步推送

服务端会主动推送以下事件，前端需监听WebSocket消息：

**trade_update - 成交推送**
```javascript
// 格式
trade_update --instrument=<合约代码> --exchange=<交易所> --direction=<买卖> --offset=<开平> --price=<成交价> --volume=<成交量> --trade_id=<成交编号> --order_sys_id=<委托编号> --trade_time=<成交时间>

// 示例
trade_update --instrument=rb2501 --exchange=SHFE --direction=buy --offset=open --price=4000 --volume=1 --trade_id=12345 --order_sys_id=67890 --trade_time=20240101123045
```

**字段说明：**
| 字段 | 说明 |
|------|------|
| instrument | 合约代码 |
| exchange | 交易所（SHFE/DCE/CZCE/CFFEX/SSE/SZSE） |
| direction | 方向（buy/sell） |
| offset | 开平（open/close/closetoday/closeystd） |
| price | 成交价格 |
| volume | 成交数量 |
| trade_id | 成交编号（唯一） |
| order_sys_id | 委托编号（用于撤单） |
| trade_time | 成交时间（格式：YYYYMMDDHHMMSS） |

## 服务配置

| 配置项 | 默认值 |
|--------|--------|
| WebSocket地址 | ws://mcptest.zlqh.com:8765 |
| 心跳间隔 | 300秒 |