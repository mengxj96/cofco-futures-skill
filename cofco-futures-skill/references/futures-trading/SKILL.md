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

### 1. 连接与登录

```javascript
// WebSocket连接
const ws = new WebSocket('ws://82.156.185.62:8765');

// 登录
ws.send('session.login --user=xxxx --password=xxxx');

// 等待登录成功推送
```

### 2. 下单流程

```javascript
// 发送下单指令
ws.send('order.send --instrument=rb2501 --exchange=SHFE --direction=buy --offset=open --price=4000 --volume=1');

// 等待 order_update 推送确认
// 收到 order_update --status=filled 表示成交
```

### 3. 撤单流程

```javascript
// 通过 order_sys_id 撤单
ws.send('order.cancel --order_sys_id=xxxx');
```

### 4. 查询流程

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

## API参考

详细API文档见 `references/api_reference.md`:
- session.login - 登录
- session.logout - 登出
- order.send - 下单
- order.cancel - 撤单
- query.account - 账户查询
- query.positions - 持仓查询
- query.orders - 委托查询
- query.trades - 成交查询
- query.instrument - 合约查询
