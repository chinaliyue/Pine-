//@version=2
//
// Title:   [STRATEGY][UL]QQE Cross v1.1
// Author:  JustUncleL
// Date:    22-Oct-2016
// Version: v1.1
//
// Description:
//  A trend following trading Strategy that uses fast QQE crosses with Moving Averages
//  for trend direction filtering. QQE or Qualitative Quantitative Estimation is based 
//  on the relative strength index, but uses a smoothing technique as an additional 
//  transformation. Three crosses can be selected (all selected by default): 
//    - RSI signal crossing ZERO (XZERO)
//    - RSI signal crossing Fast RSIatr line (XQQE)
//    - RSI signal exiting the RSI Threshhold Channel (BUY/SELL)
//  The (BUY/SELL) alerts can be optionally filtered by the Moving averages:
//    - For BUY alert the Close must be above the fast MA and 
//        fast MA (EMA8) > medium MA (EMA20) > slow MA (SMA50).
//    - For SELL alert the Close must be below the fast MA and
//        fast MA (EMA8) < medium MA (EMA20) < slow MA (SMA50).
//  and/or directional filter:
//    - For BUY alert the Close must be above the slow MA (SMA50) and the
//      directional MA (EMA20) must be green.
//    - For SELL alert the Close must be below the slow MA (SMA50) and the
//      directional MA (EMA20) must be red.
//. The XZERO and XQQE are not included in the filtering, they are used to indicate
//  pending BUY/SELL alerts, particularly the XZERO. 
//
//  This Strategy should work on any currency pair and most chart timeframes.
//  *** USE AT YOUR OWN RISK ***
//  
// 
//
// Mofidifications:
//  1.1 - Added Target Profit option, cleaned up the risk management code.
//        Changed Trade Close to EMA20 direction change instead of opposite BUY/SELL
//        signal, which will be earlier, this means stop loss setting should not be
//        required when an AutoTrader is available.
//        Modified code to prevent potential repaint issues.
//  1.0 - original
//
// References:
//  Some Code borrowed from:
//  - "Scalp Jockey - MTF MA Cross Visual Strategizer by JayRogers"
//  - "QQE MT4 by glaz"
//  - "Strategy Code Example by JayRogers"  
//  Inspiration from:
//  - http://www.forexstrategiesresources.com/binary-options-strategies-ii/189-aurora-binary-trading/
//  - http://www.forexstrategiesresources.com/metatrader-4-trading-systems-v/652-qqe-smoothed-trading/
//  - http://dewinforex.com/forex-indicators/qqe-indicator-not-quite-grail-but-accurately-defines-trend-and-flat.html
//
strategy(title='[STRATEGY][UL]QQE Cross v1.1', pyramiding=0, overlay=true, initial_capital=10000, calc_on_every_tick=false,
         currency=currency.USD,default_qty_type=strategy.percent_of_equity,default_qty_value=10)


// - INPUTS START
// Fast MA - type, source, length
type1   = input(defval="EMA", title="Fast MA Type: SMA, EMA, WMA, VWMA, SMMA, DEMA, TEMA, HullMA, ZEMA ( case sensitive )", type=string)
len1    = input(defval=8, title="Fast - Length", minval=1)
// Medium Fast MA - type, source, length
type2   = input(defval="EMA", title="Medium Fast MA Type: SMA, EMA, WMA, VWMA, SMMA, DEMA, TEMA, HullMA, ZEMA ( case sensitive )", type=string)
len2    = input(defval=20, title="Medium Fast - Length", minval=1)
// Slow MA - type, source, length
type3   = input(defval="SMA", title="Slow MA Type: SMA, EMA, WMA, VWMA, SMMA, DEMA, TEMA, HullMA, LSMA, ZEMA ( case sensitive )", type=string)
len3    = input(defval=50, title="Slow Length", minval=1)
//
// QQE rsi Length, Smoothing, fast ATR factor, source
RSILen  = input(6,title='RSI Length')
SF      = input(3,title='RSI Smoothing Factor')
QQE     = input(2.618,title='Fast QQE Factor')
threshhold = input(10, title="RSI Threshhold")
//
sQQEx   = input(false,title="Show QQE Signal crosses")
sQQEz   = input(false,title="Show QQE Zero crosses")
//
filter  = input(false,title="Use Moving Average Filter")
dfilter = input(true, title="Use Trend Directional Filter" )
RSIsrc  = input(close,title="Source")
srcclose= RSIsrc

// - INPUTS END

// - FUNCTIONS

// Returns MA input selection variant, default to SMA if blank or typo.
variant(type, src, len) =>
    v1 = sma(src, len)                                                  // Simple
    v2 = ema(src, len)                                                  // Exponential
    v3 = wma(src, len)                                                  // Weighted
    v4 = vwma(src, len)                                                 // Volume Weighted
    v5 = na(v5[1]) ? sma(src, len) : (v5[1] * (len - 1) + src) / len    // Smoothed
    v6 = 2 * v2 - ema(v2, len)                                          // Double Exponential
    v7 = 3 * (v2 - ema(v2, len)) + ema(ema(v2, len), len)               // Triple Exponential
    v8 = wma(2 * wma(src, len / 2) - wma(src, len), round(sqrt(len)))   // Hull
    ema1 = ema(src, len)
    ema2 = ema(ema1, len)
    v10 = ema1+(ema1-ema2)                                              // Zero Lag Exponential
    // return variant, defaults to SMA if input invalid.
    type=="EMA"?v2 : type=="WMA"?v3 : type=="VWMA"?v4 : type=="SMMA"?v5 : type=="DEMA"?v6 : type=="TEMA"?v7 : type=="HullMA"?v8 : type=="ZEMA"?v10 : v1

// - FUNCTIONS END

// - Fast ATR QQE
//
Wilders_Period = RSILen * 2 - 1
//
Rsi = rsi(RSIsrc,RSILen)
RSIndex = ema(Rsi, SF)
AtrRsi = abs(RSIndex[1] - RSIndex)
MaAtrRsi = ema(AtrRsi, Wilders_Period)
DeltaFastAtrRsi = ema(MaAtrRsi,Wilders_Period) * QQE
//
newshortband=  RSIndex + DeltaFastAtrRsi
newlongband= RSIndex - DeltaFastAtrRsi
longband=RSIndex[1] > longband[1] and RSIndex > longband[1] ? max(longband[1],newlongband) : newlongband
shortband=RSIndex[1] < shortband[1] and  RSIndex < shortband[1] ? min(shortband[1],newshortband) : newshortband
trend=cross(RSIndex, shortband[1])? 1 : cross(longband[1], RSIndex) ? -1 : nz(trend[1],1)
FastAtrRsiTL = trend==1 ? longband : shortband


// - SERIES VARIABLES
// MA's
ma_fast    = variant(type1, srcclose, len1)
ma_medium  = variant(type2, srcclose, len2)
ma_slow    = variant(type3, srcclose, len3)
// Get Direction From Medium Moving Average
direction = rising(ma_medium,3) ? 1 : falling(ma_medium,3) ? -1 : 0
//
// Find all the QQE Crosses
QQExshort = sQQEx and crossover(FastAtrRsiTL, RSIndex)
QQExlong  = sQQEx and crossunder(FastAtrRsiTL, RSIndex)
// Zero cross
QQEzlong = sQQEz and crossover(RSIndex,50)
QQEzshort  = sQQEz and crossunder(RSIndex,50)
//  
// Thresh Hold channel Crosses give the BUY/SELL alerts.
QQEclong = RSIndex>(50+threshhold) ? na(QQEclong[1]) ? 1 : QQEclong[1]+1 : 0
QQEcshort = RSIndex<(50-threshhold) ? na(QQEcshort[1]) ? 1 : QQEcshort[1]+1 : 0

//
// Check Filtering.
QQEflong = (not filter or (srcclose>ma_fast and ma_medium>ma_slow and ma_fast>ma_medium)) and 
  (not dfilter or (direction>0 and srcclose>ma_slow))
QQEfshort = (not filter or (srcclose<ma_fast and ma_medium<ma_slow and ma_fast<ma_medium)) and
  (not dfilter or (direction<0 and srcclose<ma_slow))
//
// Get final BUY / SELL alert determination
buy = QQEclong>0 and QQEflong ? na(buy[1]) ? 1 : buy[1]+1 : 0
sell= QQEcshort>0 and QQEfshort ? na(sell[1]) ? 1 : sell[1]+1 : 0

// - SERIES VARIABLES END

// - PLOTTING
// Ma's
plot(ma_fast, title="MA Fast", color=olive, linewidth=2, transp=20)
plot(ma_medium, title="MA Medium Fast", color=direction<0?red:green, linewidth=3, transp=0)
plot(ma_slow, title="MA Slow", color=blue, linewidth=2, transp=20)
// QQE crosses
plotshape(QQExlong and buy!=1, title="QQE Cross Over", style=shape.triangleup, location=location.belowbar, text="XQQE", color=blue, transp=20, size=size.tiny)
plotshape(QQExshort and sell!=1, title="QQE Cross Under", style=shape.triangledown, location=location.abovebar, text="XQQE", color=black, transp=20, size=size.tiny)
// Signal crosses zero line
plotshape(QQEzlong and buy!=1 and not QQExlong, title="QQE Zero Cross Over", style=shape.triangleup, location=location.belowbar, text="XZERO", color=aqua, transp=20, size=size.tiny)
plotshape(QQEzshort and sell!=1 and not QQExshort, title="QQE Zero Cross Under", style=shape.triangledown, location=location.abovebar, text="XZERO", color=fuchsia, transp=20, size=size.tiny)

// - PLOTTING END

// Create alert for cross, shunt back 1 if source is not 'open', this should prevent repaint issue.
//shunt = RSIsrc == open ? 0 : 1
shunt = 0
c_alert = (buy[shunt]==1 or sell[shunt]==1)
//alertcondition(c_alert, title="QQECROSS Alert", message="QQECROSS Alert")
// show only when alert condition is met and bar closed.
plotshape(c_alert,title= "Alert Indicator Closed", location=location.bottom, color=sell[shunt]==1?red:green, transp=0, style=shape.circle)


//  Strategy: (Thanks to JayRogers)
// === STRATEGY RELATED INPUTS ===
//tradeInvert     = input(defval = false, title = "Invert Trade Direction?")
// the risk management inputs
inpTakeProfit   = input(defval = 0, title = "Take Profit Points", minval = 0)
inpStopLoss     = input(defval = 0, title = "Stop Loss Points", minval = 0)
inpTrailStop    = input(defval = 100, title = "Trailing Stop Loss Points", minval = 0)
inpTrailOffset  = input(defval = 0, title = "Trailing Stop Loss Offset Points", minval = 0)

// === RISK MANAGEMENT VALUE PREP ===
// if an input is less than 1, assuming not wanted so we assign 'na' value to disable it.
useTakeProfit   = inpTakeProfit  >= 1 ? inpTakeProfit  : na
useStopLoss     = inpStopLoss    >= 1 ? inpStopLoss    : na
useTrailStop    = inpTrailStop   >= 1 ? inpTrailStop   : na
useTrailOffset  = inpTrailOffset >= 1 ? inpTrailOffset : na

// === STRATEGY - LONG POSITION EXECUTION ===
strategy.entry(id = "Buy", long = true, when = buy[shunt]==1 )// use function or simple condition to decide when to get in
strategy.close(id = "Buy", when = direction[shunt]!=direction[shunt+1])// ...and when to get out

// === STRATEGY - SHORT POSITION EXECUTION ===
strategy.entry(id = "Sell", long = false, when = sell[shunt]==1)
strategy.close(id = "Sell", when = direction[shunt]!=direction[shunt+1])

// === STRATEGY RISK MANAGEMENT EXECUTION ===
// finally, make use of all the earlier values we got prepped
strategy.exit("Exit Buy", from_entry = "Buy", profit = useTakeProfit, loss = useStopLoss, trail_points = useTrailStop, trail_offset = useTrailOffset)
strategy.exit("Exit Sell", from_entry = "Sell", profit = useTakeProfit, loss = useStopLoss, trail_points = useTrailStop, trail_offset = useTrailOffset)

//eof
