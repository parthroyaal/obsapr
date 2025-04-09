# resample

[h_serverb3BkResample](h_serverb3BkResample.md)
[h_clientSideadvChart](h_clientSideadvChart.md)

## Base

````html
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Live NIFTY Tick Chart (IST)</title>
    <script src="https://cdn.jsdelivr.net/gh/parth-royale/cdn@main/lightweight-charts.standalone.production.js"></script>
</head>
<body>
    <h1>Live NIFTY Tick Chart (IST)</h1>
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

// First, fetch historical data
fetch('/historic')
 .then(response => response.json())
 .then(candles => {
   // Set the initial data
   candleSeries.setData(candles);
   
   // Then connect to WebSocket for real-time updates
   const ws = new WebSocket((location.protocol === "https:" ? "wss://" : "ws://") + location.host + "/ws");
   
   ws.onmessage = function(event) {
       const data = JSON.parse(event.data);
       if (data.candle) {
           // We're receiving a candle directly from the server
           candleSeries.update(data.candle);
       }
   };
 })
 .catch(error => {
   console.error('Error fetching historical data:', error);
 });
</script>
</body>
</html>
````
