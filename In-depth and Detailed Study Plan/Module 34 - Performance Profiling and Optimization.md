# Module 34: Performance Profiling and Optimization

**Duration**: 1 hour (45 min theory + 15 min hands-on)

**Learning Objectives**:

- Master Go's profiling tools and techniques for performance analysis
- Understand CPU profiling, memory profiling, and execution tracing
- Learn to interpret profiling data and identify performance bottlenecks
- Apply profiling guidelines and best practices for accurate measurements
- Implement performance optimization strategies based on profiling results
- Design performance-aware automation systems with monitoring capabilities

**Videos Covered**:

- 14.1 Topics (0:00:55)
- 14.2 Profiling Guidelines (0:10:48)
- 14.3 Stack Traces (0:09:00)

**Key Concepts**:

- Profiling guidelines: idle machine, consistent environment, representative workloads
- CPU profiling: identifying hot paths and expensive function calls
- Memory profiling: heap analysis, allocation patterns, garbage collection impact
- Execution tracing: goroutine scheduling, blocking operations, system calls
- Stack trace analysis: understanding call chains and performance bottlenecks
- Benchmark validation: ensuring benchmarks measure what they claim to measure
- Performance optimization workflow: measure, analyze, optimize, validate
- Production profiling considerations and safety measures

**Hands-on Exercise 1: CPU Profiling and Analysis**:

Implementing CPU-intensive operations and analyzing performance with profiling tools:

```go
// Performance-critical automation functions for profiling analysis
package automation

import (
    "fmt"
    "math/rand"
    "sort"
    "time"
)

// DataProcessor handles CPU-intensive data processing operations
type DataProcessor struct {
    name            string
    processingDelay time.Duration
    bufferSize      int
}

func NewDataProcessor(name string, delay time.Duration, bufferSize int) *DataProcessor {
    return &DataProcessor{
        name:            name,
        processingDelay: delay,
        bufferSize:      bufferSize,
    }
}

// ProcessLargeDataset performs CPU-intensive processing on large datasets
func (dp *DataProcessor) ProcessLargeDataset(data [][]byte) ([]*AutomationTask, error) {
    tasks := make([]*AutomationTask, len(data))

    for i, item := range data {
        // Simulate CPU-intensive processing
        processedData := dp.complexDataTransformation(item)
        
        task := &AutomationTask{
            ID:        fmt.Sprintf("dataset-task-%d", i),
            Type:      "data",
            Data:      processedData,
            Priority:  dp.calculatePriority(item),
            CreatedAt: time.Now(),
            Status:    StatusPending,
        }

        tasks[i] = task
    }

    return tasks, nil
}

// complexDataTransformation performs multiple CPU-intensive operations
func (dp *DataProcessor) complexDataTransformation(data []byte) []byte {
    result := make([]byte, len(data))
    copy(result, data)

    // Multiple transformation passes
    for pass := 0; pass < 5; pass++ {
        // Hash-based transformation
        for i := 0; i < len(result); i++ {
            hash := uint32(result[i])
            hash = hash*31 + uint32(i)
            hash = hash*31 + uint32(pass)
            result[i] = byte(hash % 256)
        }

        // Sorting operation every other pass
        if pass%2 == 0 {
            dp.bubbleSort(result)
        }

        // XOR transformation
        for i := 0; i < len(result); i++ {
            result[i] = result[i] ^ byte(pass*7+13)
        }
    }

    return result
}

// bubbleSort implements an intentionally inefficient sorting algorithm for CPU profiling
func (dp *DataProcessor) bubbleSort(data []byte) {
    n := len(data)
    for i := 0; i < n-1; i++ {
        for j := 0; j < n-i-1; j++ {
            if data[j] > data[j+1] {
                data[j], data[j+1] = data[j+1], data[j]
            }
        }
    }
}

// calculatePriority performs CPU-intensive priority calculation
func (dp *DataProcessor) calculatePriority(data []byte) int {
    if len(data) == 0 {
        return 0
    }

    // Complex priority calculation involving multiple passes
    priority := 0
    
    // Pass 1: Sum of bytes
    for _, b := range data {
        priority += int(b)
    }
    
    // Pass 2: Weighted sum based on position
    for i, b := range data {
        priority += int(b) * (i + 1)
    }
    
    // Pass 3: Hash-based adjustment
    hash := uint32(priority)
    for i := 0; i < len(data); i++ {
        hash = hash*31 + uint32(data[i])
    }
    
    priority = int(hash % 100)
    
    // Pass 4: Fibonacci-based adjustment (intentionally inefficient)
    fibValue := dp.fibonacci(priority % 30)
    priority = (priority + fibValue) % 100
    
    return priority
}

// fibonacci implements an inefficient recursive fibonacci for CPU profiling
func (dp *DataProcessor) fibonacci(n int) int {
    if n <= 1 {
        return n
    }
    return dp.fibonacci(n-1) + dp.fibonacci(n-2)
}

// CreateLargeTaskBatch creates memory-intensive task batches for memory profiling
func (dp *DataProcessor) CreateLargeTaskBatch(count int, dataSize int) []*AutomationTask {
    tasks := make([]*AutomationTask, count)

    for i := 0; i < count; i++ {
        // Create large data blocks
        data := make([]byte, dataSize)
        for j := range data {
            data[j] = byte(rand.Intn(256))
        }

        // Create metadata maps
        metadata := make(map[string]string)
        for j := 0; j < 10; j++ {
            key := fmt.Sprintf("key_%d", j)
            value := fmt.Sprintf("value_%d_%d", i, j)
            metadata[key] = value
        }

        tasks[i] = &AutomationTask{
            ID:        fmt.Sprintf("large-task-%d", i),
            Type:      "data",
            Data:      data,
            Priority:  i % 10,
            CreatedAt: time.Now(),
            Status:    StatusPending,
            Metadata:  metadata,
        }
    }

    return tasks
}

// ProcessConcurrently demonstrates concurrent processing for execution tracing
func (dp *DataProcessor) ProcessConcurrently(tasks []*AutomationTask, workers int) error {
    taskChan := make(chan *AutomationTask, len(tasks))
    errorChan := make(chan error, workers)

    // Send tasks to channel
    for _, task := range tasks {
        taskChan <- task
    }
    close(taskChan)

    // Start workers
    for i := 0; i < workers; i++ {
        go func(workerID int) {
            for task := range taskChan {
                // Simulate processing with CPU work
                processedData := dp.complexDataTransformation(task.Data)
                task.Data = processedData
                task.Status = StatusCompleted
                now := time.Now()
                task.ProcessedAt = &now
            }
            errorChan <- nil
        }(i)
    }

    // Collect results
    for i := 0; i < workers; i++ {
        if err := <-errorChan; err != nil {
            return err
        }
    }

    return nil
}

// MemoryIntensiveOperation demonstrates memory allocation patterns for memory profiling
func (dp *DataProcessor) MemoryIntensiveOperation(pattern string, size int) [][]byte {
    switch pattern {
    case "sequential":
        return dp.allocateSequential(size)
    case "random":
        return dp.allocateRandom(size)
    case "fragmented":
        return dp.allocateFragmented(size)
    default:
        return dp.allocateSequential(size)
    }
}

func (dp *DataProcessor) allocateSequential(size int) [][]byte {
    slices := make([][]byte, size)
    for i := 0; i < size; i++ {
        slices[i] = make([]byte, i+1)
        // Fill with data to prevent optimization
        for j := range slices[i] {
            slices[i][j] = byte(i + j)
        }
    }
    return slices
}

func (dp *DataProcessor) allocateRandom(size int) [][]byte {
    slices := make([][]byte, size)
    for i := 0; i < size; i++ {
        randomSize := rand.Intn(1000) + 1
        slices[i] = make([]byte, randomSize)
        // Fill with random data
        for j := range slices[i] {
            slices[i][j] = byte(rand.Intn(256))
        }
    }
    return slices
}

func (dp *DataProcessor) allocateFragmented(size int) [][]byte {
    slices := make([][]byte, size)
    for i := 0; i < size; i++ {
        if i%2 == 0 {
            slices[i] = make([]byte, 1024)
        } else {
            slices[i] = make([]byte, 16)
        }
        // Fill with pattern data
        for j := range slices[i] {
            slices[i][j] = byte((i + j) % 256)
        }
    }
    return slices
}
```

**Profiling Test File (profiling_test.go):**

```go
package automation_test

import (
    "automation"
    "bytes"
    "fmt"
    "runtime"
    "testing"
    "time"
)

// PART 1: CPU Profiling Benchmarks
// These benchmarks are designed to generate meaningful CPU profiles

func BenchmarkDataProcessor_ComplexTransformation(b *testing.B) {
    processor := automation.NewDataProcessor("CPUBenchProcessor", 0, 1000)

    // Create test data
    data := make([]byte, 1000)
    for i := range data {
        data[i] = byte(i % 256)
    }

    b.ResetTimer()
    for i := 0; i < b.N; i++ {
        processor.ProcessLargeDataset([][]byte{data})
    }
}

func BenchmarkDataProcessor_PriorityCalculation(b *testing.B) {
    processor := automation.NewDataProcessor("PriorityBenchProcessor", 0, 1000)

    dataSizes := []int{100, 500, 1000, 2000}

    for _, size := range dataSizes {
        b.Run(fmt.Sprintf("size-%d", size), func(b *testing.B) {
            data := make([]byte, size)
            for i := range data {
                data[i] = byte(i % 256)
            }

            b.ResetTimer()
            for i := 0; i < b.N; i++ {
                processor.ProcessLargeDataset([][]byte{data})
            }
        })
    }
}

func BenchmarkDataProcessor_ConcurrentProcessing(b *testing.B) {
    processor := automation.NewDataProcessor("ConcurrentBenchProcessor", 0, 1000)

    workerCounts := []int{1, 2, 4, 8}

    for _, workers := range workerCounts {
        b.Run(fmt.Sprintf("workers-%d", workers), func(b *testing.B) {
            // Create tasks for processing
            tasks := processor.CreateLargeTaskBatch(100, 256)

            b.ResetTimer()
            for i := 0; i < b.N; i++ {
                // Reset task status for each iteration
                for _, task := range tasks {
                    task.Status = automation.StatusPending
                }

                processor.ProcessConcurrently(tasks, workers)
            }
        })
    }
}

// PART 2: Memory Profiling Benchmarks
// These benchmarks focus on memory allocation patterns

func BenchmarkDataProcessor_MemoryAllocation(b *testing.B) {
    processor := automation.NewDataProcessor("MemoryBenchProcessor", 0, 1000)

    patterns := []string{"sequential", "random", "fragmented"}
    sizes := []int{100, 500, 1000}

    for _, pattern := range patterns {
        for _, size := range sizes {
            name := fmt.Sprintf("%s-size-%d", pattern, size)
            b.Run(name, func(b *testing.B) {
                b.ReportAllocs() // Report memory allocations

                b.ResetTimer()
                for i := 0; i < b.N; i++ {
                    processor.MemoryIntensiveOperation(pattern, size)
                }
            })
        }
    }
}

func BenchmarkDataProcessor_TaskCreation(b *testing.B) {
    processor := automation.NewDataProcessor("TaskCreationBenchProcessor", 0, 1000)

    b.Run("SmallTasks", func(b *testing.B) {
        b.ReportAllocs()

        b.ResetTimer()
        for i := 0; i < b.N; i++ {
            processor.CreateLargeTaskBatch(10, 64)
        }
    })

    b.Run("MediumTasks", func(b *testing.B) {
        b.ReportAllocs()

        b.ResetTimer()
        for i := 0; i < b.N; i++ {
            processor.CreateLargeTaskBatch(50, 256)
        }
    })

    b.Run("LargeTasks", func(b *testing.B) {
        b.ReportAllocs()

        b.ResetTimer()
        for i := 0; i < b.N; i++ {
            processor.CreateLargeTaskBatch(100, 1024)
        }
    })
}

// PART 3: Execution Tracing Benchmarks
// These benchmarks demonstrate goroutine scheduling and blocking operations

func BenchmarkDataProcessor_GoroutineScheduling(b *testing.B) {
    processor := automation.NewDataProcessor("SchedulingBenchProcessor", 0, 1000)

    b.Run("HighContention", func(b *testing.B) {
        tasks := processor.CreateLargeTaskBatch(1000, 128)

        b.ResetTimer()
        for i := 0; i < b.N; i++ {
            // Reset task status
            for _, task := range tasks {
                task.Status = automation.StatusPending
            }

            // Use many workers to create scheduling contention
            processor.ProcessConcurrently(tasks, runtime.NumCPU()*4)
        }
    })

    b.Run("LowContention", func(b *testing.B) {
        tasks := processor.CreateLargeTaskBatch(100, 128)

        b.ResetTimer()
        for i := 0; i < b.N; i++ {
            // Reset task status
            for _, task := range tasks {
                task.Status = automation.StatusPending
            }

            // Use fewer workers to reduce contention
            processor.ProcessConcurrently(tasks, 2)
        }
    })
}

// PART 4: Benchmark Validation
// These benchmarks demonstrate validation techniques

func BenchmarkDataProcessor_ValidationExample(b *testing.B) {
    processor := automation.NewDataProcessor("ValidationBenchProcessor", 0, 1000)

    b.Run("ValidatedBenchmark", func(b *testing.B) {
        // Create consistent test data
        data := bytes.Repeat([]byte("test"), 250) // 1000 bytes

        // Validate that our benchmark actually does work
        result1 := processor.ProcessLargeDataset([][]byte{data})
        result2 := processor.ProcessLargeDataset([][]byte{data})

        // Ensure results are different (processing actually occurred)
        if len(result1) == 0 || len(result2) == 0 {
            b.Fatal("Benchmark validation failed: no results produced")
        }

        if bytes.Equal(result1[0].Data, data) {
            b.Fatal("Benchmark validation failed: data was not processed")
        }

        b.ResetTimer()
        for i := 0; i < b.N; i++ {
            processor.ProcessLargeDataset([][]byte{data})
        }
    })

    b.Run("UnvalidatedBenchmark", func(b *testing.B) {
        // This benchmark might be optimized away by the compiler
        data := make([]byte, 1000)

        b.ResetTimer()
        for i := 0; i < b.N; i++ {
            // Potentially optimized away if result is not used
            processor.ProcessLargeDataset([][]byte{data})
        }
    })
}

// Helper function to demonstrate profiling setup
func init() {
    // Set GOMAXPROCS for consistent benchmarking
    runtime.GOMAXPROCS(runtime.NumCPU())
}
```

**Hands-on Exercise 2: Memory Profiling and Analysis**:

Implementing memory-intensive operations and analyzing heap usage:

```go
// Memory profiling demonstration with heap analysis
package automation_test

import (
    "automation"
    "runtime"
    "testing"
    "time"
)

// TestMemoryUsageAnalysis demonstrates memory usage patterns for profiling
func TestMemoryUsageAnalysis(t *testing.T) {
    processor := automation.NewDataProcessor("MemoryAnalysisProcessor", 0, 10000)

    t.Run("MemoryGrowthPattern", func(t *testing.T) {
        var memStats runtime.MemStats

        // Baseline memory usage
        runtime.GC()
        runtime.ReadMemStats(&memStats)
        baselineAlloc := memStats.Alloc

        t.Logf("Baseline memory allocation: %d bytes", baselineAlloc)

        // Allocate memory in stages
        stages := []struct {
            name      string
            taskCount int
            dataSize  int
        }{
            {"Small", 10, 64},
            {"Medium", 50, 256},
            {"Large", 100, 1024},
            {"XLarge", 200, 2048},
        }

        for _, stage := range stages {
            // Create tasks
            tasks := processor.CreateLargeTaskBatch(stage.taskCount, stage.dataSize)

            // Measure memory after allocation
            runtime.ReadMemStats(&memStats)
            currentAlloc := memStats.Alloc

            t.Logf("%s stage - Tasks: %d, Data size: %d, Memory: %d bytes (+%d)",
                stage.name, stage.taskCount, stage.dataSize, currentAlloc, currentAlloc-baselineAlloc)

            // Keep reference to prevent GC
            _ = tasks
        }

        // Force GC and measure
        runtime.GC()
        runtime.ReadMemStats(&memStats)
        afterGC := memStats.Alloc

        t.Logf("After GC: %d bytes", afterGC)
        t.Logf("Total allocations: %d", memStats.TotalAlloc)
        t.Logf("GC cycles: %d", memStats.NumGC)
    })

    t.Run("AllocationPatterns", func(t *testing.T) {
        patterns := []string{"sequential", "random", "fragmented"}

        for _, pattern := range patterns {
            var memStats runtime.MemStats

            runtime.GC()
            runtime.ReadMemStats(&memStats)
            before := memStats.TotalAlloc

            // Allocate using different patterns
            result := processor.MemoryIntensiveOperation(pattern, 1000)

            runtime.ReadMemStats(&memStats)
            after := memStats.TotalAlloc

            t.Logf("Pattern %s: allocated %d bytes, slices: %d",
                pattern, after-before, len(result))
        }
    })
}

// BenchmarkMemoryLeakDetection demonstrates memory leak detection techniques
func BenchmarkMemoryLeakDetection(b *testing.B) {
    processor := automation.NewDataProcessor("LeakDetectionProcessor", 0, 1000)

    b.Run("PotentialLeak", func(b *testing.B) {
        var memStats runtime.MemStats

        // Measure initial memory
        runtime.GC()
        runtime.ReadMemStats(&memStats)
        initialMem := memStats.Alloc

        b.ResetTimer()
        for i := 0; i < b.N; i++ {
            // Create tasks but don't release references properly
            tasks := processor.CreateLargeTaskBatch(100, 512)

            // Simulate processing
            for _, task := range tasks {
                task.Status = automation.StatusCompleted
                now := time.Now()
                task.ProcessedAt = &now
            }

            // Intentionally keep reference (potential leak)
            if i%100 == 0 {
                _ = tasks // This could cause memory to accumulate
            }
        }
        b.StopTimer()

        // Check for memory growth
        runtime.GC()
        runtime.ReadMemStats(&memStats)
        finalMem := memStats.Alloc

        if finalMem > initialMem*2 {
            b.Logf("Potential memory leak detected: %d -> %d bytes", initialMem, finalMem)
        }
    })
}
```

**Hands-on Exercise 3: Execution Tracing and Stack Analysis**:

Implementing execution tracing to analyze goroutine behavior and blocking operations:

```go
// Execution tracing demonstration for goroutine analysis
package automation_test

import (
    "automation"
    "context"
    "runtime"
    "sync"
    "testing"
    "time"
)

// TestExecutionTracing demonstrates execution tracing scenarios
func TestExecutionTracing(t *testing.T) {
    processor := automation.NewDataProcessor("TracingProcessor", 0, 1000)

    t.Run("GoroutineLifecycle", func(t *testing.T) {
        tasks := processor.CreateLargeTaskBatch(50, 256)

        var wg sync.WaitGroup
        workerCount := 4

        // Create channel for task distribution
        taskChan := make(chan *automation.AutomationTask, len(tasks))

        // Start workers
        for i := 0; i < workerCount; i++ {
            wg.Add(1)
            go func(workerID int) {
                defer wg.Done()

                for task := range taskChan {
                    // Simulate processing time
                    time.Sleep(1 * time.Millisecond)

                    // Process task
                    task.Status = automation.StatusCompleted
                    now := time.Now()
                    task.ProcessedAt = &now

                    t.Logf("Worker %d processed task %s", workerID, task.ID)
                }
            }(i)
        }

        // Send tasks to workers
        for _, task := range tasks {
            taskChan <- task
        }
        close(taskChan)

        // Wait for completion
        wg.Wait()

        t.Logf("Processed %d tasks with %d workers", len(tasks), workerCount)
    })

    t.Run("BlockingOperations", func(t *testing.T) {
        tasks := processor.CreateLargeTaskBatch(20, 128)

        // Simulate blocking operations with channels
        resultChan := make(chan *automation.AutomationTask, len(tasks))

        // Process tasks with intentional blocking
        for _, task := range tasks {
            go func(t *automation.AutomationTask) {
                // Simulate network delay (blocking operation)
                time.Sleep(10 * time.Millisecond)

                t.Status = automation.StatusCompleted
                now := time.Now()
                t.ProcessedAt = &now

                resultChan <- t
            }(task)
        }

        // Collect results
        processed := 0
        timeout := time.After(5 * time.Second)

        for processed < len(tasks) {
            select {
            case task := <-resultChan:
                processed++
                t.Logf("Received processed task: %s", task.ID)
            case <-timeout:
                t.Fatalf("Timeout waiting for task processing")
            }
        }

        t.Logf("Successfully processed %d tasks", processed)
    })

    t.Run("ContextCancellation", func(t *testing.T) {
        tasks := processor.CreateLargeTaskBatch(100, 256)

        ctx, cancel := context.WithTimeout(context.Background(), 50*time.Millisecond)
        defer cancel()

        var wg sync.WaitGroup
        processed := 0
        var mu sync.Mutex

        // Start workers that respect context cancellation
        for i := 0; i < 4; i++ {
            wg.Add(1)
            go func(workerID int) {
                defer wg.Done()

                for _, task := range tasks {
                    select {
                    case <-ctx.Done():
                        t.Logf("Worker %d cancelled due to context", workerID)
                        return
                    default:
                        // Simulate processing
                        time.Sleep(2 * time.Millisecond)

                        mu.Lock()
                        processed++
                        mu.Unlock()

                        task.Status = automation.StatusCompleted
                    }
                }
            }(i)
        }

        wg.Wait()

        t.Logf("Processed %d out of %d tasks before cancellation", processed, len(tasks))
    })
}

// BenchmarkExecutionTracing provides benchmarks suitable for execution tracing
func BenchmarkExecutionTracing(b *testing.B) {
    processor := automation.NewDataProcessor("TracingBenchProcessor", 0, 1000)

    b.Run("GoroutineContention", func(b *testing.B) {
        tasks := processor.CreateLargeTaskBatch(1000, 128)

        b.ResetTimer()
        for i := 0; i < b.N; i++ {
            // Reset task status
            for _, task := range tasks {
                task.Status = automation.StatusPending
            }

            // Use many goroutines to create contention
            processor.ProcessConcurrently(tasks, runtime.NumCPU()*8)
        }
    })

    b.Run("ChannelOperations", func(b *testing.B) {
        b.ResetTimer()
        for i := 0; i < b.N; i++ {
            ch := make(chan int, 100)

            // Producer
            go func() {
                for j := 0; j < 1000; j++ {
                    ch <- j
                }
                close(ch)
            }()

            // Consumer
            count := 0
            for range ch {
                count++
            }
        }
    })
}
```

**Profiling Commands and Analysis Guide:**

```bash
# CPU Profiling
# Generate CPU profile during benchmarking
go test -bench=BenchmarkDataProcessor_ComplexTransformation -cpuprofile=cpu.prof

# Analyze CPU profile interactively
go tool pprof cpu.prof

# Generate CPU profile web interface
go tool pprof -http=:8080 cpu.prof

# Memory Profiling
# Generate memory profile during benchmarking
go test -bench=BenchmarkDataProcessor_MemoryAllocation -memprofile=mem.prof

# Analyze memory profile
go tool pprof mem.prof

# Generate memory profile web interface
go tool pprof -http=:8080 mem.prof

# Execution Tracing
# Generate execution trace
go test -bench=BenchmarkExecutionTracing -trace=trace.out

# Analyze execution trace
go tool trace trace.out

# Benchmark Validation
# Run benchmarks with validation
go test -bench=BenchmarkDataProcessor_ValidationExample -benchmem

# Run benchmarks multiple times for statistical accuracy
go test -bench=. -count=10

# Run benchmarks for specific duration
go test -bench=. -benchtime=30s

# Combined Profiling
# Generate both CPU and memory profiles
go test -bench=BenchmarkDataProcessor -cpuprofile=cpu.prof -memprofile=mem.prof -benchmem

# Race Detection with Profiling
go test -race -bench=BenchmarkDataProcessor_ConcurrentProcessing

# Block Profiling (for goroutine blocking analysis)
go test -bench=BenchmarkExecutionTracing -blockprofile=block.prof

# Mutex Profiling (for lock contention analysis)
go test -bench=BenchmarkDataProcessor_ConcurrentProcessing -mutexprofile=mutex.prof
```

**Profiling Analysis Workflow:**

1. **Establish Baseline**: Run benchmarks to establish performance baseline
2. **Generate Profiles**: Use appropriate profiling flags for your analysis needs
3. **Analyze Hotspots**: Use `go tool pprof` to identify performance bottlenecks
4. **Optimize Code**: Make targeted improvements based on profiling data
5. **Validate Changes**: Re-run benchmarks to confirm improvements
6. **Monitor Production**: Use production profiling for real-world validation

**Common pprof Commands:**

```bash
# In pprof interactive mode:
(pprof) top           # Show top functions by CPU/memory usage
(pprof) top10         # Show top 10 functions
(pprof) list main     # Show source code for main function
(pprof) web           # Generate SVG call graph
(pprof) peek main     # Show callers and callees of main
(pprof) disasm main   # Show assembly code for main function
(pprof) help          # Show all available commands
```

**Running the Exercises:**

```bash
# Exercise 1: CPU Profiling and Analysis
go test -bench=BenchmarkDataProcessor_ComplexTransformation -cpuprofile=cpu.prof
go tool pprof cpu.prof

# Exercise 2: Memory Profiling and Analysis
go test -bench=BenchmarkDataProcessor_MemoryAllocation -memprofile=mem.prof -benchmem
go tool pprof mem.prof

# Exercise 3: Execution Tracing and Stack Analysis
go test -bench=BenchmarkExecutionTracing -trace=trace.out
go tool trace trace.out

# Combined Analysis
go test -bench=. -cpuprofile=cpu.prof -memprofile=mem.prof -trace=trace.out -benchmem

# Memory Leak Detection
go test -run TestMemoryUsageAnalysis -v

# Goroutine Analysis
go test -run TestExecutionTracing -v
```

**Prerequisites**: Module 33
