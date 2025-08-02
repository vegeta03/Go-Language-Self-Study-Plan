# Module 8: Data-Oriented Design Principles

**Duration**: 1 hour (45 min theory + 15 min hands-on)

**Learning Objectives**:

- Master data-oriented design philosophy and its advantages over traditional class-based design
- Understand mechanical sympathy and hardware-aware programming
- Learn to optimize data structures for CPU cache efficiency
- Apply data transformation principles for performance
- Design efficient data layouts for automation workloads
- Understand the relationship between data layout and algorithm performance

**Videos Covered**:

- 3.1 Topics (0:00:41)
- 3.2 Data-Oriented Design (0:04:52)

**Key Concepts**:

**Data-Oriented Design Philosophy**:

- Data drives everything - if you don't understand the data, you don't understand the problem
- Focus on data transformations rather than object behaviors
- Optimize for the common case, not edge cases
- Design data structures based on how they will be accessed and processed
- Separate data from behavior for better performance and maintainability

**Mechanical Sympathy**:

- Understanding how hardware works to write efficient software
- CPU cache hierarchy: L1, L2, L3 caches and main memory
- Cache line size (typically 64 bytes) and its impact on performance
- Prefetching and spatial locality principles
- Memory access patterns that work with hardware, not against it

**Cache-Friendly Data Structures**:

- Struct of Arrays (SoA) vs Array of Structs (AoS)
- Data layout optimization for sequential access
- Minimizing cache misses through better data organization
- Hot/cold data separation
- Padding and alignment considerations

**Data Transformation Approach**:

- Think in terms of data pipelines and transformations
- Batch processing for better cache utilization
- Minimize data movement and copying
- Use appropriate data structures for access patterns
- Mechanical sympathy principles
- Performance implications of data organization

**Hands-on Exercise 1: Efficient Data Structures for Log Processing**:

```go
// Efficient data structures for log processing
package main

import (
    "fmt"
    "time"
    "unsafe"
)

// Poor design - scattered data
type LogEntryPoor struct {
    ID        int64
    Timestamp time.Time
    Level     string
    Message   string
    Source    string
    Tags      map[string]string
}

// Better design - grouped by access patterns
type LogBatch struct {
    // Hot data - frequently accessed together
    IDs        []int64
    Timestamps []int64 // Unix timestamps for better cache locality
    Levels     []uint8 // Encoded levels (0=DEBUG, 1=INFO, etc.)

    // Cold data - less frequently accessed
    Messages []string
    Sources  []string
}

// Level encoding for better memory efficiency
const (
    LevelDebug uint8 = iota
    LevelInfo
    LevelWarn
    LevelError
    LevelFatal
)

func (lb *LogBatch) AddEntry(id int64, timestamp time.Time, level uint8, message, source string) {
    lb.IDs = append(lb.IDs, id)
    lb.Timestamps = append(lb.Timestamps, timestamp.Unix())
    lb.Levels = append(lb.Levels, level)
    lb.Messages = append(lb.Messages, message)
    lb.Sources = append(lb.Sources, source)
}

func (lb *LogBatch) CountByLevel(targetLevel uint8) int {
    count := 0
    // Cache-friendly iteration - only touching level data
    for _, level := range lb.Levels {
        if level == targetLevel {
            count++
        }
    }
    return count
}

func main() {
    // Compare memory usage
    fmt.Printf("LogEntryPoor size: %d bytes\n", unsafe.Sizeof(LogEntryPoor{}))

    // Create efficient log batch
    batch := &LogBatch{}

    // Add sample entries
    now := time.Now()
    for i := 0; i < 1000; i++ {
        batch.AddEntry(
            int64(i),
            now.Add(time.Duration(i)*time.Second),
            LevelInfo,
            fmt.Sprintf("Processing item %d", i),
            "automation-service",
        )
    }

    // Efficient processing - only touches needed data
    errorCount := batch.CountByLevel(LevelError)
    infoCount := batch.CountByLevel(LevelInfo)

    fmt.Printf("Processed %d entries\n", len(batch.IDs))
    fmt.Printf("Info entries: %d, Error entries: %d\n", infoCount, errorCount)
}
```

**Hands-on Exercise 2: Cache-Friendly Data Transformations**:

```go
// Cache-friendly data transformations for automation metrics
package main

import (
    "fmt"
    "math"
    "time"
    "unsafe"
)

// Array of Structs (AoS) - poor cache locality
type MetricEntryAoS struct {
    Timestamp int64
    Value     float64
    ServiceID uint32
    Status    uint8
}

// Struct of Arrays (SoA) - better cache locality
type MetricBatchSoA struct {
    Timestamps []int64
    Values     []float64
    ServiceIDs []uint32
    Statuses   []uint8
    Count      int
}

func (mb *MetricBatchSoA) AddMetric(timestamp int64, value float64, serviceID uint32, status uint8) {
    mb.Timestamps = append(mb.Timestamps, timestamp)
    mb.Values = append(mb.Values, value)
    mb.ServiceIDs = append(mb.ServiceIDs, serviceID)
    mb.Statuses = append(mb.Statuses, status)
    mb.Count++
}

// Cache-friendly operations - only touch needed data
func (mb *MetricBatchSoA) CalculateAverage() float64 {
    if mb.Count == 0 {
        return 0
    }

    var sum float64
    // Sequential access to values array - cache friendly
    for _, value := range mb.Values {
        sum += value
    }

    return sum / float64(mb.Count)
}

func (mb *MetricBatchSoA) CountByStatus(targetStatus uint8) int {
    count := 0
    // Sequential access to status array - cache friendly
    for _, status := range mb.Statuses {
        if status == targetStatus {
            count++
        }
    }
    return count
}

func (mb *MetricBatchSoA) FindAnomalies(threshold float64) []int {
    var anomalies []int

    // Sequential scan through values - optimal cache usage
    for i, value := range mb.Values {
        if math.Abs(value) > threshold {
            anomalies = append(anomalies, i)
        }
    }

    return anomalies
}

// Data transformation pipeline
type MetricProcessor struct {
    inputBatch  *MetricBatchSoA
    outputBatch *MetricBatchSoA
}

func NewMetricProcessor() *MetricProcessor {
    return &MetricProcessor{
        inputBatch:  &MetricBatchSoA{},
        outputBatch: &MetricBatchSoA{},
    }
}

func (mp *MetricProcessor) ProcessBatch() {
    // Clear output batch
    mp.outputBatch.Timestamps = mp.outputBatch.Timestamps[:0]
    mp.outputBatch.Values = mp.outputBatch.Values[:0]
    mp.outputBatch.ServiceIDs = mp.outputBatch.ServiceIDs[:0]
    mp.outputBatch.Statuses = mp.outputBatch.Statuses[:0]
    mp.outputBatch.Count = 0

    // Data transformation pipeline - cache-friendly sequential processing
    for i := 0; i < mp.inputBatch.Count; i++ {
        // Filter: only process valid metrics
        if mp.inputBatch.Statuses[i] == 1 { // 1 = valid
            // Transform: normalize values
            normalizedValue := mp.inputBatch.Values[i] / 100.0

            // Store in output batch
            mp.outputBatch.AddMetric(
                mp.inputBatch.Timestamps[i],
                normalizedValue,
                mp.inputBatch.ServiceIDs[i],
                mp.inputBatch.Statuses[i],
            )
        }
    }
}

// Benchmark different approaches
func benchmarkDataAccess() {
    const numMetrics = 100000

    fmt.Println("=== Benchmarking Data Access Patterns ===")

    // Create AoS data
    metricsAoS := make([]MetricEntryAoS, numMetrics)
    for i := 0; i < numMetrics; i++ {
        metricsAoS[i] = MetricEntryAoS{
            Timestamp: int64(i),
            Value:     float64(i % 100),
            ServiceID: uint32(i % 10),
            Status:    uint8(i % 2),
        }
    }

    // Create SoA data
    metricsSoA := &MetricBatchSoA{}
    for i := 0; i < numMetrics; i++ {
        metricsSoA.AddMetric(
            int64(i),
            float64(i % 100),
            uint32(i % 10),
            uint8(i % 2),
        )
    }

    // Benchmark AoS approach
    start := time.Now()
    var sumAoS float64
    for _, metric := range metricsAoS {
        sumAoS += metric.Value
    }
    durationAoS := time.Since(start)

    // Benchmark SoA approach
    start = time.Now()
    sumSoA := metricsSoA.CalculateAverage() * float64(metricsSoA.Count)
    durationSoA := time.Since(start)

    fmt.Printf("AoS approach: %v (sum: %.2f)\n", durationAoS, sumAoS)
    fmt.Printf("SoA approach: %v (sum: %.2f)\n", durationSoA, sumSoA)
    fmt.Printf("SoA is %.2fx faster\n", float64(durationAoS)/float64(durationSoA))

    // Memory usage comparison
    fmt.Printf("\nMemory usage:\n")
    fmt.Printf("AoS entry size: %d bytes\n", unsafe.Sizeof(MetricEntryAoS{}))
    fmt.Printf("SoA batch overhead: %d bytes\n", unsafe.Sizeof(MetricBatchSoA{}))
}

func main() {
    fmt.Println("=== Cache-Friendly Data Transformations ===")

    // Create processor
    processor := NewMetricProcessor()

    // Add sample data
    now := time.Now().Unix()
    for i := 0; i < 1000; i++ {
        processor.inputBatch.AddMetric(
            now+int64(i),
            float64(i*10+50), // Values between 50-10050
            uint32(i%5),      // 5 different services
            uint8(i%2),       // Alternating valid/invalid
        )
    }

    fmt.Printf("Input batch: %d metrics\n", processor.inputBatch.Count)

    // Process the batch
    processor.ProcessBatch()

    fmt.Printf("Output batch: %d metrics\n", processor.outputBatch.Count)
    fmt.Printf("Average value: %.2f\n", processor.outputBatch.CalculateAverage())

    // Find anomalies
    anomalies := processor.outputBatch.FindAnomalies(50.0)
    fmt.Printf("Found %d anomalies\n", len(anomalies))

    // Benchmark different approaches
    benchmarkDataAccess()
}
```

**Hands-on Exercise 3: Mechanical Sympathy and Hardware-Aware Design**:

```go
// Mechanical sympathy and hardware-aware design for automation systems
package main

import (
    "fmt"
    "runtime"
    "time"
    "unsafe"
)

// Cache line size (typically 64 bytes on modern CPUs)
const CacheLineSize = 64

// Poor design - false sharing
type CounterPoor struct {
    counter1 int64 // These will likely share a cache line
    counter2 int64 // causing false sharing between goroutines
    counter3 int64
    counter4 int64
}

// Better design - cache line padding
type CounterGood struct {
    counter1 int64
    _        [CacheLineSize - 8]byte // Padding to next cache line
    counter2 int64
    _        [CacheLineSize - 8]byte
    counter3 int64
    _        [CacheLineSize - 8]byte
    counter4 int64
    _        [CacheLineSize - 8]byte
}

// Demonstrate cache-friendly vs cache-unfriendly access patterns
func demonstrateCacheEffects() {
    fmt.Println("=== Cache Effects Demonstration ===")

    const size = 1024 * 1024 // 1M elements
    data := make([]int64, size)

    // Initialize data
    for i := range data {
        data[i] = int64(i)
    }

    // Sequential access (cache-friendly)
    start := time.Now()
    var sum1 int64
    for i := 0; i < size; i++ {
        sum1 += data[i]
    }
    sequentialTime := time.Since(start)

    // Random access (cache-unfriendly)
    start = time.Now()
    var sum2 int64
    step := 4096 // Jump by page size to defeat cache
    for i := 0; i < size; i++ {
        index := (i * step) % size
        sum2 += data[index]
    }
    randomTime := time.Since(start)

    fmt.Printf("Sequential access: %v (sum: %d)\n", sequentialTime, sum1)
    fmt.Printf("Random access: %v (sum: %d)\n", randomTime, sum2)
    fmt.Printf("Random is %.2fx slower\n", float64(randomTime)/float64(sequentialTime))
}

// Data structure optimized for automation task processing
type TaskBatch struct {
    // Hot data - frequently accessed together (fits in cache line)
    Count      int32
    Status     int32
    StartTime  int64

    // Padding to separate hot and cold data
    _ [CacheLineSize - 16]byte

    // Cold data - less frequently accessed
    TaskIDs    []string
    Metadata   map[string]interface{}
    Results    []TaskResult
}

type TaskResult struct {
    TaskID    string
    Success   bool
    Duration  time.Duration
    ErrorMsg  string
}

func (tb *TaskBatch) AddTask(taskID string) {
    tb.TaskIDs = append(tb.TaskIDs, taskID)
    tb.Count++
}

func (tb *TaskBatch) CompleteTask(taskID string, success bool, duration time.Duration, errorMsg string) {
    result := TaskResult{
        TaskID:   taskID,
        Success:  success,
        Duration: duration,
        ErrorMsg: errorMsg,
    }
    tb.Results = append(tb.Results, result)
}

func (tb *TaskBatch) GetSuccessRate() float64 {
    if len(tb.Results) == 0 {
        return 0
    }

    var successful int
    for _, result := range tb.Results {
        if result.Success {
            successful++
        }
    }

    return float64(successful) / float64(len(tb.Results))
}

// Memory pool for reducing allocations
type TaskPool struct {
    batches chan *TaskBatch
}

func NewTaskPool(size int) *TaskPool {
    pool := &TaskPool{
        batches: make(chan *TaskBatch, size),
    }

    // Pre-allocate batches
    for i := 0; i < size; i++ {
        batch := &TaskBatch{
            TaskIDs:  make([]string, 0, 100),
            Metadata: make(map[string]interface{}),
            Results:  make([]TaskResult, 0, 100),
        }
        pool.batches <- batch
    }

    return pool
}

func (tp *TaskPool) Get() *TaskBatch {
    select {
    case batch := <-tp.batches:
        // Reset batch for reuse
        batch.Count = 0
        batch.Status = 0
        batch.StartTime = 0
        batch.TaskIDs = batch.TaskIDs[:0]
        batch.Results = batch.Results[:0]
        for k := range batch.Metadata {
            delete(batch.Metadata, k)
        }
        return batch
    default:
        // Pool empty, create new batch
        return &TaskBatch{
            TaskIDs:  make([]string, 0, 100),
            Metadata: make(map[string]interface{}),
            Results:  make([]TaskResult, 0, 100),
        }
    }
}

func (tp *TaskPool) Put(batch *TaskBatch) {
    select {
    case tp.batches <- batch:
        // Successfully returned to pool
    default:
        // Pool full, let GC handle it
    }
}

// Demonstrate data-oriented automation pipeline
func demonstrateDataOrientedPipeline() {
    fmt.Println("\n=== Data-Oriented Automation Pipeline ===")

    pool := NewTaskPool(10)

    // Process multiple batches
    for batchNum := 0; batchNum < 5; batchNum++ {
        batch := pool.Get()
        batch.StartTime = time.Now().UnixNano()

        // Add tasks to batch
        for i := 0; i < 20; i++ {
            taskID := fmt.Sprintf("batch-%d-task-%d", batchNum, i)
            batch.AddTask(taskID)
        }

        // Simulate task processing
        for _, taskID := range batch.TaskIDs {
            // Simulate work
            time.Sleep(time.Microsecond * 10)

            // Random success/failure
            success := (len(taskID) % 3) != 0
            duration := time.Duration(len(taskID)) * time.Microsecond
            errorMsg := ""
            if !success {
                errorMsg = "Simulated failure"
            }

            batch.CompleteTask(taskID, success, duration, errorMsg)
        }

        fmt.Printf("Batch %d: %d tasks, %.1f%% success rate\n",
            batchNum, len(batch.TaskIDs), batch.GetSuccessRate()*100)

        // Return batch to pool
        pool.Put(batch)
    }
}

// Analyze memory layout and alignment
func analyzeMemoryLayout() {
    fmt.Println("\n=== Memory Layout Analysis ===")

    fmt.Printf("Cache line size: %d bytes\n", CacheLineSize)
    fmt.Printf("CounterPoor size: %d bytes\n", unsafe.Sizeof(CounterPoor{}))
    fmt.Printf("CounterGood size: %d bytes\n", unsafe.Sizeof(CounterGood{}))
    fmt.Printf("TaskBatch size: %d bytes\n", unsafe.Sizeof(TaskBatch{}))

    // Show field offsets
    var tb TaskBatch
    fmt.Printf("TaskBatch field offsets:\n")
    fmt.Printf("  Count: %d\n", unsafe.Offsetof(tb.Count))
    fmt.Printf("  Status: %d\n", unsafe.Offsetof(tb.Status))
    fmt.Printf("  StartTime: %d\n", unsafe.Offsetof(tb.StartTime))
    fmt.Printf("  TaskIDs: %d\n", unsafe.Offsetof(tb.TaskIDs))
    fmt.Printf("  Metadata: %d\n", unsafe.Offsetof(tb.Metadata))
    fmt.Printf("  Results: %d\n", unsafe.Offsetof(tb.Results))

    // CPU information
    fmt.Printf("\nCPU Information:\n")
    fmt.Printf("  CPUs: %d\n", runtime.NumCPU())
    fmt.Printf("  GOMAXPROCS: %d\n", runtime.GOMAXPROCS(0))
}

func main() {
    fmt.Println("=== Mechanical Sympathy and Hardware-Aware Design ===")

    // Analyze memory layout
    analyzeMemoryLayout()

    // Demonstrate cache effects
    demonstrateCacheEffects()

    // Show data-oriented pipeline
    demonstrateDataOrientedPipeline()

    fmt.Println("\nKey Principles:")
    fmt.Println("1. Understand your hardware (cache lines, memory hierarchy)")
    fmt.Println("2. Design data structures for access patterns")
    fmt.Println("3. Minimize cache misses through sequential access")
    fmt.Println("4. Use padding to prevent false sharing")
    fmt.Println("5. Pool objects to reduce allocation pressure")
    fmt.Println("6. Separate hot and cold data")
}
```

**Prerequisites**: Module 7
