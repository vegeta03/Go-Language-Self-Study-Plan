# Module 35: Advanced Performance Profiling and Optimization

**Duration**: 1 hour (45 min theory + 15 min hands-on)

**Learning Objectives**:

- Master advanced profiling techniques for micro and macro level optimization
- Understand GODEBUG tracing for runtime behavior analysis
- Learn memory profiling strategies for heap analysis and garbage collection optimization
- Apply CPU profiling techniques to identify performance bottlenecks
- Implement execution tracing for comprehensive performance analysis
- Design performance-aware automation systems with continuous monitoring

**Videos Covered**:

- 14.4 Micro Level Optimization (0:31:17)
- 14.5 Macro Level Opt. - GODEBUG Tracing (0:12:49)
- 14.6 Macro Level Opt. - Memory Profiling (0:16:07)
- 14.7 Macro Level Opt. - Tooling Changes (0:06:03)
- 14.8 Macro Level Opt. - CPU Profiling (0:05:53)
- 14.9 Execution Tracing (0:34:24)
- Ultimate Go Programming Summary (0:01:11)

**Key Concepts**:

- Micro-level optimization: function-level performance tuning and algorithmic improvements
- GODEBUG environment variables: runtime tracing and debugging capabilities
- Memory profiling: heap analysis, allocation patterns, and garbage collection impact
- CPU profiling: identifying hot paths, expensive function calls, and optimization opportunities
- Execution tracing: comprehensive analysis of goroutine scheduling, blocking operations, and system calls
- Performance optimization workflow: measure, analyze, optimize, validate cycle
- Production profiling considerations: safety measures and minimal performance impact
- Tooling ecosystem: pprof, go tool trace, and third-party profiling tools

**Hands-on Exercise 1: Micro-Level Performance Optimization**:

Implementing function-level optimizations and algorithmic improvements for automation systems:

```go
// Advanced performance optimization for automation data processing
package optimization

import (
    "context"
    "fmt"
    "runtime"
    "sync"
    "time"
    "unsafe"
)

// DataProcessor represents a high-performance data processing system
type DataProcessor struct {
    workers    int
    bufferSize int
    pool       sync.Pool
    metrics    *ProcessingMetrics
}

// ProcessingMetrics tracks performance metrics
type ProcessingMetrics struct {
    mu              sync.RWMutex
    processedItems  uint64
    processingTime  time.Duration
    allocations     uint64
    gcPauses        []time.Duration
}

// NewDataProcessor creates an optimized data processor
func NewDataProcessor(workers, bufferSize int) *DataProcessor {
    return &DataProcessor{
        workers:    workers,
        bufferSize: bufferSize,
        pool: sync.Pool{
            New: func() interface{} {
                return make([]byte, bufferSize)
            },
        },
        metrics: &ProcessingMetrics{},
    }
}

// ProcessBatch demonstrates micro-level optimizations
func (dp *DataProcessor) ProcessBatch(ctx context.Context, data [][]byte) error {
    start := time.Now()
    var memStats runtime.MemStats
    runtime.ReadMemStats(&memStats)
    initialAllocs := memStats.Mallocs

    // Use worker pool pattern for concurrent processing
    jobs := make(chan []byte, len(data))
    results := make(chan ProcessingResult, len(data))
    
    // Start workers
    var wg sync.WaitGroup
    for i := 0; i < dp.workers; i++ {
        wg.Add(1)
        go dp.worker(ctx, &wg, jobs, results)
    }
    
    // Send jobs
    go func() {
        defer close(jobs)
        for _, item := range data {
            select {
            case jobs <- item:
            case <-ctx.Done():
                return
            }
        }
    }()
    
    // Collect results
    go func() {
        wg.Wait()
        close(results)
    }()
    
    processedCount := 0
    for result := range results {
        if result.Error != nil {
            return fmt.Errorf("processing failed: %w", result.Error)
        }
        processedCount++
    }
    
    // Update metrics
    runtime.ReadMemStats(&memStats)
    dp.updateMetrics(uint64(processedCount), time.Since(start), 
                    memStats.Mallocs-initialAllocs, memStats.PauseNs[:memStats.NumGC])
    
    return nil
}

// ProcessingResult represents the result of data processing
type ProcessingResult struct {
    Data  []byte
    Error error
}

// worker implements optimized worker goroutine
func (dp *DataProcessor) worker(ctx context.Context, wg *sync.WaitGroup, 
                               jobs <-chan []byte, results chan<- ProcessingResult) {
    defer wg.Done()
    
    // Reuse buffer from pool to reduce allocations
    buffer := dp.pool.Get().([]byte)
    defer dp.pool.Put(buffer)
    
    for {
        select {
        case job, ok := <-jobs:
            if !ok {
                return
            }
            
            // Perform optimized processing
            result := dp.processItem(job, buffer)
            
            select {
            case results <- result:
            case <-ctx.Done():
                return
            }
            
        case <-ctx.Done():
            return
        }
    }
}

// processItem demonstrates memory-efficient processing
func (dp *DataProcessor) processItem(data []byte, buffer []byte) ProcessingResult {
    // Zero-allocation string conversion for comparison
    dataStr := *(*string)(unsafe.Pointer(&data))
    
    // Efficient processing logic here
    if len(data) == 0 {
        return ProcessingResult{Error: fmt.Errorf("empty data")}
    }
    
    // Use pre-allocated buffer to avoid allocations
    if len(buffer) < len(data)*2 {
        buffer = make([]byte, len(data)*2)
    }
    
    // Simulate processing with memory-efficient operations
    copy(buffer[:len(data)], data)
    for i := len(data); i < len(data)*2; i++ {
        buffer[i] = buffer[i-len(data)] ^ 0xFF
    }
    
    // Return processed data
    result := make([]byte, len(data)*2)
    copy(result, buffer[:len(data)*2])
    
    return ProcessingResult{Data: result}
}

// updateMetrics safely updates processing metrics
func (dp *DataProcessor) updateMetrics(items uint64, duration time.Duration, 
                                      allocs uint64, gcPauses []uint64) {
    dp.metrics.mu.Lock()
    defer dp.metrics.mu.Unlock()
    
    dp.metrics.processedItems += items
    dp.metrics.processingTime += duration
    dp.metrics.allocations += allocs
    
    // Convert and store GC pause times
    for _, pause := range gcPauses {
        if pause > 0 {
            dp.metrics.gcPauses = append(dp.metrics.gcPauses, time.Duration(pause))
        }
    }
}

// GetMetrics returns current processing metrics
func (dp *DataProcessor) GetMetrics() ProcessingMetrics {
    dp.metrics.mu.RLock()
    defer dp.metrics.mu.RUnlock()
    
    // Return copy to avoid race conditions
    metrics := *dp.metrics
    metrics.gcPauses = make([]time.Duration, len(dp.metrics.gcPauses))
    copy(metrics.gcPauses, dp.metrics.gcPauses)
    
    return metrics
}
```

**Hands-on Exercise 2: GODEBUG Tracing and Memory Profiling**:

Using GODEBUG environment variables and memory profiling for runtime analysis:

```go
// Advanced profiling and tracing for automation systems
package profiling

import (
    "context"
    "fmt"
    "log"
    "net/http"
    _ "net/http/pprof"
    "os"
    "runtime"
    "runtime/pprof"
    "runtime/trace"
    "sync"
    "time"
)

// ProfiledAutomationService demonstrates advanced profiling techniques
type ProfiledAutomationService struct {
    name           string
    enableProfiling bool
    traceFile      *os.File
    cpuProfile     *os.File
    memProfile     *os.File
    mu             sync.RWMutex
    metrics        map[string]interface{}
}

// NewProfiledService creates a service with profiling capabilities
func NewProfiledService(name string, enableProfiling bool) *ProfiledAutomationService {
    service := &ProfiledAutomationService{
        name:           name,
        enableProfiling: enableProfiling,
        metrics:        make(map[string]interface{}),
    }
    
    if enableProfiling {
        service.setupProfiling()
    }
    
    return service
}

// setupProfiling initializes profiling infrastructure
func (pas *ProfiledAutomationService) setupProfiling() {
    // Start pprof HTTP server for live profiling
    go func() {
        log.Printf("Starting pprof server on :6060")
        log.Println(http.ListenAndServe("localhost:6060", nil))
    }()
    
    // Setup trace file
    if traceFile, err := os.Create(fmt.Sprintf("%s_trace.out", pas.name)); err == nil {
        pas.traceFile = traceFile
        trace.Start(traceFile)
        log.Printf("Execution tracing started: %s", traceFile.Name())
    }
    
    // Setup CPU profile
    if cpuFile, err := os.Create(fmt.Sprintf("%s_cpu.prof", pas.name)); err == nil {
        pas.cpuProfile = cpuFile
        pprof.StartCPUProfile(cpuFile)
        log.Printf("CPU profiling started: %s", cpuFile.Name())
    }
}

// ProcessWithProfiling demonstrates profiled processing
func (pas *ProfiledAutomationService) ProcessWithProfiling(ctx context.Context, 
                                                          workload []string) error {
    // Memory profiling checkpoint
    var m1, m2 runtime.MemStats
    runtime.ReadMemStats(&m1)
    
    start := time.Now()
    
    // Simulate intensive processing
    results := make(chan string, len(workload))
    errors := make(chan error, len(workload))
    
    // Process with controlled concurrency
    semaphore := make(chan struct{}, runtime.NumCPU())
    var wg sync.WaitGroup
    
    for _, item := range workload {
        wg.Add(1)
        go func(data string) {
            defer wg.Done()
            
            semaphore <- struct{}{} // Acquire
            defer func() { <-semaphore }() // Release
            
            result, err := pas.processItem(data)
            if err != nil {
                errors <- err
                return
            }
            results <- result
        }(item)
    }
    
    // Wait for completion
    go func() {
        wg.Wait()
        close(results)
        close(errors)
    }()
    
    // Collect results and errors
    var processedResults []string
    for {
        select {
        case result, ok := <-results:
            if !ok {
                results = nil
            } else {
                processedResults = append(processedResults, result)
            }
        case err, ok := <-errors:
            if !ok {
                errors = nil
            } else {
                return fmt.Errorf("processing error: %w", err)
            }
        }
        
        if results == nil && errors == nil {
            break
        }
    }
    
    // Memory profiling analysis
    runtime.ReadMemStats(&m2)
    duration := time.Since(start)
    
    // Update metrics
    pas.updateMetrics(map[string]interface{}{
        "processed_items":    len(processedResults),
        "processing_time":    duration,
        "memory_allocated":   m2.TotalAlloc - m1.TotalAlloc,
        "heap_objects":       m2.HeapObjects,
        "gc_cycles":          m2.NumGC - m1.NumGC,
        "gc_pause_total":     time.Duration(m2.PauseTotalNs - m1.PauseTotalNs),
    })
    
    log.Printf("Processing completed: %d items in %v", len(processedResults), duration)
    log.Printf("Memory allocated: %d bytes, GC cycles: %d", 
               m2.TotalAlloc-m1.TotalAlloc, m2.NumGC-m1.NumGC)
    
    return nil
}

// processItem simulates CPU-intensive processing
func (pas *ProfiledAutomationService) processItem(data string) (string, error) {
    // Simulate CPU-intensive work
    hash := uint64(0)
    for i, char := range data {
        hash = hash*31 + uint64(char) + uint64(i)
    }
    
    // Simulate memory allocation patterns
    buffer := make([]byte, len(data)*2)
    copy(buffer[:len(data)], data)
    
    // More processing simulation
    for i := 0; i < 1000; i++ {
        hash = hash*31 + uint64(i)
    }
    
    return fmt.Sprintf("processed_%s_%d", data, hash), nil
}

// updateMetrics safely updates service metrics
func (pas *ProfiledAutomationService) updateMetrics(newMetrics map[string]interface{}) {
    pas.mu.Lock()
    defer pas.mu.Unlock()
    
    for key, value := range newMetrics {
        pas.metrics[key] = value
    }
}

// GetMetrics returns current service metrics
func (pas *ProfiledAutomationService) GetMetrics() map[string]interface{} {
    pas.mu.RLock()
    defer pas.mu.RUnlock()
    
    // Return copy
    result := make(map[string]interface{})
    for key, value := range pas.metrics {
        result[key] = value
    }
    return result
}

// Shutdown gracefully stops profiling
func (pas *ProfiledAutomationService) Shutdown() {
    if !pas.enableProfiling {
        return
    }
    
    // Stop CPU profiling
    if pas.cpuProfile != nil {
        pprof.StopCPUProfile()
        pas.cpuProfile.Close()
        log.Printf("CPU profiling stopped")
    }
    
    // Stop execution tracing
    if pas.traceFile != nil {
        trace.Stop()
        pas.traceFile.Close()
        log.Printf("Execution tracing stopped")
    }
    
    // Write memory profile
    if memFile, err := os.Create(fmt.Sprintf("%s_mem.prof", pas.name)); err == nil {
        runtime.GC() // Force GC before memory profile
        pprof.WriteHeapProfile(memFile)
        memFile.Close()
        log.Printf("Memory profile written: %s", memFile.Name())
    }
}
```

**Hands-on Exercise 3: Execution Tracing and Performance Analysis**:

Comprehensive execution tracing for automation system performance analysis:

```go
// Execution tracing and performance analysis utilities
package tracing

import (
    "context"
    "fmt"
    "log"
    "os"
    "runtime"
    "runtime/trace"
    "sync"
    "time"
)

// TracedAutomationWorkflow demonstrates execution tracing
type TracedAutomationWorkflow struct {
    name        string
    stages      []WorkflowStage
    traceFile   *os.File
    metrics     *WorkflowMetrics
}

// WorkflowStage represents a stage in the automation workflow
type WorkflowStage struct {
    Name        string
    Function    func(context.Context, interface{}) (interface{}, error)
    Parallel    bool
    MaxWorkers  int
}

// WorkflowMetrics tracks workflow execution metrics
type WorkflowMetrics struct {
    mu              sync.RWMutex
    stageMetrics    map[string]StageMetrics
    totalDuration   time.Duration
    goroutineCount  int
    blockingTime    time.Duration
}

// StageMetrics tracks individual stage performance
type StageMetrics struct {
    ExecutionTime   time.Duration
    GoroutineCount  int
    BlockingEvents  int
    MemoryUsage     uint64
}

// NewTracedWorkflow creates a workflow with execution tracing
func NewTracedWorkflow(name string) *TracedAutomationWorkflow {
    workflow := &TracedAutomationWorkflow{
        name: name,
        metrics: &WorkflowMetrics{
            stageMetrics: make(map[string]StageMetrics),
        },
    }
    
    // Start execution tracing
    if traceFile, err := os.Create(fmt.Sprintf("%s_execution.trace", name)); err == nil {
        workflow.traceFile = traceFile
        trace.Start(traceFile)
        log.Printf("Execution tracing started for workflow: %s", name)
    }
    
    return workflow
}

// AddStage adds a stage to the workflow
func (taw *TracedAutomationWorkflow) AddStage(stage WorkflowStage) {
    taw.stages = append(taw.stages, stage)
}

// Execute runs the workflow with comprehensive tracing
func (taw *TracedAutomationWorkflow) Execute(ctx context.Context, input interface{}) (interface{}, error) {
    defer trace.StartRegion(ctx, "WorkflowExecution").End()
    
    start := time.Now()
    var memStats runtime.MemStats
    runtime.ReadMemStats(&memStats)
    initialGoroutines := runtime.NumGoroutine()
    
    current := input
    
    for _, stage := range taw.stages {
        stageCtx := trace.WithRegion(ctx, stage.Name)
        
        stageStart := time.Now()
        stageGoroutines := runtime.NumGoroutine()
        
        var err error
        if stage.Parallel {
            current, err = taw.executeParallelStage(stageCtx, stage, current)
        } else {
            current, err = taw.executeSequentialStage(stageCtx, stage, current)
        }
        
        if err != nil {
            return nil, fmt.Errorf("stage %s failed: %w", stage.Name, err)
        }
        
        // Record stage metrics
        stageDuration := time.Since(stageStart)
        stageGoroutineCount := runtime.NumGoroutine() - stageGoroutines
        
        taw.recordStageMetrics(stage.Name, StageMetrics{
            ExecutionTime:  stageDuration,
            GoroutineCount: stageGoroutineCount,
        })
        
        log.Printf("Stage %s completed in %v with %d goroutines", 
                   stage.Name, stageDuration, stageGoroutineCount)
    }
    
    // Record overall metrics
    totalDuration := time.Since(start)
    finalGoroutines := runtime.NumGoroutine()
    
    taw.metrics.mu.Lock()
    taw.metrics.totalDuration = totalDuration
    taw.metrics.goroutineCount = finalGoroutines - initialGoroutines
    taw.metrics.mu.Unlock()
    
    log.Printf("Workflow %s completed in %v", taw.name, totalDuration)
    
    return current, nil
}

// executeSequentialStage executes a stage sequentially
func (taw *TracedAutomationWorkflow) executeSequentialStage(ctx context.Context, 
                                                           stage WorkflowStage, 
                                                           input interface{}) (interface{}, error) {
    task := trace.NewTask(ctx, stage.Name)
    defer task.End()
    
    return stage.Function(ctx, input)
}

// executeParallelStage executes a stage in parallel
func (taw *TracedAutomationWorkflow) executeParallelStage(ctx context.Context, 
                                                         stage WorkflowStage, 
                                                         input interface{}) (interface{}, error) {
    task := trace.NewTask(ctx, stage.Name)
    defer task.End()
    
    // For demonstration, assume input is a slice that can be processed in parallel
    inputSlice, ok := input.([]interface{})
    if !ok {
        return stage.Function(ctx, input)
    }
    
    maxWorkers := stage.MaxWorkers
    if maxWorkers <= 0 {
        maxWorkers = runtime.NumCPU()
    }
    
    jobs := make(chan interface{}, len(inputSlice))
    results := make(chan interface{}, len(inputSlice))
    errors := make(chan error, len(inputSlice))
    
    // Start workers
    var wg sync.WaitGroup
    for i := 0; i < maxWorkers; i++ {
        wg.Add(1)
        go func(workerID int) {
            defer wg.Done()
            
            workerCtx := trace.WithRegion(ctx, fmt.Sprintf("Worker-%d", workerID))
            defer trace.StartRegion(workerCtx, "WorkerExecution").End()
            
            for job := range jobs {
                result, err := stage.Function(workerCtx, job)
                if err != nil {
                    errors <- err
                    return
                }
                results <- result
            }
        }(i)
    }
    
    // Send jobs
    go func() {
        defer close(jobs)
        for _, item := range inputSlice {
            jobs <- item
        }
    }()
    
    // Wait for completion
    go func() {
        wg.Wait()
        close(results)
        close(errors)
    }()
    
    // Collect results
    var finalResults []interface{}
    for {
        select {
        case result, ok := <-results:
            if !ok {
                results = nil
            } else {
                finalResults = append(finalResults, result)
            }
        case err, ok := <-errors:
            if !ok {
                errors = nil
            } else {
                return nil, err
            }
        }
        
        if results == nil && errors == nil {
            break
        }
    }
    
    return finalResults, nil
}

// recordStageMetrics safely records stage execution metrics
func (taw *TracedAutomationWorkflow) recordStageMetrics(stageName string, metrics StageMetrics) {
    taw.metrics.mu.Lock()
    defer taw.metrics.mu.Unlock()
    
    taw.metrics.stageMetrics[stageName] = metrics
}

// GetMetrics returns workflow execution metrics
func (taw *TracedAutomationWorkflow) GetMetrics() WorkflowMetrics {
    taw.metrics.mu.RLock()
    defer taw.metrics.mu.RUnlock()
    
    // Return copy
    result := *taw.metrics
    result.stageMetrics = make(map[string]StageMetrics)
    for k, v := range taw.metrics.stageMetrics {
        result.stageMetrics[k] = v
    }
    
    return result
}

// Shutdown stops execution tracing
func (taw *TracedAutomationWorkflow) Shutdown() {
    if taw.traceFile != nil {
        trace.Stop()
        taw.traceFile.Close()
        log.Printf("Execution tracing stopped for workflow: %s", taw.name)
        log.Printf("Trace file: %s", taw.traceFile.Name())
        log.Printf("Analyze with: go tool trace %s", taw.traceFile.Name())
    }
}
```

**Prerequisites**: Sessions 32-34 (Testing, Benchmarking, and Basic Profiling)

**Next Steps**: Module 36 (Go Generics and Type Parameters) or Module 37 (Iterators and Range-over-Func)
