

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

<!-- Generated by SpecStory -->

  

# NIFTY Chart Timeframe Selection Implementation (2025-04-05 09:08:24)

  

_**User**_

  

<!DOCTYPE html> <html lang="en"> <head>     <meta charset="UTF-8">     <meta name="viewport" content="width=device-width, initial-scale=1.0">     <title>Live NIFTY Tick Chart (IST)</title>     <script src="https://cdn.jsdelivr.net/gh/parth-royale/cdn@main/lightweight-charts.standalone.production.js"></script> </head> <body>     <h1>Live NIFTY Tick Chart (IST)</h1>     <div id="chart"></div> <script> const chart = LightweightCharts.createChart(document.getElementById('chart'), { width: window.innerWidth, height: window.innerHeight, priceScale: { borderColor: '#cccccc' }, timeScale: {      borderColor: '#cccccc',      timeVisible: true,      secondsVisible: true,     tickMarkFormatter: (time) => {         const utcDate = new Date(time * 1000); // Convert UNIX time to Date object (UTC)         const istDate = new Date(utcDate.getTime() + (5.5 * 60 * 60 * 1000)); // Convert to IST         return istDate.toLocaleTimeString('en-IN');     } }, localization: {     timeFormatter: (time) => {         const utcDate = new Date(time * 1000);         const istDate = new Date(utcDate.getTime() + (5.5 * 60 * 60 * 1000));          return istDate.toLocaleDateString('en-IN') + ' ' + istDate.toLocaleTimeString('en-IN');     } } });  const candleSeries = chart.addCandlestickSeries();  // First, fetch historical data fetch('/historic')  .then(response => response.json())  .then(candles => {    // Set the initial data    candleSeries.setData(candles);        // Then connect to WebSocket for real-time updates    const ws = new WebSocket((location.protocol === "https:" ? "wss://" : "ws://") + location.host + "/ws");        ws.onmessage = function(event) {        const data = JSON.parse(event.data);        if (data.candle) {            // We're receiving a candle directly from the server            candleSeries.update(data.candle);        }    };  })  .catch(error => {    console.error('Error fetching historical data:', error);  }); </script> </body> </html> fetch recive data as  (4508) [{…}, {…}, {…}, {…}, {…}, {…}, {…}, {…}, {…}, {…}, {…}, {…}, {…}, {…}, {…}, {…}, {…}, {…}, {…}, {…}, {…}, {…}, {…}, {…}, {…}, {…}, {…}, {…}, {…}, {…}, {…}, {…}, {…}, {…}, {…}, {…}, {…}, {…}, {…}, {…}, {…}, {…}, {…}, {…}, {…}, {…}, {…}, {…}, {…}, {…}, {…}, {…}, {…}, {…}, {…}, {…}, {…}, {…}, {…}, {…}, {…}, {…}, {…}, {…}, {…}, {…}, {…}, {…}, {…}, {…}, {…}, {…}, {…}, {…}, {…}, {…}, {…}, {…}, {…}, {…}, {…}, {…}, {…}, {…}, {…}, {…}, {…}, {…}, {…}, {…}, {…}, {…}, {…}, {…}, {…}, {…}, {…}, {…}, {…}, {…}, …]

[0 … 99]

0

:

{time: 1743545700, open: 23192.6, high: 23243.25, low: 23190.5, close: 23236.05}

1

:

{time: 1743545705, open: 23237.5, high: 23250.6, low: 23216.3, close: 23216.3} and so on and websocket update is {time: 1743632130, open: 23202.75, high: 23207, low: 23202.75, close: 23205.15} i wish to have button for chage timeframe of chart from base 5second current as how the data is recived to higher timeframe like  <!-- Buttons for second-based timeframes -->

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

        <button class="tf-button" data-timeframe="1month">1 Month</button> with         .active { background-color: #4CAF50; color: white; } use this for resmapling // Resample data into bars for the given timeframe

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

  

---

  

_**Assistant**_

  

I'll help you implement the timeframe selection functionality for your NIFTY chart. Here's how to update your HTML file:

  

```html:feb26/MAR26/sbk1.py

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

// ... existing code ...

});

  

const candleSeries = chart.addCandlestickSeries();

  

// Store original data

let originalData = [];

let currentTimeframe = '5s';

  

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

  

This implementation:

  

1. Adds the timeframe buttons with styling at the top of the chart

2. Stores the original 5-second data in an `originalData` array

3. Implements the `getTimeframeMs` function to convert timeframe strings to milliseconds

4. Uses your `resampleData` function to aggregate candles based on the selected timeframe

5. Adds click handlers for the timeframe buttons that update the chart

6. Modifies the WebSocket handler to add new candles to the original data and update the chart with resampled data

  

When a user clicks on a timeframe button, the chart will resample the data to that timeframe. The active button is highlighted with the green background color you specified. When new data comes in via WebSocket, it's added to the original data and the chart is updated with the current timeframe setting.

  

---

  

_**User**_

  

resampling is messed for historic data starting from 9:15 th 5 second base is 9:03.20 and for every it is messed

  

---

  

_**Assistant**_

  

I apologize for the issue with the resampling. The problem might be related to how the timestamps are being processed. Let me explain what might be happening:

  

The resampling function is using `Math.floor(row.time / timeframeMs) * timeframeMs` to determine the bar time, but there could be issues with the timestamp format or alignment.

  

Here are some potential fixes:

  

1. The timestamps in your data might be UNIX timestamps (seconds since epoch), but the resampling might need to account for market hours or specific time alignments.

  

2. For financial data, it's often important to align bars to specific time boundaries (like the start of market hours at 9:15).

  

3. The conversion between different time units might need adjustment, especially when dealing with market-specific timeframes.

  

To fix the resampling issue, you might need to:

  

1. Check the exact format of your timestamps and ensure they're being interpreted correctly

2. Add logic to align bars to proper market time boundaries

3. Consider adding a time offset to ensure bars start at the correct times (like 9:15 for market open)

4. Debug the actual timestamp values before and after resampling to see where the misalignment occurs

  

If you can provide a sample of the actual timestamp values from your data and what you expect them to be after resampling, I can help create a more precise fix for the resampling function.

  

---

  

_**User**_

  

format is unix seconds utc

  

---

  

_**Assistant**_

  
  
  

Thanks for confirming the format is Unix seconds in UTC. Let's fix the resampling function to properly handle the Unix timestamps.

  

```javascript:feb26/MAR26/sbk1.py

// ... existing code ...

  

// Resample data into bars for the given timeframe

function resampleData(data, timeframe) {

    const resampledData = [];

    let currentBar = null;

    const timeframeMs = getTimeframeMs(timeframe);

    const timeframeSec = timeframeMs / 1000; // Convert to seconds since data is in Unix seconds

    // Sort data by time to ensure proper resampling

    const sortedData = [...data].sort((a, b) => a.time - b.time);

    for (const row of sortedData) {

        // Calculate bar time in seconds (floor to nearest timeframe boundary)

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

  

// ... existing code ...

```

  

The key change is that I'm now converting the timeframe from milliseconds to seconds in the resampling function since your data timestamps are in Unix seconds. This should ensure that the bar time calculations align properly with your data format.

  

If you're still seeing issues with the resampling, it might be helpful to add some debugging code to check the actual values:

  

```javascript

// Add this inside the resampleData function to debug

console.log("Original time:", row.time, "Timeframe:", timeframe,

            "Timeframe seconds:", timeframeSec,

            "Bar time:", barTime,

            "Date:", new Date(row.time * 1000).toISOString());

```

  

This will help you see exactly how the timestamps are being processed and where any misalignment might be occurring.

  

---

  

_**User**_

  

ok now resonably corretly but for example for 1 hour why is candle starting from 9:00 not 9:15 since the sesssion start from 9:15

  

---

  

_**Assistant**_

  
  
  

You're right - for financial markets like NIFTY, candles should align with market hours. Since the market opens at 9:15, the hourly candles should start at 9:15, 10:15, etc., rather than at the standard hour boundaries.

  

Let's modify the resampling function to align candles with market hours:

  

```javascript:feb26/MAR26/sbk1.py

// ... existing code ...

  

// Resample data into bars for the given timeframe

function resampleData(data, timeframe) {

    const resampledData = [];

    let currentBar = null;

    const timeframeMs = getTimeframeMs(timeframe);

    const timeframeSec = timeframeMs / 1000; // Convert to seconds since data is in Unix seconds

    // Market open offset (9:15 AM IST in seconds from midnight)

    // 9 hours * 3600 + 15 minutes * 60 = 33300 seconds

    const marketOpenOffsetSec = 33300;

    // Sort data by time to ensure proper resampling

    const sortedData = [...data].sort((a, b) => a.time - b.time);

    for (const row of sortedData) {

        // Calculate bar time in seconds, aligned to market hours

        let barTime;

        if (timeframeSec >= 3600) { // For hourly or longer timeframes

            // Get seconds since midnight UTC

            const date = new Date(row.time * 1000);

            const secondsSinceMidnight = date.getUTCHours() * 3600 + date.getUTCMinutes() * 60 + date.getUTCSeconds();

            // Calculate seconds since market open (may be negative if before market open)

            const secondsSinceMarketOpen = ((secondsSinceMidnight - marketOpenOffsetSec) % 86400 + 86400) % 86400;

            // Floor to nearest timeframe boundary since market open

            const barOffsetSec = Math.floor(secondsSinceMarketOpen / timeframeSec) * timeframeSec;

            // Calculate the actual bar time

            const midnightTime = Math.floor(row.time / 86400) * 86400;

            barTime = midnightTime + marketOpenOffsetSec + barOffsetSec;

            // If we're before market open for the day, use previous day's last bar

            if (secondsSinceMidnight < marketOpenOffsetSec) {

                barTime -= 86400;

            }

        } else {

            // For smaller timeframes, just use regular alignment

            barTime = Math.floor(row.time / timeframeSec) * timeframeSec;

        }

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

  

// ... existing code ...

```

  

This updated function:

  

1. Adds special handling for hourly or longer timeframes

2. Calculates the market open offset (9:15 AM IST) in seconds from midnight

3. For each data point, calculates how many seconds have passed since the market open

4. Aligns the bar time to the appropriate timeframe boundary relative to market open

5. For smaller timeframes (less than 1 hour), it continues to use the regular alignment

  

This should ensure that your hourly candles start at 9:15, 10:15, etc., rather than at 9:00, 10:00, etc.

  

---

  

_**User**_

  

no tell me what is issue with resmapling or whatevr that chart is taking staring point from 9:00 not 9:15 as first data point is 9:15 so if for any timeframe that take first cnadle it should take 9:15 as open

  

---

  

_**Assistant**_

  
  
  

I understand the issue now. The problem is that the resampling function is calculating bar times based on standard time boundaries (like 9:00 for hourly candles) rather than using the actual data's start time (9:15).

  

Let's simplify the approach. Since your data starts at 9:15, we should use the first data point's time as our reference point for alignment:

  

```javascript:feb26/MAR26/sbk1.py

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

```

  

This approach:

  

1. Uses the first data point's time (9:15) as the reference point

2. Calculates how many timeframe periods have passed since that reference point

3. Aligns each bar to a multiple of the timeframe period from the reference point

  

This way, if your data starts at 9:15, the hourly candles will be at 9:15, 10:15, etc., because we're using 9:15 as our reference point rather than trying to align to standard hour boundaries.

  

---

  

_**User**_

  

how right what additon you maade to make it work

  

---

  

_**Assistant**_

  
  
  

The key changes I made to fix the resampling issue were:

  

1. **Using the first data point as a reference point**: Instead of trying to align candles to standard time boundaries (like 9:00 for hourly candles), I used the first data point's time (which is 9:15) as the reference point for all calculations.

  

2. **Calculating periods since the start**: I calculated how many timeframe periods have passed since that first data point using:

   ```javascript

   const periodsSinceStart = Math.floor((row.time - firstDataTime) / timeframeSec);

   ```

  

3. **Aligning bar times relative to the first data point**: Instead of using absolute time boundaries, I calculated each bar's time as an offset from the first data point:

   ```javascript

   const barTime = firstDataTime + (periodsSinceStart * timeframeSec);

   ```

  

4. **Added a null check** for empty data arrays to prevent errors.

  

This approach ensures that all timeframes are aligned relative to your actual data's start time (9:15), rather than trying to force alignment to standard clock boundaries. So hourly candles will be at 9:15, 10:15, 11:15, etc., and daily candles will start at 9:15 each day, which matches the actual market session start time.

  

The solution is elegant because it adapts to whatever your first data point is, rather than hardcoding specific market hours.

  

---

  
  
  

---