
```html
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
    tickMarkFormatter: (time) => {
        const utcDate = new Date(time * 1000); // Convert UNIX time to Date object (UTC)
        const istDate = new Date(utcDate.getTime() + (5.5 * 60 * 60 * 1000)); // Convert to IST
        return istDate.toLocaleTimeString('en-IN');
    }
},
localization: {
    timeFormatter: (time) => {
        const utcDate = new Date(time * 1000);
        const istDate = new Date(utcDate.getTime() + (5.5 * 60 * 60 * 1000)); 
        return istDate.toLocaleDateString('en-IN') + ' ' + istDate.toLocaleTimeString('en-IN');
    }
}
});

const candleSeries = chart.addCandlestickSeries();

// Store original data
let originalData = [];
let currentTimeframe = '5s';

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
  const resampledData = [];
  let currentBar = null;
    const timeframeMs = getTimeframeMs(timeframe);
    const timeframeSec = timeframeMs / 1000; // Convert to seconds since data is in Unix seconds

  for (const row of data) {
    const barTime = Math.floor(row.time / timeframeSec) * timeframeSec;
    if (!currentBar || currentBar.time !== barTime) {
      if (currentBar) {
        resampledData.push(currentBar);
      }
      currentBar = {
        time: barTime,
        open: row.open,
        high: row.high,
        low: row.low,
        close: row.close,
        volume: row.volume
      };
    } else {
      currentBar.high = Math.max(currentBar.high, row.high);
      currentBar.low = Math.min(currentBar.low, row.low);
      currentBar.close = row.close;
      currentBar.volume += row.volume;
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

// First, fetch historical data
fetch('/historic')
 .then(response => response.json())
 .then(candles => {
   // Store the original data
   originalData = candles;
   
   // Set the initial data with default timeframe
   updateChartTimeframe(currentTimeframe);
   
   // Then connect to WebSocket for real-time updates
   const ws = new WebSocket((location.protocol === "https:" ? "wss://" : "ws://") + location.host + "/ws");
   
   ws.onmessage = function(event) {
       const data = JSON.parse(event.data);
       if (data.candle) {
           // Add new candle to original data
           originalData.push(data.candle);
           
           // Update chart with resampled data including the new candle
           updateChartTimeframe(currentTimeframe);
       }
   };
 })
 .catch(error => {
   console.error('Error fetching historical data:', error);
 });
</script>
</body>
</html>
```