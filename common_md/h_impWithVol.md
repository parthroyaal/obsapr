
````html
<!DOCTYPE html>
<html lang="en">
<head>
  <meta charset="UTF-8">
  <meta name="viewport" content="width=device-width, initial-scale=1.0">
  <title>Trading Chart</title>
  <link rel="icon" href="favicon.ico" type="image/x-icon">
</head>
<body>
  <div id="chart_container"></div>

  <script type="text/javascript" src="https://cdn.jsdelivr.net/gh/parth-royale/cdn@main/charting_library/charting_library.standalone.js"></script>

  <script>
    let historicalData = [
      { timestamp: '2024-02-15T10:00:00Z', open: 100, high: 102, low: 98, close: 101, volume: 1500 },
      { timestamp: '2024-02-15T10:00:05Z', open: 101, high: 103, low: 99, close: 102, volume: 2300 },
      { timestamp: '2024-02-15T10:00:10Z', open: 102, high: 104, low: 100, close: 103, volume: 1800 },
      { timestamp: '2024-02-15T10:00:15Z', open: 103, high: 105, low: 101, close: 104, volume: 2100 },
      { timestamp: '2024-02-15T10:00:20Z', open: 104, high: 106, low: 102, close: 105, volume: 1700 }
    ];

    // Convert timestamps to time in milliseconds
    historicalData = historicalData.map(data => ({
      time: new Date(data.timestamp).getTime(),
      open: data.open,
      high: data.high,
      low: data.low,
      close: data.close,
      volume: data.volume
    }));

    console.log("Converted Data:", historicalData);

    // Define the initial candle time
    const initialCandleTimeStr = "2024-02-15T10:00:10Z";
    const initialTimeMs = new Date(initialCandleTimeStr).getTime();
    
    // Split data into historical and streaming parts
    let streamingData = historicalData.filter(bar => bar.time >= initialTimeMs);
    historicalData = historicalData.filter(bar => bar.time < initialTimeMs);
    
    console.log("Historical Data:", historicalData);
    console.log("Streaming Data:", streamingData);
    
    // Store subscribers for real-time updates
    const subscribers = new Map();
    
    // Function to notify subscribers of new data
    function notifySubscribers(newData) {
      for (let [subscriberUID, handler] of subscribers) {
        handler.callback(newData);
      }
    }

    const Datafeed = {
      onReady: (callback) => callback({ supported_resolutions: ["5S", "10S", "15S", "30S"] }),
      
      resolveSymbol: (symbolName, onSymbolResolvedCallback) => {
        onSymbolResolvedCallback({
          name: symbolName,
          type: "crypto",
          session: "24x7",
          timezone: "Asia/Kolkata",
          supported_resolutions: ["5S", "10S", "15S", "30S"],
          has_intraday: true,
          has_seconds: true,
          has_no_volume: false,
          minmov: 1,
          pricescale: 100,
          volume_precision: 2,
          data_status: "streaming"
        });
      },
      
      getBars: (symbolInfo, resolution, periodParams, onHistoryCallback, onErrorCallback) => {
        const { from, to, firstDataRequest } = periodParams;
        console.log('[getBars]: Method call', symbolInfo, resolution, new Date(from * 1000), new Date(to * 1000));
        
        if (firstDataRequest) {
          // For first request, return all historical data
          console.log(`[getBars]: Returning all historical data (${historicalData.length} bars)`);
          onHistoryCallback(historicalData, { noData: historicalData.length === 0 });
        } else {
          // For subsequent requests, indicate no additional data
          // The chart already has all our historical data from the first request
          console.log('[getBars]: No additional historical data available');
          onHistoryCallback([], { noData: true });
        }
      },
      
      subscribeBars: (symbolInfo, resolution, onRealtimeCallback, subscriberUID) => {
        console.log('[subscribeBars]: Method call with subscriberUID:', subscriberUID);
        subscribers.set(subscriberUID, { 
          symbolInfo, 
          resolution, 
          callback: onRealtimeCallback 
        });
        
        // Start streaming data if not already started
        if (!window.streamingStarted && streamingData.length > 0) {
          window.streamingStarted = true;
          startStreaming();
        }
      },
      
      unsubscribeBars: (subscriberUID) => {
        console.log('[unsubscribeBars]: Method call with subscriberUID:', subscriberUID);
        subscribers.delete(subscriberUID);
      }
    };
    
    // Simulate streaming data
    function startStreaming() {
      let index = 0;
      const streamInterval = setInterval(() => {
        if (index < streamingData.length) {
          const newData = streamingData[index++];
          console.log("Streaming new data:", newData);
          notifySubscribers(newData);
        } else {
          console.log("Streaming completed");
          clearInterval(streamInterval);
        }
      }, 1000); // Stream a new candle every second
    }

    // Create the TradingView chart
    function createChart() {
      const widget = new TradingView.widget({
        symbol: "NIFTYBANK",
        interval: "5S",
        fullscreen: false,
        timezone: "Etc/UTC",
        width: window.innerWidth,
        height: window.innerHeight,
        container: "chart_container",
        datafeed: Datafeed,
        theme: "Light",
        time_scale: { visible: true },
        enabled_features: ["dont_show_boolean_study_arguments", "seconds_resolution"],
        library_path: 'https://cdn.jsdelivr.net/gh/parth-royale/cdn@main/charting_library/charting_library.standalone.js',
        studies_overrides: {
          "volume.volume.color.0": "#f23645",
          "volume.volume.color.1": "#26a69a"
        }
      });
    }

    document.addEventListener("DOMContentLoaded", createChart);
  </script>
</body>
</html>
````

````html
<!DOCTYPE html>
<html lang="en">
<head>
  <meta charset="UTF-8">
  <meta name="viewport" content="width=device-width, initial-scale=1.0">
  <title>Trading Chart with Volume</title>
  <script src="https://unpkg.com/lightweight-charts@3.8.0/dist/lightweight-charts.standalone.production.js"></script>
  <style>
    body { margin: 0; padding: 0; }
    #chart_container { width: 100%; height: 100vh; display: flex; flex-direction: column; }
    #chart, #volume_chart { flex: 1; }
  </style>
</head>
<body>
  <div id="chart_container">
    <div id="chart"></div>
    <div id="volume_chart"></div>
  </div>

  <script>
    // Create main chart
    const chart = LightweightCharts.createChart(document.getElementById('chart'), {
      width: window.innerWidth,
      height: window.innerHeight * 0.7,
      layout: { backgroundColor: '#ffffff', textColor: '#000' },
      grid: { vertLines: { color: '#eee' }, horzLines: { color: '#eee' } }
    });

    // Create volume chart
    const volChart = LightweightCharts.createChart(document.getElementById('volume_chart'), {
      width: window.innerWidth,
      height: window.innerHeight * 0.3,
      layout: { backgroundColor: '#ffffff', textColor: '#000' },
      grid: { vertLines: { color: '#eee' }, horzLines: { color: '#eee' } }
    });

    // Add series
    const series = chart.addCandlestickSeries();
    const volSeries = volChart.addHistogramSeries({ color: '#26a69a', priceFormat: { type: 'volume' } });

    // Mock historical data with volume
    let historicalData = [
      { time: 1707984000, open: 100, high: 102, low: 98, close: 101, volume: 5000 },
      { time: 1707984010, open: 101, high: 103, low: 99, close: 102, volume: 7000 },
      { time: 1707984020, open: 102, high: 104, low: 100, close: 103, volume: 6000 },
      { time: 1707984030, open: 103, high: 105, low: 101, close: 104, volume: 8000 },
      { time: 1707984040, open: 104, high: 106, low: 102, close: 105, volume: 7500 }
    ];

    // Extract volume data
    let volumeData = historicalData.map(bar => ({
      time: bar.time,
      value: bar.volume,
      color: bar.close >= bar.open ? '#26a69a' : '#ef5350' // Green for up, red for down
    }));

    // Set initial data
    series.setData(historicalData);
    volSeries.setData(volumeData);

    // Mock streaming data with volume
    let streamingData = [
      { time: 1707984050, open: 105, high: 107, low: 103, close: 106, volume: 9000 },
      { time: 1707984060, open: 106, high: 108, low: 104, close: 107, volume: 8500 }
    ];

    let index = 0;
    function startStreaming() {
      setInterval(() => {
        if (index < streamingData.length) {
          let newCandle = streamingData[index];

          // Update chart series
          series.update(newCandle);
          volSeries.update({
            time: newCandle.time,
            value: newCandle.volume,
            color: newCandle.close >= newCandle.open ? '#26a69a' : '#ef5350'
          });

          console.log("New candle:", newCandle);
          index++;
        }
      }, 1000);
    }

    startStreaming();
  </script>
</body>
</html>

````
