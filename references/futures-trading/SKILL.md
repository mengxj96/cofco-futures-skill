---
name: simulated-trading-skill
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

# Simulated Trading Server Skill

## 概述 

期货模拟交易服务基于WebSocket，使用CLI格式进行通信。支持TCP粘包处理。

## 工作流程

### 0. 确认用户信息

### 步骤一：询问用户账户状态
向用户确认： 
> "您好！期货模拟交易相关操作（下单、撤单、账户相关查询等）需要有对应的期货模拟交易账户。请问您是否已有账户？如果有请直接提供账号信息，**您的信息将会被严格保密！**"

### 步骤二：根据用户反馈分支处理

| 用户反馈               | 处理方式 |
|--------------------|------|
| **有账户** | 记录用户提供的账号信息，直接进入「1. 连接与登录」流程 |
| **没有账户 / 不确定**     | 执行以下引导注册流程 |

### 步骤三：无账户时的引导注册流程

如果用户表示没有账户：

1. **跳转注册页面**
   ```
   请前往以下链接完成期货模拟交易账户注册：
   [账户注册入口](https://mcptest.zlqh.com:40081/mobile/ad?channelCode=K5zVXH&channelType=1&screen=freeLogin)
   ```

2. **提示用户**
   > "请您先完成期货模拟交易账户注册。注册完成后，欢迎随时返回继续进行后续操作。"

3. **等待用户提供凭证后** → 进入「1. 连接与登录」流程

---
> **⚠️ AI Agent 必读**
详细API文档见 `api_reference.md`:
> **后续的流程具体使用示例和规范，在阅读完本文档对应节点后，要去读取api_reference.md里的使用示例，参照使用。**。
> **会话管理：**
> - session.public_key - 获取RSA公钥
> - session.login - 登录（RSA-OAEP加密）
> - session.logout - 登出
> - session.status - 查询会话状态

> **订单操作：**
> - order.send - 下单
> - order.cancel - 撤单

> **查询操作：**
> - query.account - 账户查询
> - query.positions - 持仓查询
> - query.orders - 委托查询
> - query.trades - 成交查询
> - query.instrument - 合约查询
> - query.status - 查询系统状态
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

**加密参数：**
- 算法：RSA-OAEP
- Hash：SHA-256
- MGF1 Hash：SHA-256

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

## 连接生命周期管理

### 登录保持

- **WebSocket 连接建立后无需每次请求都重新登录**
- 登录成功后，连接可以一直保持，直到：
  - 主动发送 `session.logout` 退出登录
  - WebSocket 连接断开
  - 服务端异常断开

### 心跳维持连接

- 客户端需定时发送心跳维持连接
- 建议心跳间隔：300秒（5分钟）
- 服务端连接超时：300秒（无响应则断开）

```javascript
// 心跳命令
ws.send('heartbeat');

// 服务端收到后不返回任何响应
```

### 退出登录

**重要：WebSocket 关闭前应发送退出登录命令**

```javascript
// 主动退出登录
ws.send('session.logout');

// 然后再关闭 WebSocket 连接
ws.close();
```

退出登录的好处：
- 释放会话资源
- 避免登录状态残留
- 服务端可以正确清理会话数据

### 重连流程

如果连接意外断开，需要重新建立连接：

1. 建立新的 WebSocket 连接
2. 获取公钥：`session.public_key`
3. 重新登录：`session.login --data=<加密凭据>`
4. 恢复心跳机制

## 关键注意事项

### order_sys_id 处理

- 服务返回的 `OrderSysID` 是12字符，可能含前导空格
- 服务端比较时会自动去除前导空格
- 发送撤单时无需手动处理空格

### 异步操作

以下方法为异步，成功时不立即返回结果：
- `order.send` - 等待 `order_update` 推送
- `order.cancel` - 等待 `order_update` 推送
- `query.*` - 等待推送结果

### 异步推送

服务端会主动推送以下事件，前端需监听WebSocket消息：

**order_update - 委托状态推送**
```javascript
// 格式
order_update --client_id=<客户端ID> --broker=<经纪公司> --user=<用户> --order_ref=<委托引用> --order_sys_id=<系统编号> --instrument=<合约代码> --exchange=<交易所> --direction=<买卖> --offset=<开平> --price=<委托价格> --volume=<委托数量> --traded_volume=<成交数量> --status=<状态> --status_msg=<状态描述> --insert_time=<申报时间> --update_time=<更新时间>

// 示例
order_update --client_id=ws_xxx --broker=1160 --user=600007 --order_ref=1 --order_sys_id=12345 --instrument=rb2501 --exchange=SHFE --direction=buy --offset=open --price=4000 --volume=1 --traded_volume=0 --status=submitted --status_msg=正在处理 --insert_time=093000 --update_time=093000
```

**字段说明：**
| 字段 | 说明 |
|------|------|
| client_id | 客户端ID |
| broker | 经纪公司代码 |
| user | 交易用户代码 |
| order_ref | 委托引用号（下单时返回） |
| order_sys_id | 系统编号 |
| instrument | 合约代码 |
| exchange | 交易所（SHFE/DCE/CZCE/CFFEX/SSE/SZSE） |
| direction | 方向（buy/sell） |
| offset | 开平（open/close/close_today/close_yesterday） |
| price | 委托价格 |
| volume | 委托数量 |
| traded_volume | 已成交数量 |
| status | 状态（submitted/confirmed/queued/filled/partially_filled/cancelled/rejected） |
| status_msg | 状态描述 |
| insert_time | 申报时间（格式：HHMMSS） |
| update_time | 更新时间（格式：HHMMSS） |

**status 状态说明：**
| 状态 | 说明 |
|------|------|
| submitted | 已提交（正在处理） |
| confirmed | 已确认（交易所接收） |
| queued | 排队中（等待成交） |
| filled | 完全成交 |
| partially_filled | 部分成交 |
| cancelled | 已撤单 |
| rejected | 拒绝 |

**trade_update - 成交推送**
```javascript
// 格式
trade_update --client_id=<客户端ID> --broker=<经纪公司> --user=<用户> --trade_id=<成交编号> --order_ref=<委托引用> --order_sys_id=<委托编号> --instrument=<合约代码> --exchange=<交易所> --direction=<买卖> --offset=<开平> --price=<成交价> --volume=<成交量> --trade_time=<成交时间>

// 示例
trade_update --client_id=ws_xxx --broker=1160 --user=600007 --trade_id=12345 --order_ref=1 --order_sys_id=67890 --instrument=rb2501 --exchange=SHFE --direction=buy --offset=open --price=4000 --volume=1 --trade_time=093015
```

**字段说明：**
| 字段 | 说明 |
|------|------|
| client_id | 客户端ID |
| broker | 经纪公司代码 |
| user | 交易用户代码 |
| trade_id | 成交编号（唯一） |
| order_ref | 委托引用号 |
| order_sys_id | 委托编号（用于撤单） |
| instrument | 合约代码 |
| exchange | 交易所（SHFE/DCE/CZCE/CFFEX/SSE/SZSE） |
| direction | 方向（buy/sell） |
| offset | 开平（open/close/close_today/close_yesterday） |
| price | 成交价格 |
| volume | 成交数量 |
| trade_time | 成交时间（格式：HHMMSS） |

**order_error - 下单错误推送**
```javascript
// 格式
order_error --error_id=<错误码> --error_msg=<错误信息> --instrument=<合约> --exchange=<交易所> --direction=<买卖> --offset=<开平> --price=<价格> --volume=<数量> --order_ref=<委托引用> --order_sys_id=<系统编号>

// 示例
order_error --error_id=3 --error_msg=不是可交易时间 --instrument=rb2501 --exchange=SHFE --direction=buy --offset=open --price=4000 --volume=1 --order_ref=1 --order_sys_id=
```

**字段说明：**
| 字段 | 说明 |
|------|------|
| error_id | 错误码 |
| error_msg | 错误信息描述 |
| instrument | 合约代码 |
| exchange | 交易所 |
| direction | 方向（buy/sell） |
| offset | 开平（open/close/closetoday/closeystd） |
| price | 委托价格 |
| volume | 委托数量 |
| order_ref | 委托引用号 |
| order_sys_id | 系统编号（下单失败时可能为空） |

**cancel_error - 撤单错误推送**
```javascript
// 格式
cancel_error --error_id=<错误码> --error_msg=<错误信息> --order_sys_id=<委托编号>

// 示例
cancel_error --error_id=25 --error_msg=撤单请求已被接受 --order_sys_id=12345
```

**常见服务错误码：**
| 错误码 | 说明 |
|--------|------|
| 0 | 成功（无错误） |
| 1 | 系统繁忙 |
| 2 | 未知错误 |
| 3 | 非可交易时间 |
| 4 | 合约不在交易状态 |
| 5 | 资金余额不足 |
| 6 | 超过持仓限制 |
| 7 | 超过交易所仓位限制 |
| 8 | 报单价格超出涨跌停板 |
| 10 | 无此权限 |
| 11 | 报单频率超限 |
| 13 | 客户不匹配 |
| 15 | 资金账号不匹配 |
| 16 | 不在交易时间 |
| 17 | 无此合约 |
| 19 | 报单类型不支持 |
| 21 | 报单价格不合法 |
| 22 | 数量不合法 |
| 23 | 无效的报单类型 |
| 24 | 报单已部成或全成 |
| 25 | 撤单请求已被接受 |
| 26 | 撤单失败 |
| 27 | 不允许平仓 |
| 28 | 超过单笔限额 |
| 29 | 超过日开仓限额 |

## 服务配置

| 配置项 | 默认值 |
|--------|--------|
| WebSocket地址 | ws://mcptest.zlqh.com:8765 |
| 心跳间隔 | 300秒 |