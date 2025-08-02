# Module 13: Slices Part 3: Slicing Operations and References

**Duration**: 1 hour (45 min theory + 15 min hands-on)

**Learning Objectives**:

- Master slice operations and sub-slicing
- Understand slice sharing and reference behavior
- Learn safe slicing practices to avoid memory leaks
- Apply advanced slicing techniques in automation

**Videos Covered**:

- 3.7 Slices Part 3 (Taking Slices of Slices) (0:11:45)
- 3.8 Slices Part 4 (Slices and References) (0:05:51)

**Key Concepts**:

- Slice expressions and bounds checking
- Shared backing arrays and memory implications
- Copy vs slice operations
- Memory leaks with large slice references
- Safe slicing patterns

**Hands-on Exercise 1: Safe Data Windowing for Time-Series Automation**:

```go
// Safe data windowing for time-series automation
package main

import (
    "fmt"
    "time"
)

type TimeSeriesData struct {
    timestamps []int64
    values     []float64
}

func NewTimeSeriesData(capacity int) *TimeSeriesData {
    return &TimeSeriesData{
        timestamps: make([]int64, 0, capacity),
        values:     make([]float64, 0, capacity),
    }
}

func (ts *TimeSeriesData) AddPoint(timestamp int64, value float64) {
    ts.timestamps = append(ts.timestamps, timestamp)
    ts.values = append(ts.values, value)
}

// Safe windowing - creates copy to avoid memory leaks
func (ts *TimeSeriesData) GetWindow(start, end int) *TimeSeriesData {
    if start < 0 || end > len(ts.timestamps) || start >= end {
        return NewTimeSeriesData(0)
    }

    window := NewTimeSeriesData(end - start)

    // Copy data instead of sharing slice
    for i := start; i < end; i++ {
        window.AddPoint(ts.timestamps[i], ts.values[i])
    }

    return window
}

// Unsafe windowing - shares backing array (for comparison)
func (ts *TimeSeriesData) GetWindowUnsafe(start, end int) *TimeSeriesData {
    if start < 0 || end > len(ts.timestamps) || start >= end {
        return NewTimeSeriesData(0)
    }

    return &TimeSeriesData{
        timestamps: ts.timestamps[start:end], // Shares backing array
        values:     ts.values[start:end],     // Shares backing array
    }
}

func (ts *TimeSeriesData) CalculateAverage() float64 {
    if len(ts.values) == 0 {
        return 0
    }

    var sum float64
    for _, value := range ts.values {
        sum += value
    }

    return sum / float64(len(ts.values))
}

func (ts *TimeSeriesData) Size() int {
    return len(ts.timestamps)
}

func main() {
    // Create large time series dataset
    data := NewTimeSeriesData(10000)

    // Add sample data points
    baseTime := time.Now().Unix()
    for i := 0; i < 1000; i++ {
        timestamp := baseTime + int64(i*60) // Every minute
        value := 50.0 + float64(i%100)     // Varying values
        data.AddPoint(timestamp, value)
    }

    fmt.Printf("Original dataset size: %d points\n", data.Size())

    // Get safe window (last 100 points)
    safeWindow := data.GetWindow(900, 1000)
    fmt.Printf("Safe window size: %d points\n", safeWindow.Size())
    fmt.Printf("Safe window average: %.2f\n", safeWindow.CalculateAverage())

    // Get unsafe window (for comparison)
    unsafeWindow := data.GetWindowUnsafe(900, 1000)
    fmt.Printf("Unsafe window size: %d points\n", unsafeWindow.Size())
    fmt.Printf("Unsafe window average: %.2f\n", unsafeWindow.CalculateAverage())

    // Demonstrate slice sharing issue
    fmt.Println("\nDemonstrating slice sharing:")

    // Modify original data
    data.timestamps[950] = 999999
    data.values[950] = -1.0

    fmt.Printf("After modifying original data:\n")
    fmt.Printf("Safe window average: %.2f (unchanged)\n", safeWindow.CalculateAverage())
    fmt.Printf("Unsafe window average: %.2f (changed!)\n", unsafeWindow.CalculateAverage())
}
```

**Hands-on Exercise 2: Advanced Slice Operations and Memory Management**:

```go
// Advanced slice operations demonstrating memory efficiency
package main

import (
    "fmt"
    "runtime"
    "unsafe"
)

type DataProcessor struct {
    buffer []byte
    chunks [][]byte
}

func NewDataProcessor() *DataProcessor {
    return &DataProcessor{
        buffer: make([]byte, 0, 1024*1024), // 1MB buffer
        chunks: make([][]byte, 0, 100),
    }
}

func (dp *DataProcessor) ProcessData(data []byte) {
    // Add data to buffer
    dp.buffer = append(dp.buffer, data...)

    // Create chunks using slice operations
    chunkSize := 1024
    for len(dp.buffer) >= chunkSize {
        // UNSAFE: This shares the backing array
        unsafeChunk := dp.buffer[:chunkSize]

        // SAFE: This creates a copy
        safeChunk := make([]byte, chunkSize)
        copy(safeChunk, dp.buffer[:chunkSize])

        dp.chunks = append(dp.chunks, safeChunk)

        // Remove processed data from buffer
        dp.buffer = dp.buffer[chunkSize:]

        _ = unsafeChunk // Prevent unused variable error
    }
}

func (dp *DataProcessor) GetStats() (int, int, int) {
    return len(dp.buffer), cap(dp.buffer), len(dp.chunks)
}

func main() {
    demonstrateSliceExpressions()
    demonstrateMemoryLeaks()
    demonstrateSliceCopy()
}

func demonstrateSliceExpressions() {
    fmt.Println("=== Slice Expressions ===")

    data := []int{0, 1, 2, 3, 4, 5, 6, 7, 8, 9}
    fmt.Printf("Original: %v (len=%d, cap=%d)\n", data, len(data), cap(data))

    // Basic slice expressions
    slice1 := data[2:7]    // [2, 3, 4, 5, 6]
    slice2 := data[:5]     // [0, 1, 2, 3, 4]
    slice3 := data[3:]     // [3, 4, 5, 6, 7, 8, 9]
    slice4 := data[:]      // [0, 1, 2, 3, 4, 5, 6, 7, 8, 9]

    fmt.Printf("data[2:7]: %v (len=%d, cap=%d)\n", slice1, len(slice1), cap(slice1))
    fmt.Printf("data[:5]:  %v (len=%d, cap=%d)\n", slice2, len(slice2), cap(slice2))
    fmt.Printf("data[3:]:  %v (len=%d, cap=%d)\n", slice3, len(slice3), cap(slice3))
    fmt.Printf("data[:]:   %v (len=%d, cap=%d)\n", slice4, len(slice4), cap(slice4))

    // Full slice expressions (with capacity limit)
    slice5 := data[2:7:7]  // [2, 3, 4, 5, 6] with cap=5
    slice6 := data[1:5:6]  // [1, 2, 3, 4] with cap=5

    fmt.Printf("data[2:7:7]: %v (len=%d, cap=%d)\n", slice5, len(slice5), cap(slice5))
    fmt.Printf("data[1:5:6]: %v (len=%d, cap=%d)\n", slice6, len(slice6), cap(slice6))

    // Demonstrate capacity calculation
    fmt.Println("\nCapacity calculation:")
    fmt.Printf("data[2:7] capacity = %d (from index 2 to end: %d-2=%d)\n",
        cap(slice1), len(data), len(data)-2)
    fmt.Printf("data[2:7:7] capacity = %d (explicitly limited)\n", cap(slice5))
}

func demonstrateMemoryLeaks() {
    fmt.Println("\n=== Memory Leak Prevention ===")

    // Create a large slice
    largeData := make([]byte, 1000000) // 1MB
    for i := range largeData {
        largeData[i] = byte(i % 256)
    }

    fmt.Printf("Large data: %d bytes\n", len(largeData))

    // BAD: Taking a small slice keeps the entire backing array
    badSlice := largeData[0:100]
    fmt.Printf("Bad slice: len=%d, but keeps %d bytes in memory\n",
        len(badSlice), cap(badSlice))

    // GOOD: Copy only what you need
    goodSlice := make([]byte, 100)
    copy(goodSlice, largeData[0:100])
    fmt.Printf("Good slice: len=%d, uses only %d bytes\n",
        len(goodSlice), cap(goodSlice))

    // Now we can release the large data
    largeData = nil
    runtime.GC()

    // Demonstrate with string slicing (similar issue)
    largeString := string(make([]byte, 1000000))

    // BAD: String slice shares backing array
    badSubstring := largeString[0:100]
    fmt.Printf("Bad substring: len=%d, but references %d bytes\n",
        len(badSubstring), len(largeString))

    // GOOD: Create new string
    goodSubstring := string([]byte(largeString[0:100]))
    fmt.Printf("Good substring: len=%d, independent copy\n", len(goodSubstring))

    // Show memory addresses to prove sharing vs copying
    fmt.Printf("\nMemory addresses:\n")
    fmt.Printf("Bad slice data ptr: %p\n", unsafe.Pointer(&badSlice[0]))
    fmt.Printf("Good slice data ptr: %p\n", unsafe.Pointer(&goodSlice[0]))
}

func demonstrateSliceCopy() {
    fmt.Println("\n=== Safe Slice Copying Patterns ===")

    source := []string{"task1", "task2", "task3", "task4", "task5"}

    // Method 1: Using copy() function
    dest1 := make([]string, len(source))
    n := copy(dest1, source)
    fmt.Printf("copy() method: copied %d elements: %v\n", n, dest1)

    // Method 2: Using append with empty slice
    var dest2 []string
    dest2 = append(dest2, source...)
    fmt.Printf("append() method: %v\n", dest2)

    // Method 3: Partial copying
    dest3 := make([]string, 3)
    copy(dest3, source[1:4]) // Copy middle 3 elements
    fmt.Printf("Partial copy: %v\n", dest3)

    // Method 4: Copy with different types ([]byte to string)
    byteData := []byte("Hello, World!")
    stringData := string(byteData) // Creates a copy
    fmt.Printf("Byte to string: %s\n", stringData)

    // Modify original to show independence
    source[0] = "modified"
    byteData[0] = 'h'

    fmt.Printf("After modification:\n")
    fmt.Printf("  Original source: %v\n", source)
    fmt.Printf("  dest1 (copy): %v\n", dest1)
    fmt.Printf("  dest2 (append): %v\n", dest2)
    fmt.Printf("  Original bytes: %s\n", string(byteData))
    fmt.Printf("  String copy: %s\n", stringData)

    // Demonstrate efficient slice operations
    fmt.Println("\n=== Efficient Slice Operations ===")

    // Remove element from slice (efficient method)
    data := []int{1, 2, 3, 4, 5, 6, 7, 8, 9, 10}
    indexToRemove := 4 // Remove element at index 4 (value 5)

    fmt.Printf("Before removal: %v\n", data)

    // Efficient removal (order doesn't matter)
    data[indexToRemove] = data[len(data)-1] // Move last element to index
    data = data[:len(data)-1]               // Shrink slice

    fmt.Printf("After efficient removal: %v\n", data)

    // Insert element in slice
    data = []int{1, 2, 3, 4, 5}
    insertIndex := 2
    insertValue := 99

    fmt.Printf("Before insertion: %v\n", data)

    // Insert by growing slice and shifting elements
    data = append(data, 0)                    // Grow slice
    copy(data[insertIndex+1:], data[insertIndex:]) // Shift elements
    data[insertIndex] = insertValue           // Insert new value

    fmt.Printf("After insertion: %v\n", data)
}
```

**Hands-on Exercise 3: Slice Performance Optimization Patterns**:

```go
// Performance optimization patterns for slice operations
package main

import (
    "fmt"
    "runtime"
    "time"
)

type PerformanceTest struct {
    name     string
    function func() interface{}
}

func main() {
    runPerformanceTests()
    demonstrateOptimizationPatterns()
}

func runPerformanceTests() {
    fmt.Println("=== Slice Performance Tests ===")

    tests := []PerformanceTest{
        {"Append without preallocation", testAppendNoPrealloc},
        {"Append with preallocation", testAppendWithPrealloc},
        {"Copy vs slice sharing", testCopyVsSlice},
        {"Range vs index access", testRangeVsIndex},
    }

    for _, test := range tests {
        fmt.Printf("\nTesting: %s\n", test.name)

        // Warm up
        test.function()

        // Measure performance
        start := time.Now()
        result := test.function()
        duration := time.Since(start)

        fmt.Printf("Duration: %v\n", duration)

        if slice, ok := result.([]int); ok {
            fmt.Printf("Result length: %d\n", len(slice))
        }
    }
}

func testAppendNoPrealloc() interface{} {
    var slice []int
    for i := 0; i < 100000; i++ {
        slice = append(slice, i)
    }
    return slice
}

func testAppendWithPrealloc() interface{} {
    slice := make([]int, 0, 100000)
    for i := 0; i < 100000; i++ {
        slice = append(slice, i)
    }
    return slice
}

func testCopyVsSlice() interface{} {
    source := make([]int, 100000)
    for i := range source {
        source[i] = i
    }

    // Test copying
    dest := make([]int, len(source))
    copy(dest, source)

    return dest
}

func testRangeVsIndex() interface{} {
    slice := make([]int, 100000)
    for i := range slice {
        slice[i] = i
    }

    // Test range access
    sum := 0
    for _, v := range slice {
        sum += v
    }

    return sum
}

func demonstrateOptimizationPatterns() {
    fmt.Println("\n=== Optimization Patterns ===")

    // Pattern 1: Pool slices for reuse
    demonstrateSlicePooling()

    // Pattern 2: Batch operations
    demonstrateBatchOperations()

    // Pattern 3: Memory-efficient filtering
    demonstrateEfficientFiltering()
}

func demonstrateSlicePooling() {
    fmt.Println("\n--- Slice Pooling Pattern ---")

    // Simple slice pool
    type SlicePool struct {
        pool [][]int
    }

    pool := &SlicePool{
        pool: make([][]int, 0, 10),
    }

    getSlice := func() []int {
        if len(pool.pool) > 0 {
            slice := pool.pool[len(pool.pool)-1]
            pool.pool = pool.pool[:len(pool.pool)-1]
            return slice[:0] // Reset length but keep capacity
        }
        return make([]int, 0, 100)
    }

    putSlice := func(slice []int) {
        if cap(slice) >= 100 {
            pool.pool = append(pool.pool, slice)
        }
    }

    // Use the pool
    for i := 0; i < 5; i++ {
        slice := getSlice()

        // Use the slice
        for j := 0; j < 50; j++ {
            slice = append(slice, j)
        }

        fmt.Printf("Iteration %d: len=%d, cap=%d\n", i+1, len(slice), cap(slice))

        // Return to pool
        putSlice(slice)
    }

    fmt.Printf("Pool size after use: %d\n", len(pool.pool))
}

func demonstrateBatchOperations() {
    fmt.Println("\n--- Batch Operations Pattern ---")

    // Process data in batches to reduce allocations
    data := make([]int, 10000)
    for i := range data {
        data[i] = i
    }

    batchSize := 1000
    var results [][]int

    start := time.Now()

    for i := 0; i < len(data); i += batchSize {
        end := i + batchSize
        if end > len(data) {
            end = len(data)
        }

        // Process batch
        batch := make([]int, 0, end-i)
        for j := i; j < end; j++ {
            if data[j]%2 == 0 { // Filter even numbers
                batch = append(batch, data[j]*2)
            }
        }

        if len(batch) > 0 {
            results = append(results, batch)
        }
    }

    duration := time.Since(start)

    totalProcessed := 0
    for _, batch := range results {
        totalProcessed += len(batch)
    }

    fmt.Printf("Processed %d items in %d batches\n", totalProcessed, len(results))
    fmt.Printf("Batch processing time: %v\n", duration)
}

func demonstrateEfficientFiltering() {
    fmt.Println("\n--- Efficient Filtering Pattern ---")

    source := make([]int, 10000)
    for i := range source {
        source[i] = i
    }

    // Method 1: Append-based filtering (creates new slice)
    start := time.Now()
    var filtered1 []int
    for _, v := range source {
        if v%3 == 0 {
            filtered1 = append(filtered1, v)
        }
    }
    method1Time := time.Since(start)

    // Method 2: In-place filtering (reuses slice)
    start = time.Now()
    filtered2 := source[:0] // Reuse backing array
    for _, v := range source {
        if v%3 == 0 {
            filtered2 = append(filtered2, v)
        }
    }
    method2Time := time.Since(start)

    // Method 3: Pre-allocated filtering
    start = time.Now()
    filtered3 := make([]int, 0, len(source)/3) // Estimate capacity
    for _, v := range source {
        if v%3 == 0 {
            filtered3 = append(filtered3, v)
        }
    }
    method3Time := time.Since(start)

    fmt.Printf("Method 1 (append): %d items, %v\n", len(filtered1), method1Time)
    fmt.Printf("Method 2 (in-place): %d items, %v\n", len(filtered2), method2Time)
    fmt.Printf("Method 3 (pre-alloc): %d items, %v\n", len(filtered3), method3Time)

    // Show memory usage
    var m runtime.MemStats
    runtime.ReadMemStats(&m)
    fmt.Printf("Current heap: %d KB\n", m.HeapAlloc/1024)

    // Performance tips
    fmt.Println("\nPerformance Tips:")
    fmt.Println("- Pre-allocate slices when size is predictable")
    fmt.Println("- Reuse slices by resetting length to 0")
    fmt.Println("- Use copy() for safe slice copying")
    fmt.Println("- Consider slice pooling for frequent allocations")
    fmt.Println("- Batch operations to reduce allocation overhead")
    fmt.Println("- Use in-place operations when original data isn't needed")
}
```

**Prerequisites**: Module 12
