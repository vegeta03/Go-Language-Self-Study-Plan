# Module 28: Scheduler Mechanics and Goroutines

**Duration**: 1 hour (45 min theory + 15 min hands-on)

**Learning Objectives**:

- Understand OS scheduler vs Go scheduler mechanics
- Master goroutine creation and lifecycle management
- Learn about M:N threading model and work stealing
- Apply concurrent patterns in automation systems
- Design efficient concurrent automation workflows

**Videos Covered**:

- 8.1 Topics (0:00:29)
- 8.2 OS Scheduler Mechanics (0:28:59)
- 8.3 Go Scheduler Mechanics (0:20:41)
- 8.4 Creating Goroutines (0:19:43)

**Key Concepts**:

- OS threads vs goroutines: lightweight concurrency
- M:N threading model: mapping goroutines to OS threads
- Go scheduler: work stealing and cooperative scheduling
- Goroutine creation cost and lifecycle
- Context switching and stack management
- Concurrency vs parallelism

**Hands-on Exercise 1: OS Scheduler vs Go Scheduler Mechanics**:

```go
// Demonstrating OS scheduler vs Go scheduler mechanics in automation systems
package main

import (
    "fmt"
    "runtime"
    "sync"
    "time"
)

// PART 1: Understanding OS Scheduler Mechanics
// OS threads are expensive: 1MB stack, expensive context switches
// OS scheduler is preemptive and non-deterministic

// OSThreadSimulation demonstrates OS thread behavior
type OSThreadSimulation struct {
    ThreadID    int
    StackSize   int // In KB for simulation
    State       string // "running", "runnable", "waiting"
    ContextSwitches int
}

// NewOSThread creates a simulated OS thread
func NewOSThread(id int) *OSThreadSimulation {
    return &OSThreadSimulation{
        ThreadID:  id,
        StackSize: 1024, // 1MB stack
        State:     "runnable",
        ContextSwitches: 0,
    }
}

// Simulate expensive context switch
func (t *OSThreadSimulation) ContextSwitch() {
    t.ContextSwitches++
    // Simulate context switch overhead
    time.Sleep(10 * time.Microsecond) // Expensive!
    fmt.Printf("OS Thread %d: Context switch #%d (expensive!)\n",
        t.ThreadID, t.ContextSwitches)
}

// PART 2: Understanding Go Scheduler Mechanics
// Goroutines are lightweight: 2KB initial stack, cheap context switches
// Go scheduler is cooperative but feels preemptive

// GoSchedulerDemo demonstrates Go scheduler components
type GoSchedulerDemo struct {
    LogicalProcessors int // P - one per CPU core
    OSThreads        int // M - OS threads managed by Go
    GlobalRunQueue   []int // Global goroutine queue
    LocalRunQueues   [][]int // Local run queues per P
}

// NewGoScheduler creates a Go scheduler simulation
func NewGoScheduler() *GoSchedulerDemo {
    numCPU := runtime.NumCPU()
    return &GoSchedulerDemo{
        LogicalProcessors: numCPU,
        OSThreads:        numCPU, // Usually 1:1 with P
        GlobalRunQueue:   make([]int, 0),
        LocalRunQueues:   make([][]int, numCPU),
    }
}

// ShowSchedulerState displays current scheduler state
func (gs *GoSchedulerDemo) ShowSchedulerState() {
    fmt.Printf("=== Go Scheduler State ===\n")
    fmt.Printf("Logical Processors (P): %d\n", gs.LogicalProcessors)
    fmt.Printf("OS Threads (M): %d\n", gs.OSThreads)
    fmt.Printf("GOMAXPROCS: %d\n", runtime.GOMAXPROCS(0))
    fmt.Printf("NumCPU: %d\n", runtime.NumCPU())
    fmt.Printf("Current Goroutines: %d\n", runtime.NumGoroutine())

    // Show memory stats
    var m runtime.MemStats
    runtime.ReadMemStats(&m)
    fmt.Printf("Heap Objects: %d\n", m.HeapObjects)
    fmt.Printf("Stack In Use: %d KB\n", m.StackInuse/1024)
}

// PART 3: Comparing OS Threads vs Goroutines
func demonstrateSchedulerComparison() {
    fmt.Println("=== OS Scheduler vs Go Scheduler Comparison ===")

    // Show initial state
    scheduler := NewGoScheduler()
    scheduler.ShowSchedulerState()

    fmt.Println("\n--- OS Thread Characteristics ---")
    osThread := NewOSThread(1)
    fmt.Printf("OS Thread Stack Size: %d KB\n", osThread.StackSize)
    fmt.Printf("Context Switch Cost: Expensive (microseconds)\n")
    fmt.Printf("Creation Cost: Expensive (system call)\n")
    fmt.Printf("Scheduler Type: Preemptive (kernel mode)\n")

    // Simulate expensive OS context switches
    fmt.Println("\nSimulating OS thread context switches:")
    start := time.Now()
    for i := 0; i < 5; i++ {
        osThread.ContextSwitch()
    }
    osContextTime := time.Since(start)

    fmt.Println("\n--- Goroutine Characteristics ---")
    fmt.Printf("Goroutine Initial Stack: 2 KB (grows as needed)\n")
    fmt.Printf("Context Switch Cost: Cheap (nanoseconds)\n")
    fmt.Printf("Creation Cost: Cheap (user space)\n")
    fmt.Printf("Scheduler Type: Cooperative (user mode)\n")

    // Demonstrate cheap goroutine creation
    fmt.Println("\nDemonstrating goroutine creation cost:")
    start = time.Now()
    var wg sync.WaitGroup

    // Create many goroutines quickly
    for i := 0; i < 1000; i++ {
        wg.Add(1)
        go func(id int) {
            defer wg.Done()
            // Minimal work to show creation speed
            _ = id * 2
        }(i)
    }

    wg.Wait()
    goroutineTime := time.Since(start)

    fmt.Printf("Created 1000 goroutines in: %v\n", goroutineTime)
    fmt.Printf("OS context switches took: %v\n", osContextTime)
    fmt.Printf("Goroutines are ~%dx faster to create\n",
        osContextTime.Nanoseconds()/goroutineTime.Nanoseconds())
}

// PART 4: Automation System Scheduler Analysis
type AutomationSchedulerAnalyzer struct {
    TaskCount       int
    GoroutineCount  int
    StartTime       time.Time
}

func NewAutomationAnalyzer() *AutomationSchedulerAnalyzer {
    return &AutomationSchedulerAnalyzer{
        StartTime: time.Now(),
    }
}

func (asa *AutomationSchedulerAnalyzer) AnalyzeSchedulerBehavior() {
    fmt.Println("\n=== Automation System Scheduler Analysis ===")

    // Show scheduler mechanics during automation tasks
    fmt.Printf("Initial goroutines: %d\n", runtime.NumGoroutine())

    // Simulate different types of automation tasks
    var wg sync.WaitGroup

    // CPU-bound automation task
    wg.Add(1)
    go func() {
        defer wg.Done()
        fmt.Println("CPU-bound task: Data processing")
        // Simulate CPU work
        for i := 0; i < 1000000; i++ {
            _ = i * i
        }
        fmt.Println("CPU-bound task completed")
    }()

    // I/O-bound automation task
    wg.Add(1)
    go func() {
        defer wg.Done()
        fmt.Println("I/O-bound task: File operations")
        // Simulate I/O wait
        time.Sleep(100 * time.Millisecond)
        fmt.Println("I/O-bound task completed")
    }()

    // Network-bound automation task
    wg.Add(1)
    go func() {
        defer wg.Done()
        fmt.Println("Network-bound task: API calls")
        // Simulate network wait
        time.Sleep(150 * time.Millisecond)
        fmt.Println("Network-bound task completed")
    }()

    fmt.Printf("Peak goroutines: %d\n", runtime.NumGoroutine())

    wg.Wait()

    fmt.Printf("Final goroutines: %d\n", runtime.NumGoroutine())
    fmt.Printf("Total analysis time: %v\n", time.Since(asa.StartTime))
}

func main() {
    fmt.Println("=== Scheduler Mechanics Demonstration ===")

    // Part 1: Compare OS threads vs Goroutines
    demonstrateSchedulerComparison()

    // Part 2: Analyze scheduler behavior in automation context
    analyzer := NewAutomationAnalyzer()
    analyzer.AnalyzeSchedulerBehavior()

    fmt.Println("\n=== Key Scheduler Insights ===")
    fmt.Println("✅… OS THREADS: Expensive, preemptive, 1MB stack")
    fmt.Println("✅… GOROUTINES: Cheap, cooperative, 2KB initial stack")
    fmt.Println("✅… GO SCHEDULER: M:N model, work-stealing, user-space")
    fmt.Println("✅… GOMAXPROCS: Controls logical processors (P)")
    fmt.Println("✅… CONTEXT SWITCHING: Much cheaper in Go scheduler")
    fmt.Println("✅… SCALABILITY: Can handle thousands of goroutines easily")
}
```

**Hands-on Exercise 2: Goroutine Creation and Lifecycle Management**:

```go
// Demonstrating goroutine creation, lifecycle, and management in automation systems
package main

import (
    "context"
    "fmt"
    "runtime"
    "sync"
    "time"
)

// PART 1: Goroutine Lifecycle States
// Goroutines have three states: running, runnable, waiting (same as OS threads)

// AutomationTask represents work in an automation system
type AutomationTask struct {
    ID          int
    Name        string
    Type        string // "cpu-bound", "io-bound", "network-bound"
    Duration    time.Duration
    Status      string // "pending", "running", "completed", "failed"
    Result      string
    StartTime   time.Time
    EndTime     time.Time
}

// TaskProcessor manages task execution
type TaskProcessor struct {
    Name         string
    WorkerCount  int
    ProcessedTasks int
    mu           sync.Mutex
}

// NewTaskProcessor creates a new task processor
func NewTaskProcessor(name string, workers int) *TaskProcessor {
    return &TaskProcessor{
        Name:        name,
        WorkerCount: workers,
    }
}

// PART 2: Proper Goroutine Creation Patterns
// Rule: Never create a goroutine unless you know how and when it will terminate

// ProcessTask demonstrates proper goroutine lifecycle management
func (tp *TaskProcessor) ProcessTask(task *AutomationTask, wg *sync.WaitGroup) {
    // CRITICAL: Always defer wg.Done() to ensure proper cleanup
    defer wg.Done()

    // Update task status
    task.Status = "running"
    task.StartTime = time.Now()

    fmt.Printf("Goroutine %d: Starting task %d (%s)\n",
        getGoroutineID(), task.ID, task.Name)

    // Simulate different types of work
    switch task.Type {
    case "cpu-bound":
        tp.processCPUBoundTask(task)
    case "io-bound":
        tp.processIOBoundTask(task)
    case "network-bound":
        tp.processNetworkBoundTask(task)
    default:
        tp.processGenericTask(task)
    }

    // Complete task
    task.Status = "completed"
    task.EndTime = time.Now()
    task.Result = fmt.Sprintf("Task %d completed by goroutine %d",
        task.ID, getGoroutineID())

    // Update processor stats
    tp.mu.Lock()
    tp.ProcessedTasks++
    tp.mu.Unlock()

    fmt.Printf("Goroutine %d: Completed task %d in %v\n",
        getGoroutineID(), task.ID, task.EndTime.Sub(task.StartTime))
}

// Different task processing methods
func (tp *TaskProcessor) processCPUBoundTask(task *AutomationTask) {
    // CPU-intensive work - scheduler will preempt at function calls
    for i := 0; i < 1000000; i++ {
        _ = i * i
        // Yield to scheduler occasionally for cooperative behavior
        if i%100000 == 0 {
            runtime.Gosched() // Explicit yield
        }
    }
}

func (tp *TaskProcessor) processIOBoundTask(task *AutomationTask) {
    // I/O-bound work - goroutine will be in waiting state
    time.Sleep(task.Duration) // Simulates file I/O, database operations
}

func (tp *TaskProcessor) processNetworkBoundTask(task *AutomationTask) {
    // Network-bound work - goroutine will be in waiting state
    time.Sleep(task.Duration) // Simulates HTTP requests, API calls
}

func (tp *TaskProcessor) processGenericTask(task *AutomationTask) {
    // Generic processing
    time.Sleep(task.Duration)
}

// PART 3: Goroutine Management Patterns

// AutomationWorkflow demonstrates proper goroutine management
type AutomationWorkflow struct {
    Name           string
    Tasks          []*AutomationTask
    MaxConcurrency int
    Processor      *TaskProcessor
}

// NewAutomationWorkflow creates a new workflow
func NewAutomationWorkflow(name string, maxConcurrency int) *AutomationWorkflow {
    return &AutomationWorkflow{
        Name:           name,
        MaxConcurrency: maxConcurrency,
        Processor:      NewTaskProcessor("WorkflowProcessor", maxConcurrency),
    }
}

// AddTask adds a task to the workflow
func (aw *AutomationWorkflow) AddTask(id int, name, taskType string, duration time.Duration) {
    task := &AutomationTask{
        ID:       id,
        Name:     name,
        Type:     taskType,
        Duration: duration,
        Status:   "pending",
    }
    aw.Tasks = append(aw.Tasks, task)
}

// ExecuteWorkflow demonstrates proper goroutine lifecycle management
func (aw *AutomationWorkflow) ExecuteWorkflow() {
    fmt.Printf("\n=== Executing Workflow: %s ===\n", aw.Name)
    fmt.Printf("Tasks: %d, Max Concurrency: %d\n", len(aw.Tasks), aw.MaxConcurrency)
    fmt.Printf("Initial goroutines: %d\n", runtime.NumGoroutine())

    start := time.Now()

    // PATTERN 1: WaitGroup for orchestration
    var wg sync.WaitGroup

    // PATTERN 2: Semaphore for concurrency control
    semaphore := make(chan struct{}, aw.MaxConcurrency)

    // Launch goroutines for each task
    for _, task := range aw.Tasks {
        // Acquire semaphore slot
        semaphore <- struct{}{}

        // Add to wait group BEFORE creating goroutine
        wg.Add(1)

        // Create goroutine with proper cleanup
        go func(t *AutomationTask) {
            defer func() {
                // Release semaphore slot
                <-semaphore
            }()

            // Process the task
            aw.Processor.ProcessTask(t, &wg)
        }(task) // Pass task by value to avoid closure issues
    }

    fmt.Printf("Peak goroutines: %d\n", runtime.NumGoroutine())

    // Wait for all goroutines to complete
    wg.Wait()

    duration := time.Since(start)

    fmt.Printf("Final goroutines: %d\n", runtime.NumGoroutine())
    fmt.Printf("Workflow completed in: %v\n", duration)

    // Report results
    aw.reportResults()
}

// reportResults shows workflow execution results
func (aw *AutomationWorkflow) reportResults() {
    fmt.Println("\n--- Workflow Results ---")

    completed := 0
    failed := 0
    totalDuration := time.Duration(0)

    for _, task := range aw.Tasks {
        fmt.Printf("Task %d (%s): %s - %v\n",
            task.ID, task.Type, task.Status,
            task.EndTime.Sub(task.StartTime))

        if task.Status == "completed" {
            completed++
            totalDuration += task.EndTime.Sub(task.StartTime)
        } else {
            failed++
        }
    }

    fmt.Printf("\nSummary: %d completed, %d failed\n", completed, failed)
    fmt.Printf("Average task duration: %v\n", totalDuration/time.Duration(completed))
    fmt.Printf("Processor handled: %d tasks\n", aw.Processor.ProcessedTasks)
}

// PART 4: Context-based Goroutine Cancellation
func demonstrateGoroutineCancellation() {
    fmt.Println("\n=== Goroutine Cancellation Demo ===")

    ctx, cancel := context.WithTimeout(context.Background(), 2*time.Second)
    defer cancel()

    var wg sync.WaitGroup

    // Start long-running goroutines
    for i := 0; i < 5; i++ {
        wg.Add(1)
        go func(id int) {
            defer wg.Done()

            select {
            case <-ctx.Done():
                fmt.Printf("Goroutine %d: Cancelled (%v)\n", id, ctx.Err())
                return
            case <-time.After(5 * time.Second):
                fmt.Printf("Goroutine %d: Completed normally\n", id)
            }
        }(i)
    }

    wg.Wait()
    fmt.Println("All goroutines finished (cancelled or completed)")
}

// Utility function to get goroutine ID (for demonstration only)
func getGoroutineID() int {
    // In real code, avoid getting goroutine IDs
    // This is for educational purposes only
    return int(time.Now().UnixNano() % 1000)
}

func main() {
    fmt.Println("=== Goroutine Lifecycle Management Demo ===")

    // Create automation workflow
    workflow := NewAutomationWorkflow("DataProcessingWorkflow", 3)

    // Add various types of tasks
    workflow.AddTask(1, "Parse CSV", "io-bound", 100*time.Millisecond)
    workflow.AddTask(2, "Calculate Stats", "cpu-bound", 50*time.Millisecond)
    workflow.AddTask(3, "Send Notification", "network-bound", 150*time.Millisecond)
    workflow.AddTask(4, "Update Database", "io-bound", 120*time.Millisecond)
    workflow.AddTask(5, "Generate Report", "cpu-bound", 80*time.Millisecond)
    workflow.AddTask(6, "Upload to Cloud", "network-bound", 200*time.Millisecond)

    // Execute workflow
    workflow.ExecuteWorkflow()

    // Demonstrate cancellation
    demonstrateGoroutineCancellation()

    fmt.Println("\n=== Goroutine Management Best Practices ===")
    fmt.Println("✅… CREATION: Only create goroutines when you know how/when they terminate")
    fmt.Println("✅… ORCHESTRATION: Use sync.WaitGroup to wait for completion")
    fmt.Println("✅… CONCURRENCY CONTROL: Use semaphores to limit concurrent goroutines")
    fmt.Println("✅… CANCELLATION: Use context.Context for graceful cancellation")
    fmt.Println("✅… CLEANUP: Always defer cleanup operations (wg.Done, resource release)")
    fmt.Println("✅… CLOSURE SAFETY: Pass values to goroutines to avoid closure bugs")
    fmt.Println("✅… COOPERATIVE: Use runtime.Gosched() in CPU-intensive loops")
}
```

**Hands-on Exercise 3: M:N Threading Model and Work Stealing**:

```go
// Demonstrating work stealing scheduler and concurrent patterns in automation systems
package main

import (
    "fmt"
    "math/rand"
    "runtime"
    "sync"
    "sync/atomic"
    "time"
)

// PART 1: Work Stealing Scheduler Demonstration
// Go scheduler uses work stealing to balance load across logical processors

// WorkStealingDemo simulates the Go scheduler's work stealing behavior
type WorkStealingDemo struct {
    LogicalProcessors int
    LocalQueues      [][]int // Simulated local run queues per P
    GlobalQueue      []int   // Simulated global run queue
    WorkStolen       int64   // Counter for stolen work
    mu               sync.Mutex
}

// NewWorkStealingDemo creates a work stealing demonstration
func NewWorkStealingDemo() *WorkStealingDemo {
    numP := runtime.GOMAXPROCS(0)
    return &WorkStealingDemo{
        LogicalProcessors: numP,
        LocalQueues:      make([][]int, numP),
        GlobalQueue:      make([]int, 0),
    }
}

// SimulateWorkStealing demonstrates how work stealing balances load
func (wsd *WorkStealingDemo) SimulateWorkStealing() {
    fmt.Println("=== Work Stealing Scheduler Simulation ===")
    fmt.Printf("Logical Processors (P): %d\n", wsd.LogicalProcessors)

    // Create unbalanced work distribution
    wsd.mu.Lock()
    // P0 gets lots of work
    wsd.LocalQueues[0] = []int{1, 2, 3, 4, 5, 6, 7, 8, 9, 10}
    // Other Ps get little or no work
    for i := 1; i < wsd.LogicalProcessors; i++ {
        wsd.LocalQueues[i] = []int{}
    }
    wsd.mu.Unlock()

    fmt.Println("Initial work distribution:")
    wsd.showQueueState()

    var wg sync.WaitGroup

    // Start workers on each logical processor
    for p := 0; p < wsd.LogicalProcessors; p++ {
        wg.Add(1)
        go wsd.worker(p, &wg)
    }

    wg.Wait()

    fmt.Printf("Work stealing events: %d\n", atomic.LoadInt64(&wsd.WorkStolen))
    fmt.Println("Final work distribution:")
    wsd.showQueueState()
}

// worker simulates a logical processor worker
func (wsd *WorkStealingDemo) worker(processorID int, wg *sync.WaitGroup) {
    defer wg.Done()

    for {
        // Try to get work from local queue first
        work := wsd.getLocalWork(processorID)
        if work == -1 {
            // No local work, try to steal from other processors
            work = wsd.stealWork(processorID)
            if work == -1 {
                // No work available anywhere
                break
            }
            atomic.AddInt64(&wsd.WorkStolen, 1)
            fmt.Printf("P%d: Stole work %d\n", processorID, work)
        }

        // Process the work
        wsd.processWork(processorID, work)
    }
}

// getLocalWork gets work from processor's local queue
func (wsd *WorkStealingDemo) getLocalWork(processorID int) int {
    wsd.mu.Lock()
    defer wsd.mu.Unlock()

    if len(wsd.LocalQueues[processorID]) > 0 {
        work := wsd.LocalQueues[processorID][0]
        wsd.LocalQueues[processorID] = wsd.LocalQueues[processorID][1:]
        return work
    }
    return -1
}

// stealWork attempts to steal work from other processors
func (wsd *WorkStealingDemo) stealWork(processorID int) int {
    wsd.mu.Lock()
    defer wsd.mu.Unlock()

    // Try to steal from other processors
    for p := 0; p < wsd.LogicalProcessors; p++ {
        if p != processorID && len(wsd.LocalQueues[p]) > 1 {
            // Steal from the end of the queue (LIFO for stealing)
            work := wsd.LocalQueues[p][len(wsd.LocalQueues[p])-1]
            wsd.LocalQueues[p] = wsd.LocalQueues[p][:len(wsd.LocalQueues[p])-1]
            return work
        }
    }

    // Try global queue
    if len(wsd.GlobalQueue) > 0 {
        work := wsd.GlobalQueue[0]
        wsd.GlobalQueue = wsd.GlobalQueue[1:]
        return work
    }

    return -1
}

// processWork simulates processing work
func (wsd *WorkStealingDemo) processWork(processorID, work int) {
    fmt.Printf("P%d: Processing work %d\n", processorID, work)
    time.Sleep(10 * time.Millisecond) // Simulate work
}

// showQueueState displays current queue state
func (wsd *WorkStealingDemo) showQueueState() {
    wsd.mu.Lock()
    defer wsd.mu.Unlock()

    for i, queue := range wsd.LocalQueues {
        fmt.Printf("  P%d local queue: %v\n", i, queue)
    }
    fmt.Printf("  Global queue: %v\n", wsd.GlobalQueue)
}

// PART 2: Concurrent Patterns for Automation Systems

// AutomationPipeline demonstrates pipeline pattern with work stealing
type AutomationPipeline struct {
    Name        string
    Stages      []string
    WorkerCount int
    Metrics     *PipelineMetrics
}

// PipelineMetrics tracks pipeline performance
type PipelineMetrics struct {
    ItemsProcessed int64
    StagesExecuted int64
    WorkersActive  int64
    StartTime      time.Time
}

// NewAutomationPipeline creates a new pipeline
func NewAutomationPipeline(name string, stages []string, workers int) *AutomationPipeline {
    return &AutomationPipeline{
        Name:        name,
        Stages:      stages,
        WorkerCount: workers,
        Metrics: &PipelineMetrics{
            StartTime: time.Now(),
        },
    }
}

// ProcessItems processes items through the pipeline
func (ap *AutomationPipeline) ProcessItems(items []string) {
    fmt.Printf("\n=== Pipeline: %s ===\n", ap.Name)
    fmt.Printf("Items: %d, Workers: %d, Stages: %v\n",
        len(items), ap.WorkerCount, ap.Stages)

    // Create work channels
    input := make(chan string, len(items))
    output := make(chan string, len(items))

    // Fill input channel
    for _, item := range items {
        input <- item
    }
    close(input)

    var wg sync.WaitGroup

    // Start worker goroutines (work stealing happens automatically)
    for i := 0; i < ap.WorkerCount; i++ {
        wg.Add(1)
        go ap.pipelineWorker(i, input, output, &wg)
    }

    // Close output when all workers done
    go func() {
        wg.Wait()
        close(output)
    }()

    // Collect results
    var results []string
    for result := range output {
        results = append(results, result)
        atomic.AddInt64(&ap.Metrics.ItemsProcessed, 1)
    }

    ap.showMetrics()

    fmt.Printf("Processed %d items successfully\n", len(results))
    for _, result := range results {
        fmt.Printf("  %s\n", result)
    }
}

// pipelineWorker processes items through pipeline stages
func (ap *AutomationPipeline) pipelineWorker(workerID int, input <-chan string, output chan<- string, wg *sync.WaitGroup) {
    defer wg.Done()

    atomic.AddInt64(&ap.Metrics.WorkersActive, 1)
    defer atomic.AddInt64(&ap.Metrics.WorkersActive, -1)

    for item := range input {
        result := item

        // Process through all stages
        for _, stage := range ap.Stages {
            result = ap.processStage(stage, result, workerID)
            atomic.AddInt64(&ap.Metrics.StagesExecuted, 1)

            // Yield to scheduler to allow work stealing
            runtime.Gosched()
        }

        output <- result
    }
}

// processStage processes an item through a pipeline stage
func (ap *AutomationPipeline) processStage(stage, item string, workerID int) string {
    // Simulate variable processing time to trigger work stealing
    processingTime := time.Duration(rand.Intn(100)) * time.Millisecond
    time.Sleep(processingTime)

    return fmt.Sprintf("%s→%s(W%d)", item, stage, workerID)
}

// showMetrics displays pipeline performance metrics
func (ap *AutomationPipeline) showMetrics() {
    duration := time.Since(ap.Metrics.StartTime)
    fmt.Printf("\nPipeline Metrics:\n")
    fmt.Printf("  Duration: %v\n", duration)
    fmt.Printf("  Items processed: %d\n", atomic.LoadInt64(&ap.Metrics.ItemsProcessed))
    fmt.Printf("  Stages executed: %d\n", atomic.LoadInt64(&ap.Metrics.StagesExecuted))
    fmt.Printf("  Peak workers: %d\n", ap.WorkerCount)
}

// PART 3: Fan-Out/Fan-In Pattern
func demonstrateFanOutFanIn() {
    fmt.Println("\n=== Fan-Out/Fan-In Pattern ===")

    // Input data
    jobs := []int{1, 2, 3, 4, 5, 6, 7, 8, 9, 10}

    // Fan-out: distribute work to multiple goroutines
    input := make(chan int, len(jobs))
    for _, job := range jobs {
        input <- job
    }
    close(input)

    // Create multiple worker channels
    numWorkers := 3
    workerOutputs := make([]<-chan int, numWorkers)

    for i := 0; i < numWorkers; i++ {
        output := make(chan int)
        workerOutputs[i] = output

        go func(workerID int, input <-chan int, output chan<- int) {
            defer close(output)

            for job := range input {
                // Simulate work
                result := job * job
                time.Sleep(time.Duration(rand.Intn(100)) * time.Millisecond)
                fmt.Printf("Worker %d: %d → %d\n", workerID, job, result)
                output <- result
            }
        }(i, input, output)
    }

    // Fan-in: merge results from multiple goroutines
    results := fanIn(workerOutputs...)

    fmt.Println("Results:")
    for result := range results {
        fmt.Printf("  %d\n", result)
    }
}

// fanIn merges multiple channels into one
func fanIn(inputs ...<-chan int) <-chan int {
    output := make(chan int)
    var wg sync.WaitGroup

    // Start a goroutine for each input channel
    for _, input := range inputs {
        wg.Add(1)
        go func(ch <-chan int) {
            defer wg.Done()
            for value := range ch {
                output <- value
            }
        }(input)
    }

    // Close output when all inputs are done
    go func() {
        wg.Wait()
        close(output)
    }()

    return output
}

func main() {
    fmt.Println("=== Work Stealing and Concurrent Patterns Demo ===")

    // Part 1: Demonstrate work stealing
    workStealingDemo := NewWorkStealingDemo()
    workStealingDemo.SimulateWorkStealing()

    // Part 2: Pipeline pattern with work stealing
    pipeline := NewAutomationPipeline(
        "DataProcessingPipeline",
        []string{"validate", "transform", "enrich", "store"},
        4,
    )

    items := []string{"data1", "data2", "data3", "data4", "data5", "data6", "data7", "data8"}
    pipeline.ProcessItems(items)

    // Part 3: Fan-out/Fan-in pattern
    demonstrateFanOutFanIn()

    fmt.Println("\n=== Work Stealing and Concurrency Insights ===")
    fmt.Println("✅… WORK STEALING: Automatic load balancing across logical processors")
    fmt.Println("✅… LOCAL QUEUES: Each P has its own queue for better cache locality")
    fmt.Println("✅… GLOBAL QUEUE: Fallback when local queues are empty")
    fmt.Println("✅… LIFO STEALING: Steal from end of queue to preserve cache locality")
    fmt.Println("✅… PIPELINE PATTERN: Efficient for multi-stage processing")
    fmt.Println("✅… FAN-OUT/FAN-IN: Distribute work and merge results")
    fmt.Println("✅… RUNTIME.GOSCHED(): Explicit yield for cooperative scheduling")
}
```

**Prerequisites**: Module 27
