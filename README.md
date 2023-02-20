//@version=5
strategy("My Strategy", overlay=true)

// 计算ATR
atrVal = ta.atr(14)

// 计算RSI
rsiVal = ta.rsi(close, 14)

// 设置做多和做空条件
longCondition = rsiVal < 30
shortCondition = rsiVal > 70

// 绘制K线
plot(close, color=color.black, linewidth=2)

// 交易逻辑
if (longCondition)
    strategy.entry("My Long Entry Id", strategy.long, comment="My Long Entry")
if (shortCondition)
    strategy.entry("My Short Entry Id", strategy.short, comment="My Short Entry")

// 计算偏移量
stopLoss = atrVal
takeProfit = 2 * atrVal

// 退出逻辑
if strategy.position_size > 0
    strategy.exit("My Long Exit Id", "My Long Entry Id", stop=stopLoss, limit=close - takeProfit, comment="My Long Exit")
if strategy.position_size < 0
    strategy.exit("My Short Exit Id", "My Short Entry Id", stop=stopLoss, limit=close + takeProfit, comment="My Short Exit")
