# Backpack API 操作指南

本文档提供了使用Backpack交易所API进行各种操作的详细指南。

## 初始化API客户端

```python
from funding_arbitrage_bot.exchanges.backpack_api import BackpackAPI

# 初始化API客户端
backpack_api = BackpackAPI(
    api_key="您的API密钥",
    api_secret="您的API密钥",
    base_url="https://api.backpack.exchange",  # 主网API地址
    ws_url="wss://ws.backpack.exchange",  # WebSocket地址
    logger=logger  # 可选，传入日志对象
)

# 启动价格数据流（实时价格更新）
await backpack_api.start_ws_price_stream()
```

## 价格和资金费率操作

### 获取价格

```python
# 获取单个交易对价格
price = await backpack_api.get_price("BTC_USDC_PERP")
print(f"比特币价格: {price}")

# 通过WebSocket获取的最新价格（需要先启动价格数据流）
btc_price = backpack_api.prices.get("BTC_USDC_PERP")
```

### 获取资金费率

```python
# 获取单个交易对的资金费率
funding_rate = await backpack_api.get_funding_rate("BTC_USDC_PERP")
print(f"比特币资金费率: {funding_rate}")

# 获取所有交易对的资金费率
all_funding_rates = await backpack_api.get_all_funding_rates()
for symbol, rate in all_funding_rates.items():
    print(f"{symbol} 资金费率: {rate}")
```

## 订单操作

### 下单

```python
# 市价买入
market_buy_order = await backpack_api.place_order(
    symbol="BTC_USDC_PERP",
    side="BUY",
    quantity=0.001,  # 购买0.001个BTC
    order_type="MARKET"  # 市价单
)

# 限价卖出
limit_sell_order = await backpack_api.place_order(
    symbol="ETH_USDC_PERP",
    side="SELL",
    quantity=0.01,  # 卖出0.01个ETH
    price=2000,  # 限价2000美元
    order_type="LIMIT"  # 限价单
)

# 可选参数：post_only（只做挂单）
post_only_order = await backpack_api.place_order(
    symbol="SOL_USDC_PERP",
    side="BUY",
    quantity=1,
    price=100,
    order_type="LIMIT",
    post_only=True  # 确保订单只作为maker
)

# 可选参数：reduce_only（只减仓不加仓）
reduce_only_order = await backpack_api.place_order(
    symbol="BTC_USDC_PERP",
    side="SELL",
    quantity=0.001,
    order_type="MARKET",
    reduce_only=True  # 确保此订单只会减少现有仓位
)
```

### 使用深度数据下单

```python
# 使用订单簿深度数据下单，确保更好的成交价格
order_with_depth = await backpack_api.place_order_with_depth(
    symbol="BTC_USDC_PERP",
    side="BUY",
    quantity=0.001,
    order_type="LIMIT",
    depth_tolerance=0.001  # 价格容忍度，基于最佳卖价
)
```

### 查询订单状态

```python
# 通过订单ID查询订单状态
order_status = await backpack_api.get_order_status("BTC_USDC_PERP", "order_id_here")
print(f"订单状态: {order_status}")

# 查询所有未成交订单
open_orders = await backpack_api.get_open_orders()
for order in open_orders:
    print(f"订单ID: {order['orderId']}, 交易对: {order['symbol']}, 类型: {order['type']}")
```

### 取消订单

```python
# 取消特定订单
cancel_result = await backpack_api.cancel_order("BTC_USDC_PERP", "order_id_here")

# 取消特定交易对的所有订单
cancel_all_result = await backpack_api.cancel_all_orders("BTC_USDC_PERP")

# 取消所有订单
cancel_all_result = await backpack_api.cancel_all_orders()
```

## 仓位操作

### 查询持仓

```python
# 查询单个交易对的持仓
btc_position = await backpack_api.get_position("BTC_USDC_PERP")
if btc_position:
    print(f"方向: {'多' if btc_position['quantity'] > 0 else '空'}, 数量: {abs(btc_position['quantity'])}")
    print(f"入场价: {btc_position['entryPrice']}, 未实现盈亏: {btc_position['unrealizedProfit']}")

# 查询所有持仓
all_positions = await backpack_api.get_positions()
for position in all_positions:
    if float(position['quantity']) != 0:  # 只显示有仓位的交易对
        print(f"{position['symbol']} - 方向: {'多' if float(position['quantity']) > 0 else '空'}, 数量: {abs(float(position['quantity']))}")
```

### 平仓

```python
# 全部平仓
if float(btc_position['quantity']) > 0:  # 如果是多仓
    close_result = await backpack_api.place_order(
        symbol="BTC_USDC_PERP", 
        side="SELL",  # 卖出平多
        quantity=abs(float(btc_position['quantity'])),
        order_type="MARKET",
        reduce_only=True
    )
else:  # 如果是空仓
    close_result = await backpack_api.place_order(
        symbol="BTC_USDC_PERP", 
        side="BUY",  # 买入平空
        quantity=abs(float(btc_position['quantity'])),
        order_type="MARKET",
        reduce_only=True
    )

# 部分平仓
partial_close = await backpack_api.place_order(
    symbol="BTC_USDC_PERP",
    side="SELL" if float(btc_position['quantity']) > 0 else "BUY",
    quantity=abs(float(btc_position['quantity'])) / 2,  # 平仓一半
    order_type="MARKET",
    reduce_only=True
)
```

## 订单簿（深度）数据

```python
# 获取订单簿深度数据
orderbook = await backpack_api.get_orderbook("BTC_USDC_PERP")

if orderbook:
    # 显示最优买价和卖价
    best_bid = orderbook["bids"][0] if orderbook["bids"] else None
    best_ask = orderbook["asks"][0] if orderbook["asks"] else None
    
    if best_bid:
        print(f"最优买价: {best_bid[0]}, 数量: {best_bid[1]}")
    if best_ask:
        print(f"最优卖价: {best_ask[0]}, 数量: {best_ask[1]}")
    
    # 计算买单总量（前5档）
    bid_volume = sum(float(bid[1]) for bid in orderbook["bids"][:5])
    # 计算卖单总量（前5档）
    ask_volume = sum(float(ask[1]) for ask in orderbook["asks"][:5])
    
    print(f"买单总量(前5档): {bid_volume}")
    print(f"卖单总量(前5档): {ask_volume}")
```

## 账户信息

```python
# 获取账户余额
balances = await backpack_api.get_balances()
for balance in balances:
    print(f"{balance['asset']} 可用余额: {balance['available']}, 冻结余额: {balance['locked']}")

# 获取账户信息
account_info = await backpack_api.get_account_info()
print(f"账户状态: {account_info['status']}")
print(f"手续费等级: {account_info['commissionTier']}")
```

## WebSocket订阅

```python
# 启动和监听价格流
await backpack_api.start_ws_price_stream()

# 获取最新价格（从缓存）
btc_price = backpack_api.prices.get("BTC_USDC_PERP", 0)  # 默认返回0如果价格不可用

# 如果需要通过回调处理价格更新
async def price_callback(symbol, price):
    print(f"价格更新: {symbol} = {price}")

# 注册回调
backpack_api.register_price_callback(price_callback)
```

## 错误处理

所有API方法都内置了错误处理机制，但在应用程序中，您仍然应该捕获可能的异常：

```python
try:
    # 尝试调用API
    result = await backpack_api.place_order(...)
    
    # 检查返回结果是否包含错误信息
    if "code" in result and result["code"] != 0:
        print(f"操作失败: {result['msg']}")
    else:
        print("操作成功!")
        
except Exception as e:
    print(f"发生异常: {e}")
```

## 实用提示

1. **交易对格式**：Backpack使用`BTC_USDC_PERP`这样的格式表示交易对，特别注意永续合约后缀`_PERP`。

2. **保持WebSocket连接**：使用`start_ws_price_stream()`方法可以维持WebSocket连接并持续更新价格数据，确保获取最新的市场价格。

3. **资金费率周期**：了解Backpack的资金费率结算周期（通常为每8小时一次），规划交易时机。

4. **日志级别**：调整日志级别以获取更详细的信息：
   ```python
   import logging
   logging.basicConfig(level=logging.DEBUG)  # 设置为DEBUG级别获取详细日志
   ```

5. **仓位同步**：定期调用`get_positions()`同步当前仓位信息，确保本地状态与交易所状态一致。

6. **WebSocket断线重连**：如果使用WebSocket长时间运行，建议实现断线重连机制：
   ```python
   while True:
       try:
           await backpack_api.start_ws_price_stream()
           # 保持连接运行
           await asyncio.sleep(3600)  # 每小时检查一次
       except Exception as e:
           print(f"WebSocket断开: {e}")
           await asyncio.sleep(5)  # 等待5秒后重连
   ```

## 示例：执行简单的套利策略

```python
async def simple_arbitrage_example():
    # 获取价格和资金费率
    btc_price = await backpack_api.get_price("BTC_USDC_PERP")
    btc_funding_rate = await backpack_api.get_funding_rate("BTC_USDC_PERP")
    
    # 基于资金费率决定方向
    if btc_funding_rate > 0.0001:  # 正资金费率超过某个阈值
        # 做空，因为可以收取资金费
        order = await backpack_api.place_order(
            symbol="BTC_USDC_PERP",
            side="SELL",  # 卖出/做空
            quantity=0.001,
            order_type="MARKET"
        )
        print(f"做空订单已提交: {order}")
    elif btc_funding_rate < -0.0001:  # 负资金费率超过某个阈值
        # 做多，因为可以收取资金费
        order = await backpack_api.place_order(
            symbol="BTC_USDC_PERP",
            side="BUY",  # 买入/做多
            quantity=0.001,
            order_type="MARKET"
        )
        print(f"做多订单已提交: {order}")
    else:
        print("资金费率在阈值范围内，不操作")
```

## 常见问题排查

1. **API密钥权限**：确保API密钥有足够的权限执行相应操作，特别是交易需要的写入权限。

2. **订单参数格式**：确保传递正确的参数格式，例如数量和价格应该是字符串或浮点数，而不是整数。

3. **价格精度**：不同交易对对价格和数量的精度要求不同，确保遵循Backpack的精度要求。

4. **交易量限制**：注意某些交易对可能有最小和最大交易量限制，确保下单数量在允许范围内。

5. **API请求限制**：Backpack可能对API请求频率有限制，如果收到限流错误，考虑减少请求频率或实现指数退避机制。

6. **订单类型大写**：确保订单类型（如"MARKET"、"LIMIT"）和交易方向（如"BUY"、"SELL"）使用全大写格式，这是Backpack API的要求。 