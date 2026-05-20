# 期货模拟交易MCP服务器API参考

## 概述

期货模拟交易服务基于WebSocket，提供下单、撤单、查询等交易功能。

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
5. 服务端用私钥解密并用 `&` 分割出用户名和密码

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

### session.status

查询当前会话状态。

```
session.status
```

**返回值:**
```json
{
  "sessions": "ws_xxx,ws_yyy",
  "logged_in": "true",
  "broker": "1160",
  "user": "600007",
  "login_time": "20260518100000",
  "trading_day": "20260518"
}
```

### query.status

查询系统状态。

```
query.status
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
- `offset`: 开平 (`open`, `close`, `close_today`, `close_yesterday`)
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
[PUSH] order_update --client_id=ws_xxx --broker=1160 --user=600007 --order_ref=1 --order_sys_id=12345 --instrument=rb2501 --exchange=SHFE --direction=buy --offset=open --price=4000 --volume=1 --traded_volume=0 --status=filled --status_msg=已成交 --insert_time=093000 --update_time=093001
```

**字段说明:**

| 字段 | 说明 |
|------|------|
| client_id | 客户端ID |
| broker | 经纪公司代码 |
| user | 交易用户代码 |
| order_ref | 委托引用号 |
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

### trade_update

成交回报（异步推送）。

```
[PUSH] trade_update --client_id=ws_xxx --broker=1160 --user=600007 --trade_id=12345 --order_ref=1 --order_sys_id=67890 --instrument=rb2501 --exchange=SHFE --direction=buy --offset=open --price=4000 --volume=1 --trade_time=093015
```

**字段说明:**

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

### login_success / login_error

登录结果推送。

```
[PUSH] login_success --broker=1160 --client_id=ws_xxx --status=logged_in --trading_day=20260518
[PUSH] login_error --error_id=3 --error_msg=不存在该用户
```

### order_error

下单错误推送（异步）。

```
[PUSH] order_error --error_id=3 --error_msg=非可交易时间 --instrument=rb2501 --exchange=SHFE --direction=buy --offset=open --price=4000 --volume=1 --order_ref=1 --order_sys_id=
```

**字段说明：**

| 字段 | 说明 |
|------|------|
| error_id | 错误码 |
| error_msg | 错误信息描述 |
| instrument | 合约代码 |
| exchange | 交易所代码 |
| direction | 方向（buy/sell） |
| offset | 开平（open/close/close_today/close_yesterday） |
| price | 委托价格 |
| volume | 委托数量 |
| order_ref | 委托引用号 |
| order_sys_id | 系统编号（下单失败时可能为空） |

### cancel_error

撤单错误推送（异步）。

```
[PUSH] cancel_error --error_id=25 --error_msg=撤单请求已被接受 --order_sys_id=12345
```

**字段说明：**

| 字段 | 说明 |
|------|------|
| error_id | 错误码 |
| error_msg | 错误信息描述 |
| order_sys_id | 被撤委托的系统编号 |

### 常见错误码对照表

| 错误码 | 说明 | 处理建议 |
|--------|------|----------|
| 0 | 成功 | 无需处理 |
| 1 | 系统繁忙 | 重试 |
| 2 | 未知错误 | 联系技术支持 |
| 3 | 非可交易时间 | 等待交易时间 |
| 4 | 合约不在交易状态 | 检查合约交易时间 |
| 5 | 资金余额不足 | 充值或减少下单量 |
| 6 | 超过持仓限制 | 检查持仓 |
| 7 | 超过交易所仓位限制 | 调整仓位 |
| 8 | 报单价格超出涨跌停板 | 调整价格 |
| 10 | 无此权限 | 检查账户权限 |
| 11 | 报单频率超限 | 降低下单频率 |
| 13 | 客户不匹配 | 检查账户信息 |
| 15 | 资金账号不匹配 | 检查账户信息 |
| 16 | 不在交易时间 | 等待交易时间 |
| 17 | 无此合约 | 检查合约代码 |
| 19 | 报单类型不支持 | 调整报单类型 |
| 21 | 报单价格不合法 | 检查价格 |
| 22 | 数量不合法 | 检查数量 |
| 23 | 无效的报单类型 | 检查报单类型 |
| 24 | 报单已部成或全成 | 无需撤单 |
| 25 | 撤单请求已被接受 | 等待确认 |
| 26 | 撤单失败 | 检查订单状态 |
| 27 | 不允许平仓 | 检查持仓和规则 |
| 28 | 超过单笔限额 | 减少下单量 |
| 29 | 超过日开仓限额 | 等待或平仓 |

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
