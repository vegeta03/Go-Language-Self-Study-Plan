# Module 6: Pointers Part 3: Stack Growth and Garbage Collection

**Duration**: 1 hour (45 min theory + 15 min hands-on)

**Learning Objectives**:

- Master Go's innovative stack growth mechanism and its advantages
- Understand garbage collection fundamentals and the tri-color algorithm
- Learn GC tuning techniques for automation workloads
- Monitor and profile memory usage effectively in Go applications
- Understand the relationship between allocation patterns and GC performance
- Apply memory optimization techniques for long-running automation systems

**Videos Covered**:

- 2.7 Pointers Part 4 (Stack Growth) (0:07:32)
- 2.8 Pointers Part 5 (Garbage Collection) (0:15:13)

**Key Concepts**:

**Stack Growth Mechanics**:

- Go uses segmented stacks that grow and shrink dynamically
- Initial stack size is small (2KB), can grow up to 1GB per goroutine
- Stack growth is handled automatically by the runtime
- Stack overflow is virtually impossible in Go
- Each goroutine has its own stack, enabling massive concurrency
- Stack growth involves copying the entire stack to a larger segment

**Garbage Collection Fundamentals**:

- Go uses a concurrent, tri-color mark-and-sweep collector
- Tri-color algorithm: white (unvisited), gray (visited, children not processed), black (fully processed)
- GC runs concurrently with your program to minimize stop-the-world pauses
- GC is triggered when heap size doubles since last collection (GOGC=100 default)
- Write barriers ensure correctness during concurrent collection

**GC Performance Characteristics**:

- GC latency is proportional to the number of pointers, not heap size
- Fewer pointers = faster GC cycles
- Value-heavy data structures are GC-friendly
- Pointer-heavy data structures increase GC overhead

**GC Tuning and Optimization**:

- GOGC environment variable controls GC frequency (default 100%)
- Lower GOGC = more frequent GC, less memory usage
- Higher GOGC = less frequent GC, more memory usage
- Monitor GC metrics to find optimal settings for your workload
- Consider allocation patterns when designing data structures

**Memory Profiling and Monitoring**:

- Use `go tool pprof` for detailed memory analysis
- GODEBUG=gctrace=1 for GC statistics
- runtime.ReadMemStats() for programmatic monitoring
- Understanding allocation sources and patterns
- Best practices for memory-efficient automation

**Hands-on Exercise 1: GC-Aware Batch Processing System**:

```go
// GC-aware batch processing system
package main

import (
    "fmt"
    "runtime"
    "runtime/debug"
    "time"
)

type BatchProcessor struct {
    batchSize int
    processed int
}

func (bp *BatchProcessor) ProcessBatch(items []string) {
    for _, item := range items {
        // Simulate processing work
        _ = fmt.Sprintf("Processing: %s", item)
        bp.processed++
    }

    // Force GC periodically for demonstration
    if bp.processed%1000 == 0 {
        runtime.GC()
        debug.FreeOSMemory()
    }
}

func main() {
    // Configure GC for automation workload
    debug.SetGCPercent(50) // More aggressive GC

    processor := &BatchProcessor{batchSize: 100}

    // Simulate processing large batches
    start := time.Now()

    for batch := 0; batch < 50; batch++ {
        items := make([]string, processor.batchSize)
        for i := range items {
            items[i] = fmt.Sprintf("item_%d_%d", batch, i)
        }

        processor.ProcessBatch(items)

        // Monitor memory every 10 batches
        if batch%10 == 0 {
            var m runtime.MemStats
            runtime.ReadMemStats(&m)
            fmt.Printf("Batch %d: Heap=%dKB, GC=%d\n",
                batch, m.HeapAlloc/1024, m.NumGC)
        }
    }

    duration := time.Since(start)
    fmt.Printf("Processed %d items in %v\n", processor.processed, duration)
}
```

**Hands-on Exercise 2: Stack Growth and Goroutine Memory Management**:

```go
// Demonstration of stack growth and goroutine memory management
package main

import (
    "fmt"
    "runtime"
    "sync"
    "time"
)

// Recursive function to demonstrate stack growth
func deepRecursion(depth int, maxDepth int, goroutineID int) {
    if depth >= maxDepth {
        return
    }

    // Create some local variables to consume stack space
    var localArray [1024]byte
    localArray[0] = byte(depth % 256)

    // Print stack info every 100 levels
    if depth%100 == 0 {
        var m runtime.MemStats
        runtime.ReadMemStats(&m)
        fmt.Printf("Goroutine %d, Depth %d: Stack size info - Goroutines: %d\n",
            goroutineID, depth, runtime.NumGoroutine())
    }

    // Recurse deeper
    deepRecursion(depth+1, maxDepth, goroutineID)

    // Use the local array to prevent optimization
    _ = localArray[depth%1024]
}

// Function to create many goroutines and observe stack behavior
func createManyGoroutines(count int) {
    var wg sync.WaitGroup

    fmt.Printf("Creating %d goroutines...\n", count)

    for i := 0; i < count; i++ {
        wg.Add(1)
        go func(id int) {
            defer wg.Done()

            // Each goroutine does some work that uses stack space
            deepRecursion(0, 200, id)

            // Simulate some work
            time.Sleep(10 * time.Millisecond)
        }(i)
    }

    // Monitor memory while goroutines are running
    go func() {
        for i := 0; i < 10; i++ {
            var m runtime.MemStats
            runtime.ReadMemStats(&m)
            fmt.Printf("Monitor: Goroutines=%d, Heap=%dKB, Stack=%dKB\n",
                runtime.NumGoroutine(),
                m.HeapAlloc/1024,
                m.StackInuse/1024)
            time.Sleep(50 * time.Millisecond)
        }
    }()

    wg.Wait()
    fmt.Printf("All goroutines completed\n")
}

// Function to demonstrate stack vs heap allocation patterns
func demonstrateStackVsHeap() {
    fmt.Println("=== Stack vs Heap Allocation Patterns ===")

    // Function that keeps data on stack
    stackAllocation := func() {
        var data [1000]int
        for i := range data {
            data[i] = i
        }
        // data stays on stack, automatically cleaned up
    }

    // Function that allocates on heap
    heapAllocation := func() []*int {
        var ptrs []*int
        for i := 0; i < 1000; i++ {
            value := i
            ptrs = append(ptrs, &value) // Forces heap allocation
        }
        return ptrs // Returns pointers, data must survive on heap
    }

    // Measure stack allocations
    var m1, m2 runtime.MemStats
    runtime.ReadMemStats(&m1)

    for i := 0; i < 1000; i++ {
        stackAllocation()
    }

    runtime.ReadMemStats(&m2)
    fmt.Printf("Stack allocations: Heap growth = %d bytes\n",
        m2.HeapAlloc-m1.HeapAlloc)

    // Measure heap allocations
    runtime.ReadMemStats(&m1)

    var results [][]*int
    for i := 0; i < 100; i++ {
        result := heapAllocation()
        results = append(results, result)
    }

    runtime.ReadMemStats(&m2)
    fmt.Printf("Heap allocations: Heap growth = %d bytes\n",
        m2.HeapAlloc-m1.HeapAlloc)

    // Keep results alive to prevent GC
    fmt.Printf("Created %d result sets\n", len(results))
}

func main() {
    fmt.Println("=== Stack Growth and Goroutine Memory Management ===")

    // Show initial memory state
    var m runtime.MemStats
    runtime.ReadMemStats(&m)
    fmt.Printf("Initial state: Goroutines=%d, Heap=%dKB, Stack=%dKB\n",
        runtime.NumGoroutine(),
        m.HeapAlloc/1024,
        m.StackInuse/1024)

    // Demonstrate stack vs heap allocation patterns
    demonstrateStackVsHeap()

    // Create many goroutines to show stack growth
    createManyGoroutines(100)

    // Force GC and show final state
    runtime.GC()
    runtime.ReadMemStats(&m)
    fmt.Printf("Final state: Goroutines=%d, Heap=%dKB, Stack=%dKB\n",
        runtime.NumGoroutine(),
        m.HeapAlloc/1024,
        m.StackInuse/1024)
}
```

**Hands-on Exercise 3: GC Tuning and Memory Profiling for Automation Systems**:

```go
// Advanced GC tuning and memory profiling for automation systems
package main

import (
    "fmt"
    "runtime"
    "runtime/debug"
    "time"
)

// AutomationWorkload represents different types of automation tasks
type AutomationWorkload struct {
    name           string
    allocationType string
    dataSize       int
    frequency      time.Duration
}

// MemoryProfiler tracks memory usage patterns
type MemoryProfiler struct {
    samples []MemorySample
}

type MemorySample struct {
    timestamp   time.Time
    heapAlloc   uint64
    heapSys     uint64
    numGC       uint32
    gcPauseNs   uint64
    goroutines  int
}

func (mp *MemoryProfiler) TakeSample() {
    var m runtime.MemStats
    runtime.ReadMemStats(&m)

    sample := MemorySample{
        timestamp:   time.Now(),
        heapAlloc:   m.HeapAlloc,
        heapSys:     m.HeapSys,
        numGC:       m.NumGC,
        gcPauseNs:   m.PauseNs[(m.NumGC+255)%256],
        goroutines:  runtime.NumGoroutine(),
    }

    mp.samples = append(mp.samples, sample)
}

func (mp *MemoryProfiler) PrintReport() {
    if len(mp.samples) < 2 {
        fmt.Println("Not enough samples for report")
        return
    }

    first := mp.samples[0]
    last := mp.samples[len(mp.samples)-1]

    fmt.Println("=== Memory Profile Report ===")
    fmt.Printf("Duration: %v\n", last.timestamp.Sub(first.timestamp))
    fmt.Printf("Heap Growth: %d KB\n", (last.heapAlloc-first.heapAlloc)/1024)
    fmt.Printf("GC Cycles: %d\n", last.numGC-first.numGC)
    fmt.Printf("Final Heap: %d KB\n", last.heapAlloc/1024)
    fmt.Printf("Final Goroutines: %d\n", last.goroutines)

    // Calculate average GC pause
    var totalPause uint64
    var pauseCount int
    for i := 1; i < len(mp.samples); i++ {
        if mp.samples[i].gcPauseNs > 0 {
            totalPause += mp.samples[i].gcPauseNs
            pauseCount++
        }
    }

    if pauseCount > 0 {
        avgPause := totalPause / uint64(pauseCount)
        fmt.Printf("Average GC Pause: %d Âµs\n", avgPause/1000)
    }

    fmt.Println()
}

// Simulate different automation workload patterns
func (aw *AutomationWorkload) Run(duration time.Duration, profiler *MemoryProfiler) {
    fmt.Printf("Running workload: %s\n", aw.name)

    start := time.Now()
    ticker := time.NewTicker(aw.frequency)
    defer ticker.Stop()

    var allocations []interface{}

    for time.Since(start) < duration {
        select {
        case <-ticker.C:
            // Simulate different allocation patterns
            switch aw.allocationType {
            case "short-lived":
                // Create temporary data that will be GC'd quickly
                temp := make([]byte, aw.dataSize)
                _ = temp

            case "long-lived":
                // Create data that persists
                data := make([]byte, aw.dataSize)
                allocations = append(allocations, data)

            case "mixed":
                // Mix of short and long-lived allocations
                if len(allocations)%3 == 0 {
                    data := make([]byte, aw.dataSize)
                    allocations = append(allocations, data)
                } else {
                    temp := make([]byte, aw.dataSize/2)
                    _ = temp
                }
            }

            // Take memory sample
            profiler.TakeSample()

        default:
            time.Sleep(time.Millisecond)
        }
    }

    // Keep allocations alive
    fmt.Printf("Workload %s completed with %d persistent allocations\n",
        aw.name, len(allocations))
}

// Test different GC settings
func testGCSettings() {
    fmt.Println("=== Testing Different GC Settings ===")

    gcSettings := []int{50, 100, 200}

    for _, gcPercent := range gcSettings {
        fmt.Printf("\nTesting GOGC=%d\n", gcPercent)

        // Set GC percentage
        oldGC := debug.SetGCPercent(gcPercent)

        profiler := &MemoryProfiler{}

        // Run a memory-intensive workload
        workload := &AutomationWorkload{
            name:           fmt.Sprintf("GC-Test-%d", gcPercent),
            allocationType: "mixed",
            dataSize:       1024,
            frequency:      time.Millisecond * 10,
        }

        workload.Run(2*time.Second, profiler)
        profiler.PrintReport()

        // Restore original GC setting
        debug.SetGCPercent(oldGC)

        // Force cleanup
        runtime.GC()
        time.Sleep(100 * time.Millisecond)
    }
}

// Demonstrate memory-efficient automation patterns
func demonstrateMemoryEfficientPatterns() {
    fmt.Println("=== Memory-Efficient Automation Patterns ===")

    // Pattern 1: Object pooling
    fmt.Println("1. Object Pooling Pattern:")

    type TaskPool struct {
        pool chan []byte
    }

    taskPool := &TaskPool{
        pool: make(chan []byte, 10),
    }

    // Pre-allocate buffers
    for i := 0; i < 10; i++ {
        taskPool.pool <- make([]byte, 1024)
    }

    // Use pooled objects
    for i := 0; i < 5; i++ {
        buffer := <-taskPool.pool
        // Use buffer for work
        _ = buffer
        // Return to pool
        taskPool.pool <- buffer
    }

    fmt.Printf("Object pool reduces allocations\n")

    // Pattern 2: Batch processing
    fmt.Println("\n2. Batch Processing Pattern:")

    batchSize := 100
    items := make([]string, 0, batchSize)

    for i := 0; i < 250; i++ {
        items = append(items, fmt.Sprintf("item-%d", i))

        if len(items) >= batchSize {
            // Process batch
            processBatch(items)
            // Reuse slice
            items = items[:0]
        }
    }

    // Process remaining items
    if len(items) > 0 {
        processBatch(items)
    }

    fmt.Printf("Batch processing reduces allocation frequency\n")
}

func processBatch(items []string) {
    // Simulate batch processing
    _ = items
}

func main() {
    fmt.Println("=== GC Tuning and Memory Profiling ===")

    // Show initial memory state
    var m runtime.MemStats
    runtime.ReadMemStats(&m)
    fmt.Printf("Initial: Heap=%dKB, GC=%d\n", m.HeapAlloc/1024, m.NumGC)

    // Test different GC settings
    testGCSettings()

    // Demonstrate memory-efficient patterns
    demonstrateMemoryEfficientPatterns()

    // Final memory state
    runtime.GC()
    runtime.ReadMemStats(&m)
    fmt.Printf("Final: Heap=%dKB, GC=%d\n", m.HeapAlloc/1024, m.NumGC)
}
```

**Prerequisites**: Module 5
