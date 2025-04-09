# Tags

[h_tags](h_tags.md)

# Misc

[h_impWithVol](h_impWithVol.md)

# tradingPnlLog&logic

[h_order1](h_order1.md)

# Triggers

[h_order2](h_order2.md)

# Inidi

[h_order3](h_order3.md)

# Base

````html
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
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
    
    <div id="chart"></div>
<script>
const chart = LightweightCharts.createChart(document.getElementById('chart'), {
width: window.innerWidth,
height: window.innerHeight,
priceScale: { borderColor: '#cccccc' },
timeScale: { 
    borderColor: '#cccccc', 
    timeVisible: true, 
    secondsVisible: true,
}
 })


const candleSeries = chart.addCandlestickSeries();

// Store original data
let originalData = [];
let currentTimeframe = '5s';
let dataArrays = {};
let activeCallback = null;
let activeResolution = '5s';

// Original data array
let historicalData = [
  { timestamp: '2024-02-15T09:15:00Z', open: 100, high: 102, low: 98, close: 101 },
  { timestamp: '2024-02-15T09:15:05Z', open: 101, high: 103, low: 99, close: 102 },
  { timestamp: '2024-02-15T09:15:10Z', open: 102, high: 104, low: 100, close: 103 },
  { timestamp: '2024-02-15T09:15:15Z', open: 103, high: 105, low: 101, close: 104 },
  { timestamp: '2024-02-15T09:15:20Z', open: 104, high: 106, low: 102, close: 105 },
  { timestamp: '2024-02-15T09:15:25Z', open: 105, high: 107, low: 103, close: 106 },
  { timestamp: '2024-02-15T09:15:30Z', open: 106, high: 108, low: 104, close: 107 },
  { timestamp: '2024-02-15T09:15:35Z', open: 107, high: 109, low: 105, close: 108 },
  { timestamp: '2024-02-15T09:15:40Z', open: 108, high: 110, low: 106, close: 109 },
  { timestamp: '2024-02-15T09:15:45Z', open: 109, high: 111, low: 107, close: 110 },
  { timestamp: '2024-02-15T09:15:50Z', open: 110, high: 112, low: 108, close: 111 },
  { timestamp: '2024-02-15T09:15:55Z', open: 111, high: 113, low: 109, close: 112 },
  { timestamp: '2024-02-15T09:16:00Z', open: 112, high: 114, low: 110, close: 113 },
  { timestamp: '2024-02-15T09:16:05Z', open: 113, high: 115, low: 111, close: 114 },
  { timestamp: '2024-02-15T09:16:10Z', open: 114, high: 116, low: 112, close: 115 },
  { timestamp: '2024-02-15T09:16:15Z', open: 115, high: 117, low: 113, close: 116 },
  { timestamp: '2024-02-15T09:16:20Z', open: 116, high: 118, low: 114, close: 117 },
  { timestamp: '2024-02-15T09:16:25Z', open: 117, high: 119, low: 115, close: 118 },
  { timestamp: '2024-02-15T09:16:30Z', open: 118, high: 120, low: 116, close: 119 },
  { timestamp: '2024-02-15T09:16:35Z', open: 119, high: 121, low: 117, close: 120 },
  { timestamp: '2024-02-15T09:16:40Z', open: 120, high: 122, low: 118, close: 121 },
  { timestamp: '2024-02-15T09:16:45Z', open: 121, high: 123, low: 119, close: 122 },
  { timestamp: '2024-02-15T09:16:50Z', open: 122, high: 124, low: 120, close: 123 },
  { timestamp: '2024-02-15T09:16:55Z', open: 123, high: 125, low: 121, close: 124 },
  { timestamp: '2024-02-15T09:17:00Z', open: 124, high: 126, low: 122, close: 125 },
  { timestamp: '2024-02-15T09:17:05Z', open: 125, high: 127, low: 123, close: 126 },
  { timestamp: '2024-02-15T09:17:10Z', open: 126, high: 128, low: 124, close: 127 },
  { timestamp: '2024-02-15T09:17:15Z', open: 127, high: 129, low: 125, close: 128 },
  { timestamp: '2024-02-15T09:17:20Z', open: 128, high: 130, low: 126, close: 129 },
  { timestamp: '2024-02-15T09:17:25Z', open: 129, high: 131, low: 127, close: 130 },
  { timestamp: '2024-02-15T09:17:30Z', open: 130, high: 132, low: 128, close: 131 },
  { timestamp: '2024-02-15T09:17:35Z', open: 131, high: 133, low: 129, close: 132 },
  { timestamp: '2024-02-15T09:17:40Z', open: 132, high: 134, low: 130, close: 133 },
  { timestamp: '2024-02-15T09:17:45Z', open: 133, high: 135, low: 131, close: 134 },
  { timestamp: '2024-02-15T09:17:50Z', open: 134, high: 136, low: 132, close: 135 },
  { timestamp: '2024-02-15T09:17:55Z', open: 135, high: 137, low: 133, close: 136 },
  { timestamp: '2024-02-15T09:18:00Z', open: 136, high: 138, low: 134, close: 137 },
  { timestamp: '2024-02-15T09:18:05Z', open: 137, high: 139, low: 135, close: 138 },
  { timestamp: '2024-02-15T09:18:10Z', open: 138, high: 140, low: 136, close: 139 },
  { timestamp: '2024-02-15T09:18:15Z', open: 139, high: 141, low: 137, close: 140 },
  { timestamp: '2024-02-15T09:18:20Z', open: 140, high: 142, low: 138, close: 141 },
  { timestamp: '2024-02-15T09:18:25Z', open: 141, high: 143, low: 139, close: 142 },
  { timestamp: '2024-02-15T09:18:30Z', open: 142, high: 144, low: 140, close: 143 }
];

// Convert timestamps to time in milliseconds
historicalData = historicalData.map(data => ({
  time: new Date(data.timestamp).getTime() / 1000, // Convert to seconds for the chart
  open: data.open,
  high: data.high,
  low: data.low,
  close: data.close
}));

// Convert timeframe string to seconds (not milliseconds)
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

// Resample data into bars for the given timeframe
function resampleData(data, timeframe) {
    if (data.length === 0) return [];
    
    const resampledData = [];
    let currentBar = null;
    const timeframeMs = getTimeframeMs(timeframe);
    const timeframeSec = timeframeMs / 1000; // Convert to seconds since data is in Unix seconds
    
    // Sort data by time to ensure proper resampling
    const sortedData = [...data].sort((a, b) => a.time - b.time);
    
    // Get the first data point's time as reference
    const firstDataTime = sortedData[0].time;
    
    for (const row of sortedData) {
        // Calculate how many timeframe periods have passed since the first data point
        const periodsSinceStart = Math.floor((row.time - firstDataTime) / timeframeSec);
        
        // Calculate bar time based on periods since start
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

// Update chart with resampled data
function updateChartTimeframe(timeframe) {
    currentTimeframe = timeframe;
    const resampledData = resampleData(originalData, timeframe);
    candleSeries.setData(resampledData);
}

// Set up timeframe button click handlers
document.querySelectorAll('.tf-button').forEach(button => {
    button.addEventListener('click', function() {
        // Remove active class from all buttons
        document.querySelectorAll('.tf-button').forEach(btn => {
            btn.classList.remove('active');
        });
        
        // Add active class to clicked button
        this.classList.add('active');
        
        // Update chart with selected timeframe
        updateChartTimeframe(this.getAttribute('data-timeframe'));
    });
});

// Define the initial candle time
const initialCandleTimeStr = "2024-02-15T09:16:00Z";
const initialTimeMs = new Date(initialCandleTimeStr).getTime() / 1000; // Convert to seconds for the chart

// Split data into historical and streaming parts
let streamingData = historicalData.filter(bar => bar.time >= initialTimeMs);
originalData = historicalData.filter(bar => bar.time < initialTimeMs);

// Set the initial data with default timeframe
updateChartTimeframe(currentTimeframe);

// Simulate streaming data
function startStreaming() {
  let index = 0;
  
  const streamInterval = setInterval(() => {
    if (index < streamingData.length) {
      const newData = streamingData[index++];
      console.log("Streaming new data:", newData);
      
      // Add the new data to our complete dataset
      originalData.push(newData);
      
      // Update chart with resampled data including the new candle
      updateChartTimeframe(currentTimeframe);
    } else {
      console.log("Streaming completed");
      clearInterval(streamInterval);
    }
  }, 1000); // Stream a new candle every second
}

// Start streaming data
startStreaming();
</script>
</body>
</html>
````
