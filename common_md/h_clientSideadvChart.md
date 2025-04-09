
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
    // Original data array
    let historicalData = [
      { timestamp: '2024-02-15T10:00:00Z', open: 100, high: 102, low: 98, close: 101 },
      { timestamp: '2024-02-15T10:00:05Z', open: 101, high: 103, low: 99, close: 102 },
      { timestamp: '2024-02-15T10:00:10Z', open: 102, high: 104, low: 100, close: 103 },
      { timestamp: '2024-02-15T10:00:15Z', open: 103, high: 105, low: 101, close: 104 },
      { timestamp: '2024-02-15T10:00:20Z', open: 104, high: 106, low: 102, close: 105 },
      { timestamp: '2024-02-15T10:00:25Z', open: 105, high: 107, low: 103, close: 106 },
      { timestamp: '2024-02-15T10:00:30Z', open: 106, high: 108, low: 104, close: 107 },
      { timestamp: '2024-02-15T10:00:35Z', open: 107, high: 109, low: 105, close: 108 },
      { timestamp: '2024-02-15T10:00:40Z', open: 108, high: 110, low: 106, close: 109 },
      { timestamp: '2024-02-15T10:00:45Z', open: 109, high: 111, low: 107, close: 110 },
      { timestamp: '2024-02-15T10:00:50Z', open: 110, high: 112, low: 108, close: 111 },
      { timestamp: '2024-02-15T10:00:55Z', open: 111, high: 113, low: 109, close: 112 },
      { timestamp: '2024-02-15T10:01:00Z', open: 112, high: 114, low: 110, close: 113 }
    ];

    // Convert timestamps to time in milliseconds
    historicalData = historicalData.map(data => ({
      time: new Date(data.timestamp).getTime(),
      open: data.open,
      high: data.high,
      low: data.low,
      close: data.close
    }));

    // Define the initial candle time
    const initialCandleTimeStr = "2024-02-15T10:00:10Z";
    const initialTimeMs = new Date(initialCandleTimeStr).getTime();
    
    // Split data into historical and streaming parts
    let streamingData = historicalData.filter(bar => bar.time >= initialTimeMs);
    historicalData = historicalData.filter(bar => bar.time < initialTimeMs);
    
    console.log("Historical Data:", historicalData);
    console.log("Streaming Data:", streamingData);
    
    // Create data arrays for each timeframe
    const dataArrays = {
      '5S': [],
      '10S': [],
      '15S': [],
      '30S': []
    };
    
    // Store the active callback and resolution
    let activeCallback = null;
    let activeResolution = '5S';
    
    // Flag to track if streaming has started
    let streamingStarted = false;
    
    // Convert a timeframe value (e.g., "5S") into milliseconds
    function getTimeframeMs(timeframe) {
      const unit = timeframe.slice(-1);
      const value = parseInt(timeframe);
      if (unit === 'S') {
        return value * 1000;
      }
      return value * 60 * 1000; // Default to minutes
    }
    
    // Resample data into bars for the given timeframe
    function resampleData(data, timeframe) {
      const resampledData = [];
      let currentBar = null;
      const timeframeMs = getTimeframeMs(timeframe);
      
      // Sort data by time to ensure proper resampling
      const sortedData = [...data].sort((a, b) => a.time - b.time);
      
      for (const row of sortedData) {
        const barTime = Math.floor(row.time / timeframeMs) * timeframeMs;
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
    
    // Initialize data arrays for each timeframe
    for (let timeframe in dataArrays) {
      dataArrays[timeframe] = resampleData(historicalData, timeframe);
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
          minmov: 1,
          pricescale: 100,
          volume_precision: 2,
          data_status: "streaming"
        });
      },
      
      getBars: (symbolInfo, resolution, periodParams, onHistoryCallback, onErrorCallback) => {
        const { from, to, firstDataRequest } = periodParams;
        console.log('[getBars]: Method call', symbolInfo, resolution, new Date(from * 1000), new Date(to * 1000));
        
        // Update active resolution when getBars is called with firstDataRequest
        if (firstDataRequest) {
          activeResolution = resolution;
          console.log(`[getBars]: Resolution changed to ${resolution}`);
          
          // Return the appropriate data array for this resolution
          console.log(`[getBars]: Returning ${resolution} data (${dataArrays[resolution].length} bars)`);
          onHistoryCallback(dataArrays[resolution], { noData: dataArrays[resolution].length === 0 });
        } else {
          // For subsequent requests, indicate no additional data
          console.log('[getBars]: No additional historical data available');
          onHistoryCallback([], { noData: true });
        }
      },
      
      subscribeBars: (symbolInfo, resolution, onRealtimeCallback, subscriberUID) => {
        console.log('[subscribeBars]: Method call with subscriberUID:', subscriberUID);
        
        // Store the callback and resolution
        activeCallback = onRealtimeCallback;
        activeResolution = resolution;
        
        // Start streaming data if not already started
        if (!streamingStarted && streamingData.length > 0) {
          streamingStarted = true;
          startStreaming();
        }
      },
      
      unsubscribeBars: (subscriberUID) => {
        console.log('[unsubscribeBars]: Method call with subscriberUID:', subscriberUID);
        // Clear the active callback
        activeCallback = null;
      }
    };
    
    // Simulate streaming data
    function startStreaming() {
      let index = 0;
      let allData = [...historicalData]; // Keep track of all data for resampling
      
      const streamInterval = setInterval(() => {
        if (index < streamingData.length) {
          const newData = streamingData[index++];
          console.log("Streaming new data:", newData);
          
          // Add the new data to our complete dataset
          allData.push(newData);
          
          // Update all timeframe arrays
          for (let timeframe in dataArrays) {
            dataArrays[timeframe] = resampleData(allData, timeframe);
          }
          
          // Call the callback with the appropriate data for the active resolution
          if (activeCallback) {
            if (activeResolution === '5S') {
              // For 5S, send the raw data
              activeCallback(newData);
            } else {
              // For other resolutions, send the latest bar from the resampled data
              const resampledArray = dataArrays[activeResolution];
              if (resampledArray.length > 0) {
                const latestBar = resampledArray[resampledArray.length - 1];
                activeCallback(latestBar);
              }
            }
          }
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
        interval: "5S", // Default interval
        fullscreen: false,
        timezone: "Etc/UTC",
        width: window.innerWidth,
        height: window.innerHeight,
        container: "chart_container",
        datafeed: Datafeed,
        theme: "Light",
        time_scale: { visible: true },
        enabled_features: ["dont_show_boolean_study_arguments", "seconds_resolution"],
        library_path: 'https://cdn.jsdelivr.net/gh/parth-royale/cdn@main/charting_library/charting_library.standalone.js'
      });
    }

    document.addEventListener("DOMContentLoaded", createChart);
  </script>
</body>
</html>
````
