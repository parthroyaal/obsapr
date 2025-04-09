
````html
<!DOCTYPE html>
<html lang="en">
<head>
  <meta charset="UTF-8" />
  <meta name="viewport" content="width=device-width, initial-scale=1.0" />
  <title>Technical Indicators - Candlestick Chart</title>
  <style>

  /*Background color and text color */
body { font-family: Arial, sans-serif; background-color: #1a1a1a; color: white; }

  /* Container and Panels */
  .container { padding: 10px; max-width: 100%; margin: 0 auto; }
  h1 { font-size: 24px; padding: 10px; }
  .panel { background-color: #2a2a2a; padding: 15px; border-radius: 5px; width: 100%; min-width: 250px; }

  #chart { width: 100%; height: 400px; background-color: #000; margin-bottom: 15px; }



  /* The first rule sets the default styles for the chart container. On mobile devices (or any device with a viewport narrower than 768px), the chart will use a height of 400 pixels. */
/* Desktop adjustments using media query */
    /* Responsive Design */
    @media (min-width: 768px) {
      .controls { flex-direction: row; flex-wrap: wrap; }
      .panel { flex: 1; min-width: 300px; }
      h1 { font-size: 28px; }
      #chart { height: 500px; }
    }
  </style>
</head>
<body>
  <h1>Technical Indicators Demo</h1>
  <div class="container">
  <div id="chart"></div>
      <!-- Panels Section -->
    <div class="panels">
      <div class="panel" id="sma-panel">
        <h2>SMA Indicators</h2>
        <p>
          Displays Simple Moving Averages for various periods (e.g., SMA12, SMA24, SMA50, SMA200).
        </p>
      </div>
      <div class="panel" id="rsi-panel">
        <h2>RSI Indicator</h2>
        <p>
          Relative Strength Index (RSI) helps determine overbought or oversold conditions.
        </p>
      </div>
      <div class="panel" id="macd-panel">
        <h2>MACD Indicator</h2>
        <p>
          The Moving Average Convergence Divergence (MACD) reveals changes in trend direction.
        </p>
      </div>
    </div>
  </div>
</div>
  <script src="https://cdn.jsdelivr.net/gh/parth-royale/cdn@main/lightweight-charts.standalone.production.js"></script>
  <script>
    // Load CSV file using fetch and process as text
    fetch('mar10.csv')
      .then(response => response.text())
      .then(csvText => {
        // Split CSV text into lines.
        // Assumes rows like: "2025-03-07 09:59:55,22535.25"
        const lines = csvText.trim().split('\n');
        // Create an array of tick data objects
        const tickData = lines.map(line => {
          const [time, close] = line.split(',');
          // Convert time to an ISO string in UTC (adds "T" and "Z")
          const isoTime = time.trim().replace(' ', 'T') + 'Z';
          return {
            time: Math.floor(new Date(isoTime).getTime() / 1000), // Unix timestamp in seconds
            close: parseFloat(close.trim())
          };
        });
        // Aggregate tick data into 5-second candlestick bars.
        const candlestickData = aggregateToCandlestick(tickData, 5);
        // Initialize the chart using the aggregated candlestick data.
        initializeCharts(candlestickData);
      })
      .catch(error => console.error("Error loading CSV file:", error));

    /**
     * Aggregates tick data into candlestick bars using a specified timeframe (in seconds).
     * For each bucket, the first tick is the open, the last tick is the close, and
     * high/low values are computed over all ticks within that bucket.
     *
     * @param {Array} data - Tick data objects [{ time: <Unix timestamp>, close: <price> }, ...]
     * @param {number} timeframe - Aggregation interval (default is 5 seconds)
     * @returns {Array} - Sorted array of candlestick objects
     */
    function aggregateToCandlestick(data, timeframe = 5) {
      const candles = {};
      data.forEach(item => {
        // Bucket ticks into a bucket based on the timeframe.
        const bucket = Math.floor(item.time / timeframe) * timeframe;
        if (!candles[bucket]) {
          candles[bucket] = {
            time: bucket,
            open: item.close,
            high: item.close,
            low: item.close,
            close: item.close
          };
        } else {
          candles[bucket].high = Math.max(candles[bucket].high, item.close);
          candles[bucket].low = Math.min(candles[bucket].low, item.close);
          candles[bucket].close = item.close; // Latest tick becomes the close price
        }
      });
      // Convert the object to an array and sort by time.
      return Object.values(candles).sort((a, b) => a.time - b.time);
    }

    /**
     * Creates the chart with candlestick series on the main pane,
     * overlays four SMAs on the main pane, and adds separate panes for RSI and MACD.
     *
     * @param {Array} candlestickData - Aggregated candlestick data
     */
    function initializeCharts(candlestickData) {
      const chart = LightweightCharts.createChart(document.getElementById('chart'), {
        layout: {
          backgroundColor: 'white',
          textColor: '#fff'
        },
        timeScale: {
          timeVisible: true,
          borderColor: '#D1D4DC'
        },
        rightPriceScale: {
          borderColor: '#D1D4DC'
        },
        grid: {
          horzLines: { color: '#F0F3FA' },
          vertLines: { color: '#F0F3FA' }
        }
      });

      // MAIN PANE: Add candlestick series.
      const candlestickSeries = chart.addCandlestickSeries({
        upColor: '#4caf50',
        downColor: '#f44336',
        borderUpColor: '#4caf50',
        borderDownColor: '#f44336',
        wickUpColor: '#4caf50',
        wickDownColor: '#f44336'
      });
      candlestickSeries.setData(candlestickData);

      // ---- Calculate indicators on the aggregated data ----
      // Using aggregated candlestickData ensures each data point has a unique timestamp.
      const sma12 = calculateSMA(candlestickData, 12);
      const sma24 = calculateSMA(candlestickData, 24);
      const sma50 = calculateSMA(candlestickData, 50);
      const sma200 = calculateSMA(candlestickData, 200);

      // Overlay SMA 12 (yellow, lineWidth: 2)
      const sma12Series = chart.addLineSeries({
        title: 'SMA 12',
        color: '#FFEB3B',
        lineWidth: 2
      });
      sma12Series.setData(sma12.map(item => ({ time: item.time, value: item['SMA ' + 12] })));

      // Overlay SMA 24 (red, lineWidth: 2)
      const sma24Series = chart.addLineSeries({
        title: 'SMA 24',
        color: '#FF0000',
        lineWidth: 2
      });
      sma24Series.setData(sma24.map(item => ({ time: item.time, value: item['SMA ' + 24] })));

      // Overlay SMA 50 (black, lineWidth: 3)
      const sma50Series = chart.addLineSeries({
        title: 'SMA 50',
        color: '#000000',
        lineWidth: 3
      });
      sma50Series.setData(sma50.map(item => ({ time: item.time, value: item['SMA ' + 50] })));

      // Overlay SMA 200 (blue, lineWidth: 3)
      const sma200Series = chart.addLineSeries({
        title: 'SMA 200',
        color: '#0000FF',
        lineWidth: 3
      });
      sma200Series.setData(sma200.map(item => ({ time: item.time, value: item['SMA ' + 200] })));

      // Calculate and add RSI (14 period) on pane 1.
      const rsiFullData = calculateRSI(candlestickData, 14);
      const rsiData = rsiFullData.filter(item => item.RSI !== undefined)
                                 .map(item => ({ time: item.time, value: item.RSI }));
      const rsiSeries = chart.addLineSeries({
        title: 'RSI 14',
        color: '#FF6D00',
        lineWidth: 2,
        pane: 1
      });
      rsiSeries.setData(rsiData);

      // Calculate and add MACD on pane 2.
      const macdFullData = calculateMACD(candlestickData);
      const macdData = macdFullData.map(item => ({ time: item.time, value: item.MACD }));
      const signalData = macdFullData.map(item => ({ time: item.time, value: item.Signal_Line }));
      const histogramData = macdFullData.map(item => ({ time: item.time, value: item.MACD - item.Signal_Line }));
      
      const macdSeries = chart.addLineSeries({
        title: 'MACD Line',
        color: '#FF6D00',
        lineWidth: 2,
        pane: 2
      });
      macdSeries.setData(macdData);
      
      const signalSeries = chart.addLineSeries({
        title: 'Signal Line',
        color: '#00C853',
        lineWidth: 2,
        pane: 2
      });
      signalSeries.setData(signalData);
      
      const histogramSeries = chart.addHistogramSeries({
        title: 'MACD Histogram',
        color: '#E53935',
        pane: 2
      });
      histogramSeries.setData(histogramData);
    }

    // ------------------ INDICATOR FUNCTIONS ------------------

    /**
     * Calculate the Simple Moving Average (SMA) over a specified period.
     * @param {Array} data - Aggregated candlestick data.
     * @param {number} period - The SMA period.
     * @returns {Array} - Array of objects with time and SMA value.
     */
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

    /**
     * Calculate the Relative Strength Index (RSI) for a given period.
     * @param {Array} data - Aggregated candlestick data.
     * @param {number} windowLength - The RSI period (default 14).
     * @returns {Array} - Data array with an added RSI property for each element.
     */
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

    /**
     * Calculate the Exponential Moving Average (EMA) for an array of prices.
     * @param {Array} prices - Array of prices.
     * @param {number} span - The EMA period.
     * @returns {Array} - EMA values.
     */
    function calculateEMA(prices, span) {
      const ema = [];
      const multiplier = 2 / (span + 1);
      ema[0] = prices[0];
      for (let i = 1; i < prices.length; i++) {
        ema[i] = (prices[i] - ema[i - 1]) * multiplier + ema[i - 1];
      }
      return ema;
    }

    /**
     * Calculate the Moving Average Convergence Divergence (MACD).
     * @param {Array} data - Aggregated candlestick data.
     * @param {number} short_window - Short EMA period (default 12).
     * @param {number} long_window - Long EMA period (default 26).
     * @param {number} signal_window - Signal EMA period (default 9).
     * @returns {Array} - Data array with added MACD and Signal_Line properties.
     */
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
  </script>
</body>
</html>
````
