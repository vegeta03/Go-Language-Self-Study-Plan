# Module 31: Advanced Channel Patterns and Context

**Duration**: 1 hour (45 min theory + 15 min hands-on)

**Learning Objectives**:

- Master advanced channel patterns: drop, cancellation, context
- Understand context package for cancellation and timeouts
- Learn channel-based concurrency patterns
- Apply advanced patterns in automation systems
- Design robust, cancellable automation workflows

**Videos Covered**:

- 10.9 Drop Pattern (0:07:14)
- 10.10 Cancellation Pattern (0:08:15)
- 11.1 Topics (0:00:34)
- 11.2 Context Part 1 (0:16:23)
- 11.3 Context Part 2 (0:11:24)
- 11.4 Failure Detection (0:23:17)

**Key Concepts**:

- Drop pattern: handling overload gracefully
- Cancellation: stopping work in progress
- Context package: cancellation, deadlines, values
- Failure detection and recovery patterns
- Graceful shutdown patterns
- Resource cleanup with defer and context

**Hands-on Exercise 1: Drop Pattern and Overload Handling**:

Implementing drop patterns to handle system overload gracefully:

```go
// Drop pattern and overload handling in automation systems
package main

import (
    "fmt"
    "math/rand"
    "runtime"
    "sync"
    "sync/atomic"
    "time"
)

// PART 1: Drop Pattern Implementation
// Drop pattern prevents system overload by dropping tasks when queues are full

// AutomationTask represents work in the automation system
type AutomationTask struct {
    ID          string
    Type        string
    Data        string
    Priority    int
    CreatedAt   time.Time
    Deadline    time.Time
    RetryCount  int
}

// TaskResult represents the outcome of task processing
type TaskResult struct {
    TaskID      string
    Success     bool
    Output      string
    Error       error
    Duration    time.Duration
    ProcessedBy string
    CompletedAt time.Time
}

// DropStatistics tracks dropped task metrics
type DropStatistics struct {
    TotalSubmitted int64
    TotalProcessed int64
    TotalDropped   int64
    DropsByType    map[string]int64
    mu             sync.RWMutex
}

func NewDropStatistics() *DropStatistics {
    return &DropStatistics{
        DropsByType: make(map[string]int64),
    }
}

func (ds *DropStatistics) RecordSubmission() {
    atomic.AddInt64(&ds.TotalSubmitted, 1)
}

func (ds *DropStatistics) RecordProcessed() {
    atomic.AddInt64(&ds.TotalProcessed, 1)
}

func (ds *DropStatistics) RecordDropped(taskType string) {
    atomic.AddInt64(&ds.TotalDropped, 1)

    ds.mu.Lock()
    ds.DropsByType[taskType]++
    ds.mu.Unlock()
}

func (ds *DropStatistics) GetStats() (int64, int64, int64, map[string]int64) {
    ds.mu.RLock()
    defer ds.mu.RUnlock()

    dropsByType := make(map[string]int64)
    for k, v := range ds.DropsByType {
        dropsByType[k] = v
    }

    return atomic.LoadInt64(&ds.TotalSubmitted),
           atomic.LoadInt64(&ds.TotalProcessed),
           atomic.LoadInt64(&ds.TotalDropped),
           dropsByType
}

// PART 2: Basic Drop Pattern
func demonstrateBasicDropPattern() {
    fmt.Println("=== Basic Drop Pattern ===")

    const queueSize = 5
    const numTasks = 15
    const processingDelay = 200 * time.Millisecond

    taskQueue := make(chan AutomationTask, queueSize)
    resultQueue := make(chan TaskResult, numTasks)
    droppedQueue := make(chan AutomationTask, numTasks)

    stats := NewDropStatistics()

    // Fast producer (generates tasks faster than they can be processed)
    go func() {
        defer close(taskQueue)

        for i := 0; i < numTasks; i++ {
            task := AutomationTask{
                ID:        fmt.Sprintf("task-%03d", i),
                Type:      []string{"email", "file", "report", "backup"}[i%4],
                Data:      fmt.Sprintf("data-payload-%d", i),
                Priority:  rand.Intn(10),
                CreatedAt: time.Now(),
                Deadline:  time.Now().Add(5 * time.Second),
            }

            stats.RecordSubmission()

            // Try to queue task, drop if queue is full
            select {
            case taskQueue <- task:
                fmt.Printf("❌“ Queued: %s (type: %s, priority: %d)\n",
                    task.ID, task.Type, task.Priority)
            default:
                // Queue is full - drop the task
                stats.RecordDropped(task.Type)
                droppedQueue <- task
                fmt.Printf("✗ DROPPED: %s (queue full, type: %s)\n",
                    task.ID, task.Type)
            }

            time.Sleep(30 * time.Millisecond) // Fast production rate
        }
    }()

    // Slow worker (processes tasks slower than they're produced)
    go func() {
        defer close(resultQueue)

        for task := range taskQueue {
            start := time.Now()

            fmt.Printf("🔥”„ Processing: %s (type: %s)\n", task.ID, task.Type)

            // Simulate processing time
            time.Sleep(processingDelay)

            stats.RecordProcessed()

            result := TaskResult{
                TaskID:      task.ID,
                Success:     rand.Float32() > 0.1, // 90% success rate
                Output:      fmt.Sprintf("processed-%s", task.Data),
                Duration:    time.Since(start),
                ProcessedBy: "worker-1",
                CompletedAt: time.Now(),
            }

            if !result.Success {
                result.Error = fmt.Errorf("processing failed for task %s", task.ID)
            }

            resultQueue <- result
        }
    }()

    // Collect dropped tasks
    go func() {
        defer close(droppedQueue)
        time.Sleep(3 * time.Second) // Wait for producer to finish
    }()

    // Collect results and dropped tasks
    var results []TaskResult
    var droppedTasks []AutomationTask

    for {
        select {
        case result, ok := <-resultQueue:
            if !ok {
                resultQueue = nil
            } else {
                results = append(results, result)
                status := "SUCCESS"
                if !result.Success {
                    status = "FAILED"
                }
                fmt.Printf("✅ Result: %s - %s (%v)\n",
                    result.TaskID, status, result.Duration)
            }

        case dropped, ok := <-droppedQueue:
            if !ok {
                droppedQueue = nil
            } else {
                droppedTasks = append(droppedTasks, dropped)
            }
        }

        if resultQueue == nil && droppedQueue == nil {
            break
        }
    }

    // Display statistics
    submitted, processed, dropped, dropsByType := stats.GetStats()

    fmt.Printf("\n=== Drop Pattern Statistics ===\n")
    fmt.Printf("Total Submitted: %d\n", submitted)
    fmt.Printf("Total Processed: %d\n", processed)
    fmt.Printf("Total Dropped: %d\n", dropped)
    fmt.Printf("Drop Rate: %.2f%%\n", float64(dropped)/float64(submitted)*100)

    fmt.Println("\nDrops by Task Type:")
    for taskType, count := range dropsByType {
        fmt.Printf("  %s: %d\n", taskType, count)
    }
}

// PART 3: Priority-Based Drop Pattern
func demonstratePriorityDropPattern() {
    fmt.Println("\n=== Priority-Based Drop Pattern ===")

    const queueSize = 4
    const numTasks = 12

    taskQueue := make(chan AutomationTask, queueSize)
    resultQueue := make(chan TaskResult, numTasks)

    stats := NewDropStatistics()

    // Producer with mixed priority tasks
    go func() {
        defer close(taskQueue)

        for i := 0; i < numTasks; i++ {
            task := AutomationTask{
                ID:        fmt.Sprintf("priority-task-%03d", i),
                Type:      []string{"critical", "normal", "low"}[i%3],
                Data:      fmt.Sprintf("priority-data-%d", i),
                Priority:  []int{9, 5, 1}[i%3], // Critical=9, Normal=5, Low=1
                CreatedAt: time.Now(),
                Deadline:  time.Now().Add(3 * time.Second),
            }

            stats.RecordSubmission()

            // Priority-based dropping logic
            select {
            case taskQueue <- task:
                fmt.Printf("❌“ Queued: %s (priority: %d, type: %s)\n",
                    task.ID, task.Priority, task.Type)
            default:
                // Queue is full - implement priority-based dropping
                if task.Priority >= 7 { // High priority tasks
                    // Try to make room by dropping a lower priority task
                    if tryDropLowerPriorityTask(taskQueue, task) {
                        fmt.Printf("❌“ High-priority queued: %s (displaced lower priority)\n", task.ID)
                    } else {
                        stats.RecordDropped(task.Type)
                        fmt.Printf("✗ DROPPED high-priority: %s (no lower priority to displace)\n", task.ID)
                    }
                } else {
                    // Low/medium priority - just drop
                    stats.RecordDropped(task.Type)
                    fmt.Printf("✗ DROPPED low-priority: %s (queue full)\n", task.ID)
                }
            }

            time.Sleep(50 * time.Millisecond)
        }
    }()

    // Worker
    go func() {
        defer close(resultQueue)

        for task := range taskQueue {
            start := time.Now()

            // Process based on priority (higher priority = faster processing)
            processingTime := time.Duration(1000-task.Priority*100) * time.Millisecond
            time.Sleep(processingTime)

            stats.RecordProcessed()

            result := TaskResult{
                TaskID:      task.ID,
                Success:     true,
                Output:      fmt.Sprintf("priority-processed-%s", task.Data),
                Duration:    time.Since(start),
                ProcessedBy: "priority-worker",
                CompletedAt: time.Now(),
            }

            resultQueue <- result
            fmt.Printf("✅ Completed: %s (priority: %d, duration: %v)\n",
                task.ID, task.Priority, result.Duration)
        }
    }()

    // Collect results
    var results []TaskResult
    for result := range resultQueue {
        results = append(results, result)
    }

    // Display priority-based statistics
    submitted, processed, dropped, dropsByType := stats.GetStats()

    fmt.Printf("\n=== Priority Drop Pattern Statistics ===\n")
    fmt.Printf("Total Submitted: %d\n", submitted)
    fmt.Printf("Total Processed: %d\n", processed)
    fmt.Printf("Total Dropped: %d\n", dropped)
    fmt.Printf("Processing Rate: %.2f%%\n", float64(processed)/float64(submitted)*100)

    fmt.Println("\nDrops by Task Type:")
    for taskType, count := range dropsByType {
        fmt.Printf("  %s: %d\n", taskType, count)
    }
}

// tryDropLowerPriorityTask attempts to make room for high-priority tasks
// Note: This is a simplified implementation for demonstration
func tryDropLowerPriorityTask(queue chan AutomationTask, newTask AutomationTask) bool {
    // In a real implementation, you would need a more sophisticated queue
    // that allows inspection and removal of lower priority items
    // For this demo, we'll just return false to show the concept
    return false
}

// PART 4: Adaptive Drop Pattern
func demonstrateAdaptiveDropPattern() {
    fmt.Println("\n=== Adaptive Drop Pattern ===")

    const initialQueueSize = 3
    const numTasks = 20

    // Adaptive queue that can adjust drop behavior based on system load
    taskQueue := make(chan AutomationTask, initialQueueSize)
    resultQueue := make(chan TaskResult, numTasks)

    stats := NewDropStatistics()

    // System load monitor
    systemLoad := int32(0) // 0=low, 1=medium, 2=high

    go func() {
        ticker := time.NewTicker(500 * time.Millisecond)
        defer ticker.Stop()

        for range ticker.C {
            queueLength := len(taskQueue)
            queueCapacity := cap(taskQueue)

            loadRatio := float64(queueLength) / float64(queueCapacity)

            var newLoad int32
            if loadRatio > 0.8 {
                newLoad = 2 // High load
            } else if loadRatio > 0.5 {
                newLoad = 1 // Medium load
            } else {
                newLoad = 0 // Low load
            }

            atomic.StoreInt32(&systemLoad, newLoad)

            loadDesc := []string{"LOW", "MEDIUM", "HIGH"}[newLoad]
            fmt.Printf("🔥“Š System Load: %s (queue: %d/%d, ratio: %.2f)\n",
                loadDesc, queueLength, queueCapacity, loadRatio)
        }
    }()

    // Adaptive producer
    go func() {
        defer close(taskQueue)

        for i := 0; i < numTasks; i++ {
            task := AutomationTask{
                ID:        fmt.Sprintf("adaptive-task-%03d", i),
                Type:      []string{"batch", "realtime", "maintenance"}[i%3],
                Data:      fmt.Sprintf("adaptive-data-%d", i),
                Priority:  rand.Intn(10),
                CreatedAt: time.Now(),
                Deadline:  time.Now().Add(2 * time.Second),
            }

            stats.RecordSubmission()

            // Adaptive dropping based on system load
            currentLoad := atomic.LoadInt32(&systemLoad)

            shouldDrop := false
            switch currentLoad {
            case 2: // High load - drop low priority tasks
                shouldDrop = task.Priority < 7
            case 1: // Medium load - drop very low priority tasks
                shouldDrop = task.Priority < 3
            case 0: // Low load - accept all tasks
                shouldDrop = false
            }

            if shouldDrop {
                stats.RecordDropped(task.Type)
                fmt.Printf("✗ ADAPTIVE DROP: %s (load: %s, priority: %d)\n",
                    task.ID, []string{"LOW", "MEDIUM", "HIGH"}[currentLoad], task.Priority)
                continue
            }

            select {
            case taskQueue <- task:
                fmt.Printf("❌“ Adaptive queued: %s (priority: %d)\n", task.ID, task.Priority)
            default:
                stats.RecordDropped(task.Type)
                fmt.Printf("✗ QUEUE FULL DROP: %s\n", task.ID)
            }

            time.Sleep(100 * time.Millisecond)
        }
    }()

    // Worker
    go func() {
        defer close(resultQueue)

        for task := range taskQueue {
            start := time.Now()

            // Variable processing time
            processingTime := time.Duration(rand.Intn(300)+100) * time.Millisecond
            time.Sleep(processingTime)

            stats.RecordProcessed()

            result := TaskResult{
                TaskID:      task.ID,
                Success:     true,
                Output:      fmt.Sprintf("adaptive-processed-%s", task.Data),
                Duration:    time.Since(start),
                ProcessedBy: "adaptive-worker",
                CompletedAt: time.Now(),
            }

            resultQueue <- result
        }
    }()

    // Collect results
    var results []TaskResult
    for result := range resultQueue {
        results = append(results, result)
    }

    // Final statistics
    submitted, processed, dropped, dropsByType := stats.GetStats()

    fmt.Printf("\n=== Adaptive Drop Pattern Statistics ===\n")
    fmt.Printf("Total Submitted: %d\n", submitted)
    fmt.Printf("Total Processed: %d\n", processed)
    fmt.Printf("Total Dropped: %d\n", dropped)
    fmt.Printf("Efficiency: %.2f%%\n", float64(processed)/float64(submitted)*100)

    fmt.Println("\nDrops by Task Type:")
    for taskType, count := range dropsByType {
        fmt.Printf("  %s: %d\n", taskType, count)
    }
}

func main() {
    fmt.Printf("Starting with GOMAXPROCS=%d\n", runtime.GOMAXPROCS(0))
    fmt.Println("=== Drop Pattern and Overload Handling Demo ===")

    // Demonstrate basic drop pattern
    demonstrateBasicDropPattern()

    // Demonstrate priority-based dropping
    demonstratePriorityDropPattern()

    // Demonstrate adaptive dropping
    demonstrateAdaptiveDropPattern()

    fmt.Println("\n=== Drop Pattern Key Insights ===")
    fmt.Println("✅ BASIC DROP: Prevent overload by dropping tasks when queues are full")
    fmt.Println("✅ PRIORITY DROP: Preserve high-priority tasks, drop low-priority ones")
    fmt.Println("✅ ADAPTIVE DROP: Adjust drop behavior based on system load")
    fmt.Println("✅ STATISTICS: Track drop rates and patterns for system optimization")
    fmt.Println("✅ GRACEFUL DEGRADATION: Maintain system stability under load")
    fmt.Println("✅ RESOURCE PROTECTION: Prevent memory exhaustion and system crashes")
}
```

**Hands-on Exercise 2: Cancellation Pattern and Context Management**:

Implementing cancellation patterns with context for robust automation workflows:

```go
// Cancellation patterns and context management in automation systems
package main

import (
    "context"
    "fmt"
    "math/rand"
    "runtime"
    "sync"
    "sync/atomic"
    "time"
)

// PART 1: Context-Aware Task Processing
// Context provides cancellation, deadlines, and request-scoped values

// CancellableTask represents work that can be cancelled
type CancellableTask struct {
    ID          string
    Type        string
    Data        string
    Priority    int
    Timeout     time.Duration
    CreatedAt   time.Time
    MaxRetries  int
}

// TaskResult represents the outcome of cancellable task processing
type TaskResult struct {
    TaskID      string
    Success     bool
    Output      string
    Error       error
    Duration    time.Duration
    ProcessedBy string
    Cancelled   bool
    Retries     int
    CompletedAt time.Time
}

// CancellationMetrics tracks cancellation statistics
type CancellationMetrics struct {
    TasksStarted    int64
    TasksCompleted  int64
    TasksCancelled  int64
    TasksTimedOut   int64
    TotalDuration   int64 // in nanoseconds
}

func (cm *CancellationMetrics) RecordStarted() {
    atomic.AddInt64(&cm.TasksStarted, 1)
}

func (cm *CancellationMetrics) RecordCompleted(duration time.Duration) {
    atomic.AddInt64(&cm.TasksCompleted, 1)
    atomic.AddInt64(&cm.TotalDuration, duration.Nanoseconds())
}

func (cm *CancellationMetrics) RecordCancelled() {
    atomic.AddInt64(&cm.TasksCancelled, 1)
}

func (cm *CancellationMetrics) RecordTimedOut() {
    atomic.AddInt64(&cm.TasksTimedOut, 1)
}

func (cm *CancellationMetrics) GetStats() (int64, int64, int64, int64, time.Duration) {
    started := atomic.LoadInt64(&cm.TasksStarted)
    completed := atomic.LoadInt64(&cm.TasksCompleted)
    cancelled := atomic.LoadInt64(&cm.TasksCancelled)
    timedOut := atomic.LoadInt64(&cm.TasksTimedOut)
    totalDuration := atomic.LoadInt64(&cm.TotalDuration)

    var avgDuration time.Duration
    if completed > 0 {
        avgDuration = time.Duration(totalDuration / completed)
    }

    return started, completed, cancelled, timedOut, avgDuration
}

// PART 2: Basic Cancellation Pattern
func demonstrateBasicCancellation() {
    fmt.Println("=== Basic Cancellation Pattern ===")

    // Create context with timeout
    ctx, cancel := context.WithTimeout(context.Background(), 3*time.Second)
    defer cancel()

    taskQueue := make(chan CancellableTask, 10)
    resultQueue := make(chan TaskResult, 10)

    metrics := &CancellationMetrics{}

    // Start workers with context
    var wg sync.WaitGroup
    numWorkers := 3

    for i := 0; i < numWorkers; i++ {
        wg.Add(1)
        go cancellableWorker(ctx, i, taskQueue, resultQueue, metrics, &wg)
    }

    // Task generator
    go func() {
        defer close(taskQueue)

        for i := 0; i < 8; i++ {
            task := CancellableTask{
                ID:         fmt.Sprintf("basic-task-%03d", i),
                Type:       []string{"email", "report", "backup", "sync"}[i%4],
                Data:       fmt.Sprintf("task-data-%d", i),
                Priority:   rand.Intn(10),
                Timeout:    time.Duration(rand.Intn(2000)+1000) * time.Millisecond,
                CreatedAt:  time.Now(),
                MaxRetries: 2,
            }

            select {
            case taskQueue <- task:
                fmt.Printf("🔥“¤ Queued: %s (timeout: %v)\n", task.ID, task.Timeout)
            case <-ctx.Done():
                fmt.Println("⏰ Context cancelled, stopping task generation")
                return
            }

            time.Sleep(200 * time.Millisecond)
        }
    }()

    // Close results when all workers are done
    go func() {
        wg.Wait()
        close(resultQueue)
    }()

    // Collect results
    var results []TaskResult
    for result := range resultQueue {
        results = append(results, result)

        if result.Success {
            fmt.Printf("✅ Completed: %s by %s (%v)\n",
                result.TaskID, result.ProcessedBy, result.Duration)
        } else if result.Cancelled {
            fmt.Printf("❌ Cancelled: %s - %v\n", result.TaskID, result.Error)
        } else {
            fmt.Printf("⚙️ ï¸  Failed: %s - %v\n", result.TaskID, result.Error)
        }
    }

    // Display metrics
    started, completed, cancelled, timedOut, avgDuration := metrics.GetStats()

    fmt.Printf("\n=== Basic Cancellation Statistics ===\n")
    fmt.Printf("Tasks Started: %d\n", started)
    fmt.Printf("Tasks Completed: %d\n", completed)
    fmt.Printf("Tasks Cancelled: %d\n", cancelled)
    fmt.Printf("Tasks Timed Out: %d\n", timedOut)
    fmt.Printf("Average Duration: %v\n", avgDuration)
    fmt.Printf("Success Rate: %.2f%%\n", float64(completed)/float64(started)*100)
}

// cancellableWorker processes tasks with cancellation support
func cancellableWorker(ctx context.Context, workerID int, tasks <-chan CancellableTask,
                      results chan<- TaskResult, metrics *CancellationMetrics, wg *sync.WaitGroup) {
    defer wg.Done()

    workerName := fmt.Sprintf("worker-%d", workerID)
    fmt.Printf("🔥š€ %s started\n", workerName)

    for {
        select {
        case task, ok := <-tasks:
            if !ok {
                fmt.Printf("🔥›‘ %s: Task channel closed\n", workerName)
                return
            }

            metrics.RecordStarted()
            result := processCancellableTask(ctx, task, workerName, metrics)

            select {
            case results <- result:
                // Result sent successfully
            case <-ctx.Done():
                fmt.Printf("🔥›‘ %s: Context cancelled while sending result\n", workerName)
                return
            }

        case <-ctx.Done():
            fmt.Printf("🔥›‘ %s: Context cancelled\n", workerName)
            return
        }
    }
}

// processCancellableTask handles individual task processing with cancellation
func processCancellableTask(ctx context.Context, task CancellableTask, workerName string,
                           metrics *CancellationMetrics) TaskResult {
    start := time.Now()

    // Create task-specific context with timeout
    taskCtx, cancel := context.WithTimeout(ctx, task.Timeout)
    defer cancel()

    fmt.Printf("🔥”„ %s: Processing %s (timeout: %v)\n", workerName, task.ID, task.Timeout)

    // Simulate work with cancellation checks
    workSteps := 10
    stepDuration := task.Timeout / time.Duration(workSteps)

    for i := 0; i < workSteps; i++ {
        select {
        case <-taskCtx.Done():
            // Task was cancelled or timed out
            duration := time.Since(start)

            var cancelled bool
            var err error

            if taskCtx.Err() == context.DeadlineExceeded {
                metrics.RecordTimedOut()
                err = fmt.Errorf("task timeout after %v", duration)
            } else {
                metrics.RecordCancelled()
                cancelled = true
                err = fmt.Errorf("task cancelled after %v", duration)
            }

            return TaskResult{
                TaskID:      task.ID,
                Success:     false,
                Error:       err,
                Duration:    duration,
                ProcessedBy: workerName,
                Cancelled:   cancelled,
                CompletedAt: time.Now(),
            }

        default:
            // Simulate work step
            time.Sleep(stepDuration)
        }
    }

    // Task completed successfully
    duration := time.Since(start)
    metrics.RecordCompleted(duration)

    return TaskResult{
        TaskID:      task.ID,
        Success:     true,
        Output:      fmt.Sprintf("processed-%s", task.Data),
        Duration:    duration,
        ProcessedBy: workerName,
        CompletedAt: time.Now(),
    }
}

// PART 3: Hierarchical Cancellation
func demonstrateHierarchicalCancellation() {
    fmt.Println("\n=== Hierarchical Cancellation Pattern ===")

    // Root context with overall timeout
    rootCtx, rootCancel := context.WithTimeout(context.Background(), 5*time.Second)
    defer rootCancel()

    // Service-level context
    serviceCtx, serviceCancel := context.WithTimeout(rootCtx, 4*time.Second)
    defer serviceCancel()

    // Batch-level context
    batchCtx, batchCancel := context.WithTimeout(serviceCtx, 3*time.Second)
    defer batchCancel()

    var wg sync.WaitGroup

    // Start multiple service components
    wg.Add(1)
    go func() {
        defer wg.Done()
        runEmailService(serviceCtx, "EmailService")
    }()

    wg.Add(1)
    go func() {
        defer wg.Done()
        runReportService(serviceCtx, "ReportService")
    }()

    wg.Add(1)
    go func() {
        defer wg.Done()
        runBatchProcessor(batchCtx, "BatchProcessor")
    }()

    // Simulate external cancellation after 2 seconds
    go func() {
        time.Sleep(2 * time.Second)
        fmt.Println("🔥š¨ External cancellation triggered!")
        batchCancel() // Cancel batch processing early
    }()

    wg.Wait()

    fmt.Println("✅ Hierarchical cancellation demonstration completed")
}

func runEmailService(ctx context.Context, serviceName string) {
    fmt.Printf("🔥“§ %s started\n", serviceName)

    ticker := time.NewTicker(500 * time.Millisecond)
    defer ticker.Stop()

    emailCount := 0

    for {
        select {
        case <-ticker.C:
            emailCount++
            fmt.Printf("🔥“§ %s: Sent email #%d\n", serviceName, emailCount)

        case <-ctx.Done():
            fmt.Printf("🔥“§ %s: Cancelled after sending %d emails (%v)\n",
                serviceName, emailCount, ctx.Err())
            return
        }
    }
}

func runReportService(ctx context.Context, serviceName string) {
    fmt.Printf("🔥“Š %s started\n", serviceName)

    ticker := time.NewTicker(800 * time.Millisecond)
    defer ticker.Stop()

    reportCount := 0

    for {
        select {
        case <-ticker.C:
            reportCount++
            fmt.Printf("🔥“Š %s: Generated report #%d\n", serviceName, reportCount)

        case <-ctx.Done():
            fmt.Printf("🔥“Š %s: Cancelled after generating %d reports (%v)\n",
                serviceName, reportCount, ctx.Err())
            return
        }
    }
}

func runBatchProcessor(ctx context.Context, serviceName string) {
    fmt.Printf("⚙️  %s started\n", serviceName)

    ticker := time.NewTicker(300 * time.Millisecond)
    defer ticker.Stop()

    batchCount := 0

    for {
        select {
        case <-ticker.C:
            batchCount++
            fmt.Printf("⚙️  %s: Processed batch #%d\n", serviceName, batchCount)

        case <-ctx.Done():
            fmt.Printf("⚙️  %s: Cancelled after processing %d batches (%v)\n",
                serviceName, batchCount, ctx.Err())
            return
        }
    }
}

// PART 4: Context Values and Request Tracing
func demonstrateContextValues() {
    fmt.Println("\n=== Context Values and Request Tracing ===")

    // Create context with values for request tracing
    ctx := context.Background()
    ctx = context.WithValue(ctx, "requestID", "req-12345")
    ctx = context.WithValue(ctx, "userID", "user-67890")
    ctx = context.WithValue(ctx, "sessionID", "sess-abcdef")

    // Add timeout
    ctx, cancel := context.WithTimeout(ctx, 2*time.Second)
    defer cancel()

    // Process request through multiple services
    processRequest(ctx)
}

func processRequest(ctx context.Context) {
    requestID := ctx.Value("requestID").(string)
    userID := ctx.Value("userID").(string)

    fmt.Printf("🔥” Processing request %s for user %s\n", requestID, userID)

    // Call authentication service
    if !authenticateUser(ctx) {
        fmt.Printf("❌ Authentication failed for request %s\n", requestID)
        return
    }

    // Call business logic service
    if !processBusinessLogic(ctx) {
        fmt.Printf("❌ Business logic failed for request %s\n", requestID)
        return
    }

    // Call data persistence service
    if !persistData(ctx) {
        fmt.Printf("❌ Data persistence failed for request %s\n", requestID)
        return
    }

    fmt.Printf("✅ Request %s completed successfully\n", requestID)
}

func authenticateUser(ctx context.Context) bool {
    requestID := ctx.Value("requestID").(string)
    userID := ctx.Value("userID").(string)

    fmt.Printf("🔥” Auth service: Authenticating user %s (request: %s)\n", userID, requestID)

    select {
    case <-time.After(200 * time.Millisecond):
        fmt.Printf("🔥” Auth service: User %s authenticated\n", userID)
        return true
    case <-ctx.Done():
        fmt.Printf("🔥” Auth service: Authentication cancelled (%v)\n", ctx.Err())
        return false
    }
}

func processBusinessLogic(ctx context.Context) bool {
    requestID := ctx.Value("requestID").(string)

    fmt.Printf("⚙️  Business service: Processing logic (request: %s)\n", requestID)

    select {
    case <-time.After(500 * time.Millisecond):
        fmt.Printf("⚙️  Business service: Logic processed\n")
        return true
    case <-ctx.Done():
        fmt.Printf("⚙️  Business service: Processing cancelled (%v)\n", ctx.Err())
        return false
    }
}

func persistData(ctx context.Context) bool {
    requestID := ctx.Value("requestID").(string)

    fmt.Printf("🔥’¾ Data service: Persisting data (request: %s)\n", requestID)

    select {
    case <-time.After(300 * time.Millisecond):
        fmt.Printf("🔥’¾ Data service: Data persisted\n")
        return true
    case <-ctx.Done():
        fmt.Printf("🔥’¾ Data service: Persistence cancelled (%v)\n", ctx.Err())
        return false
    }
}

func main() {
    fmt.Printf("Starting with GOMAXPROCS=%d\n", runtime.GOMAXPROCS(0))
    fmt.Println("=== Cancellation Pattern and Context Management Demo ===")

    // Demonstrate basic cancellation
    demonstrateBasicCancellation()

    // Demonstrate hierarchical cancellation
    demonstrateHierarchicalCancellation()

    // Demonstrate context values
    demonstrateContextValues()

    fmt.Println("\n=== Cancellation Pattern Key Insights ===")
    fmt.Println("✅ CONTEXT TIMEOUT: Set deadlines for operations")
    fmt.Println("✅ CANCELLATION PROPAGATION: Cancel dependent operations")
    fmt.Println("✅ HIERARCHICAL CANCELLATION: Organize cancellation scopes")
    fmt.Println("✅ CONTEXT VALUES: Pass request-scoped data")
    fmt.Println("✅ GRACEFUL SHUTDOWN: Clean up resources on cancellation")
    fmt.Println("✅ TIMEOUT HANDLING: Prevent operations from running indefinitely")
}
```

**Hands-on Exercise 3: Failure Detection and Recovery Patterns**:

Implementing failure detection and recovery mechanisms for resilient automation systems:

```go
// Failure detection and recovery patterns in automation systems
package main

import (
    "context"
    "fmt"
    "math/rand"
    "runtime"
    "sync"
    "sync/atomic"
    "time"
)

// PART 1: Failure Detection and Recovery Infrastructure

// FailureDetectionTask represents work with failure detection capabilities
type FailureDetectionTask struct {
    ID          string
    Type        string
    Data        string
    Priority    int
    MaxRetries  int
    Timeout     time.Duration
    CreatedAt   time.Time
    Attempts    int
}

// TaskOutcome represents the result of task processing with failure details
type TaskOutcome struct {
    TaskID       string
    Success      bool
    Output       string
    Error        error
    Duration     time.Duration
    ProcessedBy  string
    Attempts     int
    FailureType  FailureType
    RecoveryUsed bool
    CompletedAt  time.Time
}

// FailureType categorizes different types of failures
type FailureType int

const (
    NoFailure FailureType = iota
    TimeoutFailure
    ProcessingFailure
    ResourceFailure
    NetworkFailure
    ValidationFailure
)

func (ft FailureType) String() string {
    switch ft {
    case NoFailure:
        return "none"
    case TimeoutFailure:
        return "timeout"
    case ProcessingFailure:
        return "processing"
    case ResourceFailure:
        return "resource"
    case NetworkFailure:
        return "network"
    case ValidationFailure:
        return "validation"
    default:
        return "unknown"
    }
}

// FailureDetector monitors and categorizes failures
type FailureDetector struct {
    failureCounts map[FailureType]int64
    mu            sync.RWMutex
}

func NewFailureDetector() *FailureDetector {
    return &FailureDetector{
        failureCounts: make(map[FailureType]int64),
    }
}

func (fd *FailureDetector) RecordFailure(failureType FailureType) {
    fd.mu.Lock()
    fd.failureCounts[failureType]++
    fd.mu.Unlock()
}

func (fd *FailureDetector) GetFailureCounts() map[FailureType]int64 {
    fd.mu.RLock()
    defer fd.mu.RUnlock()

    counts := make(map[FailureType]int64)
    for ft, count := range fd.failureCounts {
        counts[ft] = count
    }
    return counts
}

// PART 2: Resilient Automation Service
type ResilientAutomationService struct {
    name            string
    workers         int
    timeout         time.Duration
    failureDetector *FailureDetector
    circuitBreaker  *CircuitBreaker
    retryPolicy     *RetryPolicy
}

// CircuitBreaker prevents cascading failures
type CircuitBreaker struct {
    failureThreshold int
    resetTimeout     time.Duration
    failureCount     int64
    lastFailureTime  time.Time
    state            CircuitState
    mu               sync.RWMutex
}

type CircuitState int

const (
    CircuitClosed CircuitState = iota
    CircuitOpen
    CircuitHalfOpen
)

func (cs CircuitState) String() string {
    switch cs {
    case CircuitClosed:
        return "CLOSED"
    case CircuitOpen:
        return "OPEN"
    case CircuitHalfOpen:
        return "HALF-OPEN"
    default:
        return "UNKNOWN"
    }
}

func NewCircuitBreaker(failureThreshold int, resetTimeout time.Duration) *CircuitBreaker {
    return &CircuitBreaker{
        failureThreshold: failureThreshold,
        resetTimeout:     resetTimeout,
        state:            CircuitClosed,
    }
}

func (cb *CircuitBreaker) CanExecute() bool {
    cb.mu.Lock()
    defer cb.mu.Unlock()

    switch cb.state {
    case CircuitClosed:
        return true
    case CircuitOpen:
        if time.Since(cb.lastFailureTime) > cb.resetTimeout {
            cb.state = CircuitHalfOpen
            return true
        }
        return false
    case CircuitHalfOpen:
        return true
    default:
        return false
    }
}

func (cb *CircuitBreaker) RecordSuccess() {
    cb.mu.Lock()
    defer cb.mu.Unlock()

    cb.failureCount = 0
    cb.state = CircuitClosed
}

func (cb *CircuitBreaker) RecordFailure() {
    cb.mu.Lock()
    defer cb.mu.Unlock()

    cb.failureCount++
    cb.lastFailureTime = time.Now()

    if cb.failureCount >= int64(cb.failureThreshold) {
        cb.state = CircuitOpen
    }
}

func (cb *CircuitBreaker) GetState() CircuitState {
    cb.mu.RLock()
    defer cb.mu.RUnlock()
    return cb.state
}

// RetryPolicy defines retry behavior
type RetryPolicy struct {
    maxRetries      int
    baseDelay       time.Duration
    maxDelay        time.Duration
    backoffFactor   float64
    retryableErrors map[FailureType]bool
}

func NewRetryPolicy(maxRetries int, baseDelay time.Duration) *RetryPolicy {
    return &RetryPolicy{
        maxRetries:    maxRetries,
        baseDelay:     baseDelay,
        maxDelay:      10 * time.Second,
        backoffFactor: 2.0,
        retryableErrors: map[FailureType]bool{
            TimeoutFailure:    true,
            NetworkFailure:    true,
            ResourceFailure:   true,
            ProcessingFailure: false, // Don't retry processing errors
            ValidationFailure: false, // Don't retry validation errors
        },
    }
}

func (rp *RetryPolicy) ShouldRetry(failureType FailureType, attempts int) bool {
    if attempts >= rp.maxRetries {
        return false
    }
    return rp.retryableErrors[failureType]
}

func (rp *RetryPolicy) GetDelay(attempts int) time.Duration {
    delay := time.Duration(float64(rp.baseDelay) *
        (1 << uint(attempts-1)) * rp.backoffFactor)

    if delay > rp.maxDelay {
        delay = rp.maxDelay
    }

    return delay
}

// NewResilientAutomationService creates a service with failure detection and recovery
func NewResilientAutomationService(name string, workers int, timeout time.Duration) *ResilientAutomationService {
    return &ResilientAutomationService{
        name:            name,
        workers:         workers,
        timeout:         timeout,
        failureDetector: NewFailureDetector(),
        circuitBreaker:  NewCircuitBreaker(5, 30*time.Second),
        retryPolicy:     NewRetryPolicy(3, 100*time.Millisecond),
    }
}

// PART 3: Failure Detection and Recovery Implementation
func demonstrateFailureDetectionAndRecovery() {
    fmt.Println("=== Failure Detection and Recovery ===")

    ctx, cancel := context.WithTimeout(context.Background(), 10*time.Second)
    defer cancel()

    service := NewResilientAutomationService("ResilientService", 3, 2*time.Second)

    // Create tasks with different failure characteristics
    tasks := []FailureDetectionTask{
        {ID: "reliable-1", Type: "reliable", Data: "stable-data", Priority: 5, MaxRetries: 2, Timeout: 1 * time.Second},
        {ID: "flaky-1", Type: "flaky", Data: "unstable-data", Priority: 3, MaxRetries: 3, Timeout: 1 * time.Second},
        {ID: "slow-1", Type: "slow", Data: "heavy-data", Priority: 1, MaxRetries: 2, Timeout: 3 * time.Second},
        {ID: "reliable-2", Type: "reliable", Data: "stable-data-2", Priority: 5, MaxRetries: 2, Timeout: 1 * time.Second},
        {ID: "flaky-2", Type: "flaky", Data: "unstable-data-2", Priority: 3, MaxRetries: 3, Timeout: 1 * time.Second},
        {ID: "timeout-1", Type: "timeout", Data: "timeout-data", Priority: 2, MaxRetries: 2, Timeout: 500 * time.Millisecond},
        {ID: "reliable-3", Type: "reliable", Data: "stable-data-3", Priority: 5, MaxRetries: 2, Timeout: 1 * time.Second},
    }

    results := service.ProcessTasksWithRecovery(ctx, tasks)

    // Analyze results
    var successful, failed, recovered int

    fmt.Println("\n=== Task Processing Results ===")
    for _, result := range results {
        if result.Success {
            successful++
            status := "SUCCESS"
            if result.RecoveryUsed {
                status += " (RECOVERED)"
                recovered++
            }
            fmt.Printf("✅ %s: %s - %s (attempts: %d, duration: %v)\n",
                result.TaskID, status, result.Output, result.Attempts, result.Duration)
        } else {
            failed++
            fmt.Printf("❌ %s: FAILED - %v (attempts: %d, failure: %s)\n",
                result.TaskID, result.Error, result.Attempts, result.FailureType)
        }
    }

    // Display failure statistics
    failureCounts := service.failureDetector.GetFailureCounts()
    circuitState := service.circuitBreaker.GetState()

    fmt.Printf("\n=== Failure Detection Statistics ===\n")
    fmt.Printf("Total Tasks: %d\n", len(results))
    fmt.Printf("Successful: %d\n", successful)
    fmt.Printf("Failed: %d\n", failed)
    fmt.Printf("Recovered: %d\n", recovered)
    fmt.Printf("Success Rate: %.2f%%\n", float64(successful)/float64(len(results))*100)
    fmt.Printf("Recovery Rate: %.2f%%\n", float64(recovered)/float64(successful)*100)
    fmt.Printf("Circuit Breaker State: %s\n", circuitState)

    fmt.Println("\nFailure Breakdown:")
    for failureType, count := range failureCounts {
        if count > 0 {
            fmt.Printf("  %s: %d\n", failureType, count)
        }
    }
}

// ProcessTasksWithRecovery processes tasks with comprehensive failure handling
func (ras *ResilientAutomationService) ProcessTasksWithRecovery(ctx context.Context,
                                                               tasks []FailureDetectionTask) []TaskOutcome {
    fmt.Printf("🔥š€ %s: Processing %d tasks with failure detection and recovery\n",
        ras.name, len(tasks))

    taskQueue := make(chan FailureDetectionTask, len(tasks))
    resultQueue := make(chan TaskOutcome, len(tasks))

    // Start resilient workers
    var wg sync.WaitGroup
    for i := 0; i < ras.workers; i++ {
        wg.Add(1)
        go ras.resilientWorker(ctx, i, taskQueue, resultQueue, &wg)
    }

    // Send tasks
    go func() {
        defer close(taskQueue)
        for _, task := range tasks {
            task.CreatedAt = time.Now()
            select {
            case taskQueue <- task:
            case <-ctx.Done():
                return
            }
        }
    }()

    // Close results when workers are done
    go func() {
        wg.Wait()
        close(resultQueue)
    }()

    // Collect results
    var results []TaskOutcome
    for result := range resultQueue {
        results = append(results, result)
    }

    return results
}

// resilientWorker processes tasks with failure detection and recovery
func (ras *ResilientAutomationService) resilientWorker(ctx context.Context, workerID int,
                                                      tasks <-chan FailureDetectionTask,
                                                      results chan<- TaskOutcome, wg *sync.WaitGroup) {
    defer wg.Done()

    workerName := fmt.Sprintf("resilient-worker-%d", workerID)

    for {
        select {
        case task, ok := <-tasks:
            if !ok {
                return
            }

            result := ras.processTaskWithRetry(ctx, task, workerName)

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

// processTaskWithRetry handles task processing with retry logic
func (ras *ResilientAutomationService) processTaskWithRetry(ctx context.Context,
                                                           task FailureDetectionTask,
                                                           workerName string) TaskOutcome {
    var lastError error
    var lastFailureType FailureType
    var totalDuration time.Duration

    for attempt := 1; attempt <= task.MaxRetries+1; attempt++ {
        // Check circuit breaker
        if !ras.circuitBreaker.CanExecute() {
            ras.failureDetector.RecordFailure(ResourceFailure)
            return TaskOutcome{
                TaskID:      task.ID,
                Success:     false,
                Error:       fmt.Errorf("circuit breaker open"),
                Duration:    totalDuration,
                ProcessedBy: workerName,
                Attempts:    attempt,
                FailureType: ResourceFailure,
            }
        }

        start := time.Now()
        outcome := ras.attemptTaskProcessing(ctx, task, workerName, attempt)
        attemptDuration := time.Since(start)
        totalDuration += attemptDuration

        if outcome.Success {
            ras.circuitBreaker.RecordSuccess()
            outcome.Duration = totalDuration
            outcome.Attempts = attempt
            outcome.RecoveryUsed = attempt > 1
            return outcome
        }

        // Record failure
        lastError = outcome.Error
        lastFailureType = outcome.FailureType
        ras.failureDetector.RecordFailure(outcome.FailureType)
        ras.circuitBreaker.RecordFailure()

        // Check if we should retry
        if attempt <= task.MaxRetries && ras.retryPolicy.ShouldRetry(outcome.FailureType, attempt) {
            delay := ras.retryPolicy.GetDelay(attempt)
            fmt.Printf("🔥”„ %s: Retrying %s (attempt %d/%d) after %v\n",
                workerName, task.ID, attempt+1, task.MaxRetries+1, delay)

            select {
            case <-time.After(delay):
                continue
            case <-ctx.Done():
                break
            }
        } else {
            break
        }
    }

    // All retries exhausted
    return TaskOutcome{
        TaskID:      task.ID,
        Success:     false,
        Error:       lastError,
        Duration:    totalDuration,
        ProcessedBy: workerName,
        Attempts:    task.MaxRetries + 1,
        FailureType: lastFailureType,
    }
}

// attemptTaskProcessing performs a single task processing attempt
func (ras *ResilientAutomationService) attemptTaskProcessing(ctx context.Context,
                                                            task FailureDetectionTask,
                                                            workerName string,
                                                            attempt int) TaskOutcome {
    taskCtx, cancel := context.WithTimeout(ctx, task.Timeout)
    defer cancel()

    // Simulate different failure modes based on task type
    var processingTime time.Duration
    var failureRate float32
    var failureType FailureType

    switch task.Type {
    case "reliable":
        processingTime = time.Duration(rand.Intn(200)+100) * time.Millisecond
        failureRate = 0.1 // 10% failure rate
        failureType = ProcessingFailure

    case "flaky":
        processingTime = time.Duration(rand.Intn(300)+200) * time.Millisecond
        failureRate = 0.4 // 40% failure rate
        failureType = NetworkFailure

    case "slow":
        processingTime = time.Duration(rand.Intn(2000)+1000) * time.Millisecond
        failureRate = 0.2 // 20% failure rate
        failureType = TimeoutFailure

    case "timeout":
        processingTime = time.Duration(rand.Intn(1000)+800) * time.Millisecond
        failureRate = 0.6 // 60% failure rate (often exceeds timeout)
        failureType = TimeoutFailure

    default:
        processingTime = time.Duration(rand.Intn(500)+250) * time.Millisecond
        failureRate = 0.3
        failureType = ProcessingFailure
    }

    select {
    case <-time.After(processingTime):
        // Simulate random failures
        if rand.Float32() < failureRate {
            var err error
            switch failureType {
            case NetworkFailure:
                err = fmt.Errorf("network connection failed")
            case TimeoutFailure:
                err = fmt.Errorf("operation timed out")
            case ProcessingFailure:
                err = fmt.Errorf("processing error occurred")
            default:
                err = fmt.Errorf("unknown error")
            }

            return TaskOutcome{
                TaskID:      task.ID,
                Success:     false,
                Error:       err,
                ProcessedBy: workerName,
                FailureType: failureType,
            }
        }

        // Success
        return TaskOutcome{
            TaskID:      task.ID,
            Success:     true,
            Output:      fmt.Sprintf("processed-%s-attempt-%d", task.Data, attempt),
            ProcessedBy: workerName,
            FailureType: NoFailure,
            CompletedAt: time.Now(),
        }

    case <-taskCtx.Done():
        return TaskOutcome{
            TaskID:      task.ID,
            Success:     false,
            Error:       taskCtx.Err(),
            ProcessedBy: workerName,
            FailureType: TimeoutFailure,
        }
    }
}

func main() {
    fmt.Printf("Starting with GOMAXPROCS=%d\n", runtime.GOMAXPROCS(0))
    fmt.Println("=== Failure Detection and Recovery Patterns Demo ===")

    // Demonstrate failure detection and recovery
    demonstrateFailureDetectionAndRecovery()

    fmt.Println("\n=== Failure Detection and Recovery Key Insights ===")
    fmt.Println("✅ FAILURE DETECTION: Categorize and track different failure types")
    fmt.Println("✅ CIRCUIT BREAKER: Prevent cascading failures in distributed systems")
    fmt.Println("✅ RETRY POLICY: Implement intelligent retry with exponential backoff")
    fmt.Println("✅ RECOVERY MECHANISMS: Automatically recover from transient failures")
    fmt.Println("✅ RESILIENCE PATTERNS: Build fault-tolerant automation systems")
    fmt.Println("✅ MONITORING: Track failure rates and recovery effectiveness")
}
```

**Prerequisites**: Module 30
