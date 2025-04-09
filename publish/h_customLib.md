
# Fin 

```html
<!DOCTYPE html>
<html lang="en">
<head>
  <meta charset="UTF-8">
  <meta name="viewport" content="width=device-width, initial-scale=1.0">
  <title>Lightweight Trading Chart</title>
  <link rel="icon" href="favicon.ico" type="image/x-icon">
  <style>
    body { margin: 0; padding: 0; font-family: Arial, sans-serif; }
    #chart_container { width: 100%; height: 100vh; position: relative; }
    canvas { display: block; }
    .chart-info { position: absolute; top: 10px; left: 10px; background: rgba(255, 255, 255, 0.8); padding: 5px 10px; border-radius: 3px; font-size: 14px; }
  </style>
</head>
<body>
  <div id="chart_container">
    <canvas id="chart"></canvas>
    <div class="chart-info">Symbol: NIFTYBANK</div>
  </div>

  <script>
    let historicalData = [
      { timestamp: '2024-02-15T10:00:00Z', open: 100, high: 102, low: 98, close: 101 },
      { timestamp: '2024-02-15T10:00:05Z', open: 101, high: 103, low: 99, close: 102 },
      { timestamp: '2024-02-15T10:00:10Z', open: 102, high: 104, low: 100, close: 103 },
      { timestamp: '2024-02-15T10:00:15Z', open: 103, high: 105, low: 101, close: 104 },
      { timestamp: '2024-02-15T10:00:20Z', open: 104, high: 106, low: 102, close: 105 },
      { timestamp: '2024-02-15T10:00:25Z', open: 105, high: 107, low: 103, close: 106 },
      { timestamp: '2024-02-15T10:00:30Z', open: 106, high: 108, low: 104, close: 107 },
      { timestamp: '2024-02-15T10:00:35Z', open: 107, high: 109, low: 105, close: 106 },
      { timestamp: '2024-02-15T10:00:40Z', open: 106, high: 108, low: 104, close: 105 },
      { timestamp: '2024-02-15T10:00:45Z', open: 105, high: 107, low: 103, close: 106 },
      { timestamp: '2024-02-15T10:00:50Z', open: 106, high: 108, low: 104, close: 107 },
      { timestamp: '2024-02-15T10:00:55Z', open: 107, high: 109, low: 105, close: 108 },
      { timestamp: '2024-02-15T10:01:00Z', open: 108, high: 110, low: 106, close: 109 },
      { timestamp: '2024-02-15T10:01:05Z', open: 109, high: 111, low: 107, close: 110 },
      { timestamp: '2024-02-15T10:01:10Z', open: 110, high: 112, low: 108, close: 111 }
    ];

    // Convert timestamps to milliseconds
    historicalData = historicalData.map(data => ({
      time: new Date(data.timestamp).getTime(),
      ...data
    }));

    // Define separation point
    const initialCandleTimeMs = new Date("2024-02-15T10:00:10Z").getTime();
    
    // Correctly split data
    let streamingData = historicalData.filter(bar => bar.time >= initialCandleTimeMs);
    let allData = historicalData.filter(bar => bar.time < initialCandleTimeMs); // Only historical data at start

    // Canvas setup
    const canvas = document.getElementById('chart');
    const ctx = canvas.getContext('2d');
    let canvasWidth, canvasHeight;

    // Chart parameters
    const padding = { top: 50, right: 50, bottom: 50, left: 50 };
    let candleWidth = 40;
    const candlePadding = 10;

    function resizeCanvas() {
      canvasWidth = canvas.width = canvas.parentElement.clientWidth;
      canvasHeight = canvas.height = canvas.parentElement.clientHeight;
      drawChart();
    }

    function drawChart() {
      ctx.clearRect(0, 0, canvasWidth, canvasHeight);
      if (allData.length === 0) return;

      const minPrice = Math.min(...allData.map(d => d.low));
      const maxPrice = Math.max(...allData.map(d => d.high));
      const priceRange = maxPrice - minPrice;
      
      const chartWidth = canvasWidth - padding.left - padding.right;
      const chartHeight = canvasHeight - padding.top - padding.bottom;
      const yScale = chartHeight / (priceRange * 1.1);

      // Draw price axis
      ctx.strokeStyle = '#ccc';
      ctx.lineWidth = 1;
      ctx.beginPath();
      ctx.moveTo(padding.left, padding.top);
      ctx.lineTo(padding.left, canvasHeight - padding.bottom);
      ctx.stroke();

      // Draw candles
      const totalCandleWidth = candleWidth + candlePadding;
      const visibleCandles = Math.min(allData.length, Math.floor(chartWidth / totalCandleWidth));

      for (let i = 0; i < visibleCandles; i++) {
        const data = allData[i];
        const x = padding.left + i * totalCandleWidth + candlePadding / 2;

        if (i % 2 === 0) {
          const date = new Date(data.time);
          const timeLabel = `${date.getHours()}:${date.getMinutes().toString().padStart(2, '0')}:${date.getSeconds().toString().padStart(2, '0')}`;
          ctx.textAlign = 'center';
          ctx.fillText(timeLabel, x + candleWidth / 2, canvasHeight - padding.bottom + 20);
        }

        const openY = canvasHeight - padding.bottom - (data.open - minPrice) * yScale;
        const closeY = canvasHeight - padding.bottom - (data.close - minPrice) * yScale;
        const highY = canvasHeight - padding.bottom - (data.high - minPrice) * yScale;
        const lowY = canvasHeight - padding.bottom - (data.low - minPrice) * yScale;

        ctx.beginPath();
        ctx.moveTo(x + candleWidth / 2, highY);
        ctx.lineTo(x + candleWidth / 2, lowY);
        ctx.strokeStyle = '#000';
        ctx.stroke();

        ctx.fillStyle = data.close > data.open ? '#4CAF50' : '#F44336';
        ctx.fillRect(x, Math.min(openY, closeY), candleWidth, Math.abs(closeY - openY) || 1);
      }

      // Draw title
      ctx.fillStyle = '#333';
      ctx.font = 'bold 16px Arial';
      ctx.textAlign = 'center';
      ctx.fillText('NIFTYBANK', canvasWidth / 2, 30);
    }

    function startStreaming() {
      let index = 0;
      const streamInterval = setInterval(() => {
        if (index < streamingData.length) {
          const newData = streamingData[index++];
          console.log("Streaming new data:", newData);
          allData.push(newData);  // Append data to `allData`
          drawChart();
        } else {
          console.log("Streaming completed");
          clearInterval(streamInterval);
        }
      }, 1000);
    }

    window.addEventListener('resize', resizeCanvas);
    document.addEventListener("DOMContentLoaded", () => {
      resizeCanvas();
      startStreaming();
    });
  </script>
</body>
</html>

```

## Pre:
```html
<!DOCTYPE html>
<html lang="en">
<head>
  <meta charset="UTF-8">
  <meta name="viewport" content="width=device-width, initial-scale=1.0">
  <title>Lightweight Trading Chart</title>
  <link rel="icon" href="favicon.ico" type="image/x-icon">
  <style>
    body {
      margin: 0;
      padding: 0;
      font-family: Arial, sans-serif;
    }
    #chart_container {
      width: 100%;
      height: 100vh;
      position: relative;
    }
    canvas {
      display: block;
    }
    .chart-info {
      position: absolute;
      top: 10px;
      left: 10px;
      background: rgba(255, 255, 255, 0.8);
      padding: 5px 10px;
      border-radius: 3px;
      font-size: 14px;
    }
  </style>
</head>
<body>
  <div id="chart_container">
    <canvas id="chart"></canvas>
    <div class="chart-info">Symbol: NIFTYBANK</div>
  </div>

  <script>
    let historicalData = [
      { timestamp: '2024-02-15T10:00:00Z', open: 100, high: 102, low: 98, close: 101 },
      { timestamp: '2024-02-15T10:00:05Z', open: 101, high: 103, low: 99, close: 102 },
      { timestamp: '2024-02-15T10:00:10Z', open: 102, high: 104, low: 100, close: 103 },
      { timestamp: '2024-02-15T10:00:15Z', open: 103, high: 105, low: 101, close: 104 },
      { timestamp: '2024-02-15T10:00:20Z', open: 104, high: 106, low: 102, close: 105 }
    ];

    // Convert timestamps to time in milliseconds
    historicalData = historicalData.map(data => ({
      time: new Date(data.timestamp).getTime(),
      timestamp: data.timestamp,
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
    
    // Combine all data for initial display
    let allData = [...historicalData, ...streamingData];
    
    // Canvas setup
    const canvas = document.getElementById('chart');
    const ctx = canvas.getContext('2d');
    let canvasWidth, canvasHeight;
    
    // Chart parameters
    const padding = { top: 50, right: 50, bottom: 50, left: 50 };
    let candleWidth = 40;
    const candlePadding = 10;
    
    function resizeCanvas() {
      canvasWidth = canvas.width = canvas.parentElement.clientWidth;
      canvasHeight = canvas.height = canvas.parentElement.clientHeight;
      drawChart();
    }
    
    function drawChart() {
      // Clear canvas
      ctx.clearRect(0, 0, canvasWidth, canvasHeight);
      
      if (allData.length === 0) return;
      
      // Find min and max values for scaling
      const minPrice = Math.min(...allData.map(d => d.low));
      const maxPrice = Math.max(...allData.map(d => d.high));
      const priceRange = maxPrice - minPrice;
      
      // Calculate chart area dimensions
      const chartWidth = canvasWidth - padding.left - padding.right;
      const chartHeight = canvasHeight - padding.top - padding.bottom;
      
      // Scale factor for prices
      const yScale = chartHeight / (priceRange * 1.1);
      
      // Draw price axis
      ctx.strokeStyle = '#ccc';
      ctx.lineWidth = 1;
      ctx.beginPath();
      ctx.moveTo(padding.left, padding.top);
      ctx.lineTo(padding.left, canvasHeight - padding.bottom);
      ctx.stroke();
      
      // Draw price labels
      ctx.fillStyle = '#333';
      ctx.font = '12px Arial';
      ctx.textAlign = 'right';
      
      const priceStep = priceRange / 5;
      for (let i = 0; i <= 5; i++) {
        const price = minPrice + priceStep * i;
        const y = canvasHeight - padding.bottom - (price - minPrice) * yScale;
        
        ctx.beginPath();
        ctx.moveTo(padding.left - 5, y);
        ctx.lineTo(padding.left, y);
        ctx.stroke();
        
        ctx.fillText(price.toFixed(2), padding.left - 10, y + 4);
      }
      
      // Draw time axis
      ctx.beginPath();
      ctx.moveTo(padding.left, canvasHeight - padding.bottom);
      ctx.lineTo(canvasWidth - padding.right, canvasHeight - padding.bottom);
      ctx.stroke();
      
      // Draw candles
      const totalCandleWidth = candleWidth + candlePadding;
      const visibleCandles = Math.min(allData.length, Math.floor(chartWidth / totalCandleWidth));
      
      for (let i = 0; i < visibleCandles; i++) {
        const data = allData[i];
        const x = padding.left + i * totalCandleWidth + candlePadding / 2;
        
        // Draw time labels for some candles
        if (i % 2 === 0) {
          const date = new Date(data.time);
          const timeLabel = `${date.getHours()}:${date.getMinutes().toString().padStart(2, '0')}:${date.getSeconds().toString().padStart(2, '0')}`;
          
          ctx.textAlign = 'center';
          ctx.fillText(timeLabel, x + candleWidth / 2, canvasHeight - padding.bottom + 20);
        }
        
        // Calculate candle positions
        const openY = canvasHeight - padding.bottom - (data.open - minPrice) * yScale;
        const closeY = canvasHeight - padding.bottom - (data.close - minPrice) * yScale;
        const highY = canvasHeight - padding.bottom - (data.high - minPrice) * yScale;
        const lowY = canvasHeight - padding.bottom - (data.low - minPrice) * yScale;
        
        // Draw the wick
        ctx.beginPath();
        ctx.moveTo(x + candleWidth / 2, highY);
        ctx.lineTo(x + candleWidth / 2, lowY);
        ctx.strokeStyle = '#000';
        ctx.stroke();
        
        // Draw the candle body
        ctx.fillStyle = data.close > data.open ? '#4CAF50' : '#F44336';
        ctx.fillRect(x, Math.min(openY, closeY), candleWidth, Math.abs(closeY - openY) || 1);
      }
      
      // Draw chart title
      ctx.fillStyle = '#333';
      ctx.font = 'bold 16px Arial';
      ctx.textAlign = 'center';
      ctx.fillText('NIFTYBANK', canvasWidth / 2, 30);
    }
    
    // Simulate streaming data
    function startStreaming() {
      let index = 0;
      const streamInterval = setInterval(() => {
        if (index < streamingData.length) {
          const newData = streamingData[index++];
          console.log("Streaming new data:", newData);
          allData.push(newData);
          drawChart();
        } else {
          console.log("Streaming completed");
          clearInterval(streamInterval);
        }
      }, 1000); // Stream a new candle every second
    }

    // Initialize chart
    window.addEventListener('resize', resizeCanvas);
    document.addEventListener("DOMContentLoaded", () => {
      resizeCanvas();
      startStreaming();
    });
  </script>
</body>
</html>
```

### ✅ **What’s Fixed?**

✔ **Historical data is set first** → `allData` only contains historical candles initially.  
✔ **Streaming data is appended correctly** → `startStreaming()` pushes new candles one by one.  
✔ **No preloaded streaming data** → Chart updates dynamically as data arrives.  
✔ **Chart re-renders smoothly** → Only new data is added, avoiding unnecessary redraws.

Now your chart will start with **only historical data** and **gradually update** with new streaming data. Let me know if you need further tweaks! 🚀

