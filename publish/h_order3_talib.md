
```html
<!DOCTYPE html>
<html lang="en">
<head>
  <meta charset="UTF-8" />
  <meta name="viewport" content="width=device-width, initial-scale=1.0" />
  <title>Live NIFTY Tick Chart (IST)</title>
  <script src="https://cdn.jsdelivr.net/gh/parth-royale/cdn@main/lightweight-charts.standalone.production.js"></script>
  <style>
    .timeframe-container {
      margin: 10px 0;
      text-align: center;
    }
    .tf-button {
      padding: 5px 10px;
      margin: 2px;
      cursor: pointer;
      border: 1px solid #ccc;
      background-color: #f8f8f8;
      border-radius: 3px;
    }
    .active {
      background-color: #4CAF50;
      color: white;
    }
    
    /* Chart Container */
    #chart {
      position: relative;
      height: 600px; /* Set a fixed height for the chart */
      width: 100%; /* Ensure it takes the full width of its parent */
      background-color: #131722; /* Match TradingView's dark theme */
    }
      
    .floating-rectangle {
      position: absolute;
      top: 10px;
      right: 10px;
      width: 300px;
      height: 500px;
      background-color: rgba(0, 128, 255, 0.8);
      border: 2px solid #0056b3;
      border-radius: 4px;
      display: flex;
      flex-direction: column;
      color: white;
      font-family: Arial, sans-serif;
      font-size: 12px;
      cursor: grab;
      z-index: 1000;
    }
    .log-section {
      flex: 1;
      overflow-y: auto;
      padding: 10px;
      border-bottom: 1px solid rgba(255,255,255,0.2);
    }
    #chartUpdateLog {
      background-color: rgba(0,0,0,0.2);
    }
    #tradeLog {
      background-color: rgba(0,0,0,0.1);
    }
    #profitLog {
      background-color: rgba(0,0,0,0.05);
    }
    .log-entry {
      margin-bottom: 5px;
      word-wrap: break-word;
    }
    .profit-positive {
      color: #081ff2; /* Darker green */
    }
    .profit-negative {
      color: #a5fb06; /* Darker red */
    }
    
    /* Real-Time Profit/Loss */
    .real-time-pl {
      font-size: 10px;
      margin-top: 2px;
    }
  </style>
</head>
<body>
  <h1>Live NIFTY Tick Chart (IST)</h1>
  
  <div class="timeframe-container">
    <!-- Buttons for second-based timeframes -->
    <button class="tf-button active" data-timeframe="5s">5 Second</button>
    <button class="tf-button" data-timeframe="10s">10 Second</button>
    <button class="tf-button" data-timeframe="15s">15 Second</button>
    <button class="tf-button" data-timeframe="30s">30 Second</button>
    <button class="tf-button" data-timeframe="45s">45 Second</button>

    <!-- Buttons for minute-based timeframes -->
    <button class="tf-button" data-timeframe="1">1 Minute</button>
    <button class="tf-button" data-timeframe="3">3 Minute</button>
    <button class="tf-button" data-timeframe="5">5 Minute</button>
    <button class="tf-button" data-timeframe="10">10 Minute</button>
    <button class="tf-button" data-timeframe="15">15 Minute</button>
    <button class="tf-button" data-timeframe="30">30 Minute</button>
    <button class="tf-button" data-timeframe="60">1 Hour</button>
    
    <!-- Buttons for extended timeframes -->
    <button class="tf-button" data-timeframe="2h">2 Hour</button>
    <button class="tf-button" data-timeframe="4h">4 Hour</button>
    <button class="tf-button" data-timeframe="1d">1 Day</button>
    <button class="tf-button" data-timeframe="1w">1 Week</button>
    <button class="tf-button" data-timeframe="1month">1 Month</button>
  </div>

  <!-- Trading Simulator UI -->
  <div>
    <button id="l">Long</button>
    <button id="s">Short</button>
    <button id="p">Pause</button>
  </div>
  
  <div id="chart"></div>
  
  <div id="floatingRectangle" class="floating-rectangle">
    <div id="chartUpdateLog" class="log-section">Chart Updates</div>
    <div id="tradeLog" class="log-section">Trade Logs</div>
    <div id="profitLog" class="log-section">Overall Profit</div>
  </div>
  
  <script>
    // ----------------------- CHART SETUP -----------------------
    const chart = LightweightCharts.createChart(document.getElementById('chart'), {
      width: window.innerWidth,
      height: window.innerHeight - 150,
      priceScale: { borderColor: '#cccccc' },
      timeScale: { 
        borderColor: '#cccccc', 
        timeVisible: true, 
        secondsVisible: true,
      }
    });

    // MAIN PANE: Add candlestick series with custom colors.
    const candlestickSeries = chart.addCandlestickSeries({
      upColor: '#4caf50',
      downColor: '#f44336',
      borderUpColor: '#4caf50',
      borderDownColor: '#f44336',
      wickUpColor: '#4caf50',
      wickDownColor: '#f44336'
    });

    // ---- Add Indicator Series ----
    // SMA overlays:
    const sma12Series = chart.addLineSeries({
      title: 'SMA 12',
      color: '#FFEB3B',
      lineWidth: 2
    });
    const sma24Series = chart.addLineSeries({
      title: 'SMA 24',
      color: '#FF0000',
      lineWidth: 2
    });
    const sma50Series = chart.addLineSeries({
      title: 'SMA 50',
      color: '#000000',
      lineWidth: 3
    });
    const sma200Series = chart.addLineSeries({
      title: 'SMA 200',
      color: '#0000FF',
      lineWidth: 3
    });
    
    // RSI (placed on a separate pane if your library version supports it)
    const rsiSeries = chart.addLineSeries({
      title: 'RSI 14',
      color: '#FF6D00',
      lineWidth: 2,
      pane: 1
    });
    
    // MACD on another pane:
    const macdSeries = chart.addLineSeries({
      title: 'MACD Line',
      color: '#FF6D00',
      lineWidth: 2,
      pane: 2
    });
    const signalSeries = chart.addLineSeries({
      title: 'Signal Line',
      color: '#00C853',
      lineWidth: 2,
      pane: 2
    });
    const histogramSeries = chart.addHistogramSeries({
      title: 'MACD Histogram',
      color: '#E53935',
      pane: 2
    });

    // ----------------------- DATA & TIMEFRAME SETUP -----------------------
    let originalData = [];
    let currentTimeframe = '5s';
    let dataArrays = {};
    let activeCallback = null;
    let activeResolution = '5s';

    // Original data array
    let historicalData = [
      { timestamp: '2024-02-15T09:15:00Z', open: 100, high: 102, low: 98, close: 100 },
      { timestamp: '2024-02-15T09:15:05Z', open: 101, high: 113, low: 99, close: 110 },
      { timestamp: '2024-02-15T09:15:10Z', open: 102, high: 124, low: 100, close: 121 },
      { timestamp: '2024-02-15T09:15:15Z', open: 103, high: 136, low: 101, close: 133.1 },
      { timestamp: '2024-02-15T09:15:20Z', open: 104, high: 150, low: 102, close: 146.41 },
      { timestamp: '2024-02-15T09:15:25Z', open: 105, high: 165, low: 103, close: 161.05 },
      { timestamp: '2024-02-15T09:15:30Z', open: 106, high: 181, low: 104, close: 177.16 },
      { timestamp: '2024-02-15T09:15:35Z', open: 107, high: 199, low: 105, close: 194.87 },
      { timestamp: '2024-02-15T09:15:40Z', open: 108, high: 219, low: 106, close: 214.36 },
      { timestamp: '2024-02-15T09:15:45Z', open: 109, high: 241, low: 107, close: 235.79 },
      { timestamp: '2024-02-15T09:15:50Z', open: 110, high: 265, low: 108, close: 259.37 },
      { timestamp: '2024-02-15T09:15:55Z', open: 111, high: 291, low: 109, close: 285.31 },
      { timestamp: '2024-02-15T09:16:00Z', open: 112, high: 320, low: 110, close: 313.84 },
      { timestamp: '2024-02-15T09:16:05Z', open: 113, high: 352, low: 111, close: 345.22 },
      { timestamp: '2024-02-15T09:16:10Z', open: 114, high: 387, low: 112, close: 379.75 },
      { timestamp: '2024-02-15T09:16:15Z', open: 115, high: 426, low: 113, close: 417.72 },
      { timestamp: '2024-02-15T09:16:20Z', open: 116, high: 468, low: 114, close: 459.49 },
      { timestamp: '2024-02-15T09:16:25Z', open: 117, high: 515, low: 115, close: 505.44 },
      { timestamp: '2024-02-15T09:16:30Z', open: 118, high: 567, low: 116, close: 555.99 },
      { timestamp: '2024-02-15T09:16:35Z', open: 119, high: 623, low: 117, close: 611.58 },
      { timestamp: '2024-02-15T09:16:40Z', open: 120, high: 686, low: 118, close: 672.74 },
      { timestamp: '2024-02-15T09:16:45Z', open: 121, high: 754, low: 119, close: 740.02 },
      { timestamp: '2024-02-15T09:16:50Z', open: 122, high: 830, low: 120, close: 814.02 },
      { timestamp: '2024-02-15T09:16:55Z', open: 123, high: 913, low: 121, close: 895.42 },
      { timestamp: '2024-02-15T09:17:00Z', open: 124, high: 1004, low: 122, close: 984.96 },
      { timestamp: '2024-02-15T09:17:05Z', open: 125, high: 1105, low: 123, close: 1083.46 },
      { timestamp: '2024-02-15T09:17:10Z', open: 126, high: 1215, low: 124, close: 1191.80 },
      { timestamp: '2024-02-15T09:17:15Z', open: 127, high: 1337, low: 125, close: 1310.98 },
      { timestamp: '2024-02-15T09:17:20Z', open: 128, high: 1471, low: 126, close: 1442.08 },
      { timestamp: '2024-02-15T09:17:25Z', open: 129, high: 1618, low: 127, close: 1586.29 },
      { timestamp: '2024-02-15T09:17:30Z', open: 130, high: 1780, low: 128, close: 1744.92 },
      { timestamp: '2024-02-15T09:17:35Z', open: 131, high: 1958, low: 129, close: 1919.41 },
      { timestamp: '2024-02-15T09:17:40Z', open: 132, high: 2153, low: 130, close: 2111.35 },
      { timestamp: '2024-02-15T09:17:45Z', open: 133, high: 2369, low: 131, close: 2322.48 },
      { timestamp: '2024-02-15T09:17:50Z', open: 134, high: 2606, low: 132, close: 2554.73 },
      { timestamp: '2024-02-15T09:17:55Z', open: 135, high: 2866, low: 133, close: 2810.21 },
      { timestamp: '2024-02-15T09:18:00Z', open: 136, high: 3153, low: 134, close: 3091.23 },
      { timestamp: '2024-02-15T09:18:05Z', open: 137, high: 3468, low: 135, close: 3400.35 },
      { timestamp: '2024-02-15T09:18:10Z', open: 138, high: 3815, low: 136, close: 3740.38 },
      { timestamp: '2024-02-15T09:18:15Z', open: 139, high: 4196, low: 137, close: 4114.42 },
      { timestamp: '2024-02-15T09:18:20Z', open: 140, high: 4616, low: 138, close: 4525.86 },
      { timestamp: '2024-02-15T09:18:25Z', open: 141, high: 5078, low: 139, close: 4978.45 },
      { timestamp: '2024-02-15T09:18:30Z', open: 142, high: 5586, low: 140, close: 5476.30 }
    ];

    // Convert timestamps to Unix seconds
    historicalData = historicalData.map(data => ({
      time: new Date(data.timestamp).getTime() / 1000,
      open: data.open,
      high: data.high,
      low: data.low,
      close: data.close
    }));

    // Convert timeframe string to milliseconds
    function getTimeframeMs(timeframe) {
      if (timeframe.endsWith('s')) {
        return parseInt(timeframe) * 1000;
      } else if (timeframe.endsWith('h')) {
        return parseInt(timeframe) * 60 * 60 * 1000;
      } else if (timeframe === '1d') {
        return 24 * 60 * 60 * 1000;
      } else if (timeframe === '1w') {
        return 7 * 24 * 60 * 60 * 1000;
      } else if (timeframe === '1month') {
        return 30 * 24 * 60 * 60 * 1000;
      } else {
        // Default to minutes
        return parseInt(timeframe) * 60 * 1000;
      }
    }

    // Resample data into bars for the given timeframe.
    function resampleData(data, timeframe) {
      if (data.length === 0) return [];
      
      const resampledData = [];
      let currentBar = null;
      const timeframeMs = getTimeframeMs(timeframe);
      const timeframeSec = timeframeMs / 1000; // data time is in seconds
      
      // Ensure data is sorted
      const sortedData = [...data].sort((a, b) => a.time - b.time);
      const firstDataTime = sortedData[0].time;
      
      for (const row of sortedData) {
        const periodsSinceStart = Math.floor((row.time - firstDataTime) / timeframeSec);
        const barTime = firstDataTime + (periodsSinceStart * timeframeSec);
        
        if (!currentBar || currentBar.time !== barTime) {
          if (currentBar) {
            resampledData.push(currentBar);
          }
          currentBar = {
            time: barTime,
            open: row.open,
            high: row.high,
            low: row.low,
            close: row.close
          };
        } else {
          currentBar.high = Math.max(currentBar.high, row.high);
          currentBar.low = Math.min(currentBar.low, row.low);
          currentBar.close = row.close;
        }
      }
      if (currentBar) {
        resampledData.push(currentBar);
      }
      return resampledData;
    }

    // ----------------------- INDICATOR FUNCTIONS -----------------------
    // Simple Moving Average (SMA)
    function calculateSMA(data, period = 50) {
      const result = [];
      for (let i = period - 1; i < data.length; i++) {
        let sum = 0;
        for (let j = i - period + 1; j <= i; j++) {
          sum += data[j].close;
        }
        const sma = sum / period;
        result.push({ time: data[i].time, ['SMA ' + period]: sma });
      }
      return result;
    }

    // Relative Strength Index (RSI)
    function calculateRSI(data, windowLength = 14) {
      const delta = new Array(data.length).fill(0);
      for (let i = 1; i < data.length; i++) {
        delta[i] = data[i].close - data[i - 1].close;
      }
      const gains = new Array(data.length).fill(0);
      const losses = new Array(data.length).fill(0);
      for (let i = 1; i < delta.length; i++) {
        gains[i] = delta[i] > 0 ? delta[i] : 0;
        losses[i] = delta[i] < 0 ? -delta[i] : 0;
      }
      const avgGains = new Array(data.length).fill(null);
      const avgLosses = new Array(data.length).fill(null);
      const result = data.map(item => Object.assign({}, item));
      for (let i = windowLength; i < data.length; i++) {
        let sumGains = 0, sumLosses = 0;
        for (let j = i - windowLength + 1; j <= i; j++) {
          sumGains += gains[j];
          sumLosses += losses[j];
        }
        avgGains[i] = sumGains / windowLength;
        avgLosses[i] = sumLosses / windowLength;
      }
      for (let i = windowLength; i < data.length; i++) {
        const rs = avgLosses[i] === 0 ? Infinity : avgGains[i] / avgLosses[i];
        const rsi = 100 - (100 / (1 + rs));
        result[i].RSI = rsi;
      }
      return result;
    }

    // Exponential Moving Average (EMA)
    function calculateEMA(prices, span) {
      const ema = [];
      const multiplier = 2 / (span + 1);
      ema[0] = prices[0];
      for (let i = 1; i < prices.length; i++) {
        ema[i] = (prices[i] - ema[i - 1]) * multiplier + ema[i - 1];
      }
      return ema;
    }

    // Moving Average Convergence Divergence (MACD)
    function calculateMACD(data, short_window = 12, long_window = 26, signal_window = 9) {
      const prices = data.map(item => item.close);
      const shortEMA = calculateEMA(prices, short_window);
      const longEMA = calculateEMA(prices, long_window);
      const macdLine = prices.map((_, i) => shortEMA[i] - longEMA[i]);
      const signalLine = calculateEMA(macdLine, signal_window);
      data.forEach((item, i) => {
        item.MACD = macdLine[i];
        item.Signal_Line = signalLine[i];
      });
      return data;
    }

    // Update indicators based on current candlestick data.
    function updateIndicators() {
      const candlestickData = resampleData(originalData, currentTimeframe);
      
      // Calculate SMAs
      const sma12 = calculateSMA(candlestickData, 12);
      const sma24 = calculateSMA(candlestickData, 24);
      const sma50 = calculateSMA(candlestickData, 50);
      const sma200 = calculateSMA(candlestickData, 200);
      sma12Series.setData(sma12.map(item => ({ time: item.time, value: item['SMA 12'] })));
      sma24Series.setData(sma24.map(item => ({ time: item.time, value: item['SMA 24'] })));
      sma50Series.setData(sma50.map(item => ({ time: item.time, value: item['SMA 50'] })));
      sma200Series.setData(sma200.map(item => ({ time: item.time, value: item['SMA 200'] })));
      
      // Calculate RSI (14 period)
      const rsiFullData = calculateRSI(candlestickData, 14);
      const rsiData = rsiFullData.filter(item => item.RSI !== undefined)
                                 .map(item => ({ time: item.time, value: item.RSI }));
      rsiSeries.setData(rsiData);
      
      // Calculate MACD
      const macdFullData = calculateMACD(candlestickData);
      const macdData = macdFullData.map(item => ({ time: item.time, value: item.MACD }));
      const signalData = macdFullData.map(item => ({ time: item.time, value: item.Signal_Line }));
      const histogramData = macdFullData.map(item => ({ time: item.time, value: item.MACD - item.Signal_Line }));
      
      macdSeries.setData(macdData);
      signalSeries.setData(signalData);
      histogramSeries.setData(histogramData);
    }

    // ----------------------- UPDATE CHART -----------------------
    function updateChartTimeframe(timeframe) {
      currentTimeframe = timeframe;
      const resampledData = resampleData(originalData, timeframe);
      // Update candlestick series with resampled data.
      candlestickSeries.setData(resampledData);
      // Update all indicators.
      updateIndicators();
    }

    // Set up timeframe button click handlers.
    document.querySelectorAll('.tf-button').forEach(button => {
      button.addEventListener('click', function() {
        // Remove active class from all buttons.
        document.querySelectorAll('.tf-button').forEach(btn => {
          btn.classList.remove('active');
        });
        // Add active class to clicked button.
        this.classList.add('active');
        // Update chart with selected timeframe.
        updateChartTimeframe(this.getAttribute('data-timeframe'));
      });
    });

    // Define the initial candle time.
    const initialCandleTimeStr = "2024-02-15T09:16:00Z";
    const initialTimeMs = new Date(initialCandleTimeStr).getTime() / 1000; // seconds

    // Split data into historical and streaming parts.
    let streamingData = historicalData.filter(bar => bar.time >= initialTimeMs);
    originalData = historicalData.filter(bar => bar.time < initialTimeMs);

    // Set the initial data with default timeframe.
    updateChartTimeframe(currentTimeframe);

    // ----------------------- TRADING SIMULATOR -----------------------
    let totalTrades = 0;
    let profitableTrades = 0;
    let totalProfit = 0;
    let longTrades = 0;
    let shortTrades = 0;
    let isPaused = false;
    let isLong = false;
    let isShort = false;
    let entryPrice = 0;
    let latestPrice = null; 
    let activeTrade = null;

    const floatingRectangleId = 'floatingRectangle';

    function formatTime(timestamp) {
      return new Date(timestamp).toLocaleString('en-US', {
        timeZone: 'UTC',
        hour: '2-digit', minute: '2-digit', second: '2-digit', hour12: false
      });
    }

    function calculateProfit(entryPrice, exitPrice, isLong) {
      return (((isLong ? exitPrice - entryPrice : entryPrice - exitPrice) / entryPrice) * 100).toFixed(2);
    }

    function logChartUpdate(message) {
      document.getElementById('chartUpdateLog').innerHTML = `Chart Updates<br>${message}`;
    }

    function logTrade(message) {
      const tradeLog = document.getElementById('tradeLog');
      tradeLog.innerHTML += `<div class="log-entry">${message}</div>`;
      tradeLog.scrollTop = tradeLog.scrollHeight;
    }

    function updateProfitLog() {
      const profitPercentage = totalProfit.toFixed(2);
      const profitClass = profitPercentage >= 0 ? 'profit-positive' : 'profit-negative';

      document.getElementById('profitLog').innerHTML = `
        Overall Profit<br>
        <span class="${profitClass}">Total Profit: ${profitPercentage}%</span><br>
        Total Trades: ${totalTrades}<br>
        Profitable Trades: ${profitableTrades}<br>
        Long Trades: ${longTrades}<br>
        Short Trades: ${shortTrades}
      `;
    }

    // ----------------------- FLOATING RECTANGLE -----------------------
    function initializeFloatingRectangle() {
      const floatingRectangle = document.getElementById(floatingRectangleId);
      let isDragging = false;
      let offsetX = 0;
      let offsetY = 0;

      const startDrag = (e) => {
        isDragging = true;
        const rect = floatingRectangle.getBoundingClientRect();
        const clientX = e.touches ? e.touches[0].clientX : e.clientX;
        const clientY = e.touches ? e.touches[0].clientY : e.clientY;
        offsetX = clientX - rect.left;
        offsetY = clientY - rect.top;
        floatingRectangle.style.cursor = 'grabbing';
      };

      const onDrag = (e) => {
        if (!isDragging) return;
        const clientX = e.touches ? e.touches[0].clientX : e.clientX;
        const clientY = e.touches ? e.touches[0].clientY : e.clientY;
        floatingRectangle.style.left = `${clientX - offsetX}px`;
        floatingRectangle.style.top = `${clientY - offsetY}px`;
      };

      const endDrag = () => {
        isDragging = false;
        floatingRectangle.style.cursor = 'grab';
      };

      floatingRectangle.addEventListener('mousedown', startDrag);
      floatingRectangle.addEventListener('touchstart', startDrag, { passive: false });
      document.addEventListener('mousemove', onDrag);
      document.addEventListener('touchmove', onDrag, { passive: false });
      document.addEventListener('mouseup', endDrag);
      document.addEventListener('touchend', endDrag);
    }

    // ----------------------- TRADE FUNCTIONS -----------------------
    function executeLongTrade() {
      if (!latestPrice) {
        alert("No price data available. Please wait for the data to update.");
        return;
      }

      if (isLong) {
        const profit = parseFloat(calculateProfit(entryPrice, latestPrice, true));
        logTrade(`Long Close: ${profit}%, Price: $${latestPrice.toFixed(2)}, Time: ${formatTime(window.currentCandleTime)}`);
        totalTrades++;
        totalProfit += profit;
        if (profit > 0) profitableTrades++;
        longTrades++;
        activeTrade = null; 
      } else {
        logTrade(`Long Entry: Price: $${latestPrice.toFixed(2)}, Time: ${formatTime(window.currentCandleTime)}`);
        activeTrade = { type: 'Long', entryPrice: latestPrice }; 
      }
      isLong = !isLong;
      entryPrice = latestPrice;
      updateProfitLog();
    }

    function executeShortTrade() {
      if (!latestPrice) {
        alert("No price data available. Please wait for the data to update.");
        return;
      }

      if (isShort) {
        const profit = parseFloat(calculateProfit(entryPrice, latestPrice, false));
        logTrade(`Short Close: ${profit}%, Price: $${latestPrice.toFixed(2)}, Time: ${formatTime(window.currentCandleTime)}`);
        totalTrades++;
        totalProfit += profit;
        if (profit > 0) profitableTrades++;
        shortTrades++;
        activeTrade = null; 
      } else {
        logTrade(`Short Entry: Price: $${latestPrice.toFixed(2)}, Time: ${formatTime(window.currentCandleTime)}`);
        activeTrade = { type: 'Short', entryPrice: latestPrice }; 
      }
      isShort = !isShort;
      entryPrice = latestPrice;
      updateProfitLog();
    }

    function togglePause() {
      isPaused = !isPaused;
      logTrade(isPaused ? 'Paused' : 'Resumed');
    }

    // Update Real-Time Profit/Loss
    function updateRealTimeProfitLoss() {
      if (activeTrade && latestPrice) {
        const profit = parseFloat(calculateProfit(activeTrade.entryPrice, latestPrice, activeTrade.type === 'Long'));
        const profitClass = profit >= 0 ? 'profit-positive' : 'profit-negative';
        const tradeLog = document.getElementById('tradeLog');

        const lastLogEntry = tradeLog.querySelector('.log-entry:last-child');
        if (lastLogEntry) {
          lastLogEntry.innerHTML = `${lastLogEntry.innerText.split('<br>')[0]}<br><span class="${profitClass}">Real-Time P/L: ${profit}%</span>`;
        }
      }
    }

    // Simulate streaming data and connect to trading simulator.
    function startStreaming() {
      let index = 0;
      
      const streamInterval = setInterval(() => {
        if (isPaused) return;
        
        if (index < streamingData.length) {
          const newData = streamingData[index++];
          console.log("Streaming new data:", newData);
          
          // Add the new data to our complete dataset.
          originalData.push(newData);
          
          // Update latestPrice for trading simulator.
          latestPrice = newData.close;
          
          // Store the current candle timestamp for trades.
          window.currentCandleTime = newData.time * 1000; // seconds -> ms
          
          // Log chart update.
          logChartUpdate(`Price: $${latestPrice.toFixed(2)}, Time: ${formatTime(new Date(newData.time * 1000).getTime())}`);
          
          // Update real-time profit/loss if there's an active trade.
          updateRealTimeProfitLoss();
          
          // Update chart with resampled data including the new candle.
          updateChartTimeframe(currentTimeframe);
        } else {
          console.log("Streaming completed");
          clearInterval(streamInterval);
        }
      }, 1000); // Stream a new candle every second
    }

    // Initialize trading simulator.
    function initializeTradingSimulator() {
      initializeFloatingRectangle();
      
      document.getElementById('l').onclick = executeLongTrade;
      document.getElementById('s').onclick = executeShortTrade;
      document.getElementById('p').onclick = togglePause;
      
      // Add keyboard shortcuts.
      document.addEventListener('keydown', function(event) {
        if (event.ctrlKey && event.key === 'b') {
          event.preventDefault();
          executeLongTrade();
        } else if (event.ctrlKey && event.key === 'v') {
          event.preventDefault();
          executeShortTrade();
        }
      });
    }

    // Initialize everything.
    initializeTradingSimulator();
    startStreaming();
  </script>
</body>
</html>

```