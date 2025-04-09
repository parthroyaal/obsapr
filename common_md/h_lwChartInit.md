[h_impWithVol](h_impWithVol.md)

[h_clientSideadvChart](h_clientSideadvChart.md)
 start at 9:00 instead of 9:15, because it's rounding down to the nearest hour boundary in Unix time.
[h_serverb3BkResample](h_serverb3BkResample.md)  [h_serverb3BkResample_tmp](h_serverb3BkResample_tmp.md)

# resample block

````html
<script>
// Convert a timeframe value (e.g., "5S") into milliseconds
function getTimeframeMs(timeframe) {
    const unit = timeframe.slice(-1);
    const value = parseInt(timeframe);
    if (unit === 'S') {
        return value * 1000;
    }
    return value * 60 * 1000; // Default to minutes
    }

function resampleData(data, timeframe) {
    if (data.length === 0) return [];
    
    const timeframeMs = getTimeframeMs(timeframe);
    console.log(timeframeMs)
    const timeframeSec = timeframeMs / 1000;
    const firstDataTime = data[0].time;

    return data.reduce(function(resampledData, row)  {
        const periodsSinceStart = Math.floor((row.time - firstDataTime) / timeframeSec);
        const barTime = firstDataTime + (periodsSinceStart * timeframeSec);
        
        let currentBar = resampledData[resampledData.length - 1];  // Get the last bar

        if (!currentBar || currentBar.time !== barTime) {
            // Start a new candle
            resampledData.push({
                time: barTime,
                open: row.open,
                high: row.high,
                low: row.low,
                close: row.close
            });
        } else {
            // Update existing candle
            currentBar.high = Math.max(currentBar.high, row.high);
            currentBar.low = Math.min(currentBar.low, row.low);
            currentBar.close = row.close;
        }

        return resampledData;
    }, []);
}

// Resample to 5-second candles
const resampled = resampleData(data, '20s');
console.log(resampled);
</script>
````

# old imp

````html
<script>
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

</script>
````

# Specstory

---

Now I understand! The issue is in how the time boundaries are calculated:

1. In your first approach:

````javascript

const barTime = Math.floor(row.time / timeframeSec) * timeframeSec;

````

This aligns candles to absolute Unix time boundaries regardless of when your data starts. So for hourly candles, it will always use 9:00, 10:00, etc. (standard hour marks) even though your market opens at 9:15.

2. In your working approach:

````javascript

const periodsSinceStart = Math.floor((row.time - firstDataTime) / timeframeSec);

const barTime = firstDataTime + (periodsSinceStart * timeframeSec);

````

This correctly uses the first data point (9:15) as the anchor point, creating candles at 9:15, 10:15, 11:15, etc.

The first method rounds timestamps to the nearest absolute timeframe boundary in Unix time, while the second method creates timeframe boundaries relative to your first data point (market open time).

For market data, the second approach is correct because you want candles aligned to market hours, not arbitrary Unix time boundaries.

---

---
