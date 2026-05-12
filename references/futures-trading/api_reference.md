# CTP交易MCP服务器API参考

## 概述

CTP (China Futures OrderTrading System) MCP服务器是一个基于WebSocket的期货交易服务，提供下单、撤单、查询等交易功能。

## 服务地址

- WebSocket: `ws://mcptest.zlqh.com:8765`

---

## 认证方式

### session.public_key

获取服务端RSA公钥。

**指令:**
```
session.public_key
```

**返回值:**
```
[RESP] success=true --key=-----BEGIN PUBLIC KEY-----\nMIIBIjANBgkqhkiG9w0BAQEFAAOCAQ8AMIIBCgKCAQEA...\n-----END PUBLIC KEY-----\n
```

### session.login

登录到交易服务器。使用 RSA 加密方式。

**指令:**
```
session.login --data=<RSA加密的Base64字符串>
```

**参数:**
- `data`: "用户名&密码" 用 RSA 公钥加密后的 Base64 字符串

**登录流程:**
1. 先调用 `session.public_key` 获取RSA公钥
2. 将用户名和密码拼接成 "用户名&密码" 格式（如 `600007&123456`）
3. 使用获取的RSA公钥加密该字符串（RSA/OAEP/SHA-256）
4. 将加密结果（Base64编码）作为 `data` 参数发送

**前端JavaScript示例:**
```javascript
// 1. 获取公钥
await sendCommand('session.public_key');
// 收到: public_key --key=-----BEGIN PUBLIC KEY-----\n...

// 2. 加密登录
async function encryptedLogin(publicKeyPem, username, password) {
    const publicKey = await crypto.subtle.importKey(
        "spki",
        atob(publicKeyPem.replace(/-----BEGIN PUBLIC KEY-----|-----END PUBLIC KEY-----|\n/g, '')),
        { name: "RSA-OAEP", hash: "SHA-256" },
        false,
        ["encrypt"]
    );
    
    const encoder = new TextEncoder();
    const data = await crypto.subtle.encrypt(
        { name: "RSA-OAEP" },
        publicKey,
        encoder.encode(username + '&' + password)
    );
    
    return btoa(String.fromCharCode(...new Uint8Array(data)));
}

const encryptedData = await encryptedLogin(publicKey, '600007', '123456');
await sendCommand(`session.login --data=${encryptedData}`);
```

### session.logout

登出交易服务器。

```
session.logout
```

---

## 订单操作

### order.send

下单。

```
order.send --instrument=rb2501 --exchange=SHFE --direction=buy --offset=open --price=4000 --volume=1
```

**参数:**
- `instrument`: 合约代码 (如 rb2501, IF2503)
- `exchange`: 交易所代码 (SHFE, DCE, CZCE, CFFEX, INE)
- `direction`: 买卖方向 (`buy` 或 `sell`)
- `offset`: 开平 (`open`, `close`, `closetoday`, `closehistory`)
- `price`: 价格 (0为市价)
- `volume`: 数量

**返回值:**
```json
{"success": true, "order_ref": "1", "order_sys_id": "       12345"}
```

### order.cancel

撤单。

```
order.cancel --order_sys_id=12345
```

**参数:**
- `order_sys_id`: 报单系统编号

**注意:** `order_sys_id` 比较时会自动去除前导空格。

---

## 查询操作

### query.account

查询账户资金。

```
query.account
```

**返回值:**
```json
{
  "Balance": "1000000.00",
  "Available": "950000.00",
  "PositionProfit": "5000.00",
  "Commission": "120.00"
}
```

### query.positions

查询所有持仓。

```
query.positions
```

**返回值:**
```json
{
  "positions": [
    {
      "instrument": "rb2501",
      "exchange": "SHFE",
      "direction": "buy",
      "volume": "10",
      "position_cost": "400000.00",
      "position_profit": "5000.00"
    }
  ]
}
```

### query.orders

查询所有委托。

```
query.orders
```

**返回值:**
```json
{
  "orders": [
    {
      "OrderRef": "1",
      "OrderSysID": "       12345",
      "InstrumentID": "rb2501",
      "ExchangeID": "SHFE",
      "Direction": "buy",
      "OrderStatus": "0",
      "VolumeTraded": "1",
      "VolumeTotalOriginal": "0"
    }
  ]
}
```

### query.order

查询指定委托。

```
query.order --order_ref=1
```

### query.trades

查询成交记录。

```
query.trades --instrument=rb2501
```

### query.instrument

查询合约信息。

```
query.instrument --instrument=rb2501 --exchange=SHFE
```

---

## 推送消息

服务器会主动推送以下消息:

### order_update

报单状态更新。

```
[PUSH] order_update --order_ref=1 --order_sys_id=12345 --status=filled --instrument=rb2501
```

### trade_update

成交回报（异步推送）。

```
[PUSH] trade_update --instrument=rb2501 --exchange=SHFE --direction=buy --offset=open --price=4000 --volume=1 --trade_id=12345 --order_sys_id=67890 --trade_time=20240101123045
```

**字段说明:**

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

### login_success / login_error

登录结果推送。

---

## 心跳机制

```
// 前端每 300 秒（5分钟）发送一次心跳
heartbeat
```

**注意事项:**
- 连接超时为 300 秒
- 如果服务端超过 300 秒未收到任何消息，会断开连接
- 建议前端每 280 秒发送一次心跳以留有余量

---

## 错误处理

操作失败时返回:

```json
{"success": false, "error": "错误信息"}
```

常见错误:
- `请先登录`: 需要先调用 `session.login`
- `缺少order_sys_id`: 撤单时必须提供 order_sys_id
- `无法获取OrderSysID`: 订单信息不完整
- `未找到订单`: 指定的订单不存在

---

## 数据结构

### 字段说明

| 字段 | 说明 |
|------|------|
| OrderRef | 报单引用，CThostFtdcOrderField::OrderRef，带前导空格 |
| OrderSysID | 报单系统编号，12字符，可能含前导空格 |
| InstrumentID | 合约代码 |
| ExchangeID | 交易所代码 |
| Direction | 方向: '0'=买入, '1'=卖出 |
| OrderStatus | 状态: '0'=全部成交, '1'=部分成交还在队列中, '3'=部分成交不在队列中, '5'=指令接受, 'a'=未知, 'c'=已撤 |
| CombOffsetFlag | 开平标志: '0'=开仓, '1'=平仓, '3'=平今, '4'=平昨 |
