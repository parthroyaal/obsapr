
````html
<script>

    let historicalData = [
    
        { timestamp: '2024-02-15T10:00:00Z', open: 100, high: 102, low: 98, close: 101 },
    
        { timestamp: '2024-02-15T10:00:05Z', open: 101, high: 103, low: 99, close: 102 },
    
        { timestamp: '2024-02-15T10:00:10Z', open: 102, high: 104, low: 100, close: 103 },
    
        { timestamp: '2024-02-15T10:00:15Z', open: 103, high: 105, low: 101, close: 104 },
    
        { timestamp: '2024-02-15T10:00:20Z', open: 104, high: 106, low: 102, close: 105 },
    
        { timestamp: '2024-02-15T10:00:25Z', open: 105, high: 107, low: 103, close: 106 },
    
    
    ];
    
    
    
    // Convert timestamps to time in milliseconds
    
    historicalData = historicalData.map(data => ({
    
        time: new Date(data.timestamp).getTime(),
    
        open: data.open,
    
        high: data.high,
    
        low: data.low,
    
        close: data.close
    
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
    
</script>

````
