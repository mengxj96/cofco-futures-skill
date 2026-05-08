# CTP交易MCP服务器API参考

## 概述

CTP (China Futures OrderTrading System) MCP服务器是一个基于WebSocket的期货交易服务，提供下单、撤单、查询等交易功能。

## 服务地址

- WebSocket: `ws://82.156.185.62:8765`

---

## 认证方式

### session.login

登录到交易服务器。

```
session.login --user=xxxx --password=xxxx
```

**参数:**
- `user`: 交易账号
- `password`: 交易密码

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
      "OrderSysID": "xxxx",
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

成交回报。

```
[PUSH] trade_update --trade_id=456 --order_ref=1 --instrument=rb2501 --direction=buy --volume=1 --price=4000
```

### login_success / login_error

登录结果推送。

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
