# arr.map usage

# arr.reduce usage

````js
array.reduce((accumulator, currentValue) => {
    // Process currentValue and update accumulator
    return accumulator;
}, initialValue);


array.reduce(function(accumulator, currentValue) {
    // Process currentValue and update accumulator
    return accumulator;
}, initialValue);

````

|Iteration|`total` (accumulator)|`num` (current element)|Computation (`total + num`)|New `total`|
|---------|---------------------|-----------------------|---------------------------|-----------|
|1|0|1|`0 + 1 = 1`|1|
|2|1|2|`1 + 2 = 3`|3|
|3|3|3|`3 + 3 = 6`|6|
|4|6|4|`6 + 4 = 10`|10|

In your case, `accumulator = resampledData`, and `initialValue = []` (an empty array).

### **Why `[]`?**

* `resampledData` needs to start as **an empty array** because we are **building** a new dataset.

* If we didn’t provide `[]`, `reduce()` would try to use the **first element of `data` as the initial value**, which wouldn't work for resampling.

|teration|`accumulator` (resampledData)|`num` (current row)|Computation (New Bar Created or Existing Updated)|New `accumulator`|
|--------|-----------------------------|-------------------|-------------------------------------------------|-----------------|
|**1**|`[]`|`{time: 1700000000, open: 100, high: 102, low: 99, close: 101}`|**Start new bar** (Time = 1700000000)|`[ {time: 1700000000, open: 100, high: 102, low: 99, close: 101} ]`|
|**2**|`[ {1700000000, 100, 102, 99, 101} ]`|`{1700000005, 105, 107, 104, 106}`|**Update high/low/close**|`[ {1700000000, 100, 107, 99, 106} ]`|
|**3**|`[ {1700000000, 100, 107, 99, 106} ]`|`{1700000010, 109, 111, 108, 110}`|**Update high/low/close**|`[ {1700000000, 100, 111, 99, 110} ]`|
|**4**|`[ {1700000000, 100, 111, 99, 110} ]`|`{1700000015, 112, 114, 111, 113}`|**Update high/low/close**|`[ {1700000000, 100, 114, 99, 113} ]`|
|**5**|`[ {1700000000, 100, 114, 99, 113} ]`|`{1700000020, 115, 117, 114, 116}`|**Start new bar** (Time = 1700000020)|`[ {1700000000, 100, 114, 99, 113}, {1700000020, 115, 117, 114, 116} ]`|
|**6**|`[ {1700000000, 100, 114, 99, 113}, {1700000020, 115, 117, 114, 116} ]`|`{1700000025, 118, 120, 117, 119}`|**Update high/low/close**|`[ {1700000000, 100, 114, 99, 113}, {1700000020, 115, 120, 114, 119} ]`|
