# Module 30: Channels and Signaling Semantics

**Duration**: 1 hour (45 min theory + 15 min hands-on)

**Learning Objectives**:

- Master channel creation, operations, and signaling semantics
- Understand buffered vs unbuffered channels
- Learn channel patterns: fan-out, fan-in, worker pools
- Apply channel-based communication in automation systems
- Design robust concurrent automation pipelines

**Videos Covered**:

- 10.1 Topics (0:00:43)
- 10.2 Signaling Semantics (0:17:50)
- 10.3 Basic Patterns Part 1 (0:11:12)
- 10.4 Basic Patterns Part 2 (0:04:19)
- 10.5 Basic Patterns Part 3 (0:05:59)
- 10.6 Pooling Pattern (0:06:23)
- 10.7 Fan Out Pattern Part 1 (0:08:37)
- 10.8 Fan Out Pattern Part 2 (0:06:24)

**Key Concepts**:

- Channels: Go's communication mechanism
- Signaling semantics: guarantee vs no guarantee
- Buffered vs unbuffered channels
- Channel operations: send, receive, close
- Channel patterns: pooling, fan-out, fan-in
- Select statement for non-blocking operations
- Channel-based synchronization

**Hands-on Exercise 1: Channel Signaling Semantics and Basic Operations**:

Understanding channel creation, signaling semantics, and basic communication patterns:

```go
// Channel signaling semantics and basic operations in automation systems
package main

import (
    "fmt"
    "runtime"
    "sync"
    "time"
)

// PART 1: Channel Signaling Semantics
// Channels provide signaling semantics: guarantee vs no guarantee of delivery

// AutomationTask represents work in an automation system
type AutomationTask struct {
    ID          string
    Type        string
    Data        string
    Priority    int
    CreatedAt   time.Time
}

// TaskResult represents the outcome of task processing
type TaskResult struct {
    TaskID      string
    Success     bool
    Output      string
    Duration    time.Duration
    ProcessedBy string
    CompletedAt time.Time
}

// PART 2: Unbuffered Channels - Guarantee Semantics
// Unbuffered channels provide guarantee: sender knows receiver got the data
func demonstrateUnbufferedChannels() {
    fmt.Println("=== Unbuffered Channels (Guarantee Semantics) ===")

    // Unbuffered channel - synchronous communication with guarantee
    taskChannel := make(chan AutomationTask)
    ackChannel := make(chan bool)

    // Receiver goroutine
    go func() {
        fmt.Println("Worker: Ready to receive tasks...")

        task := <-taskChannel // Blocks until sender sends
        fmt.Printf("Worker: Received task %s (type: %s)\n", task.ID, task.Type)

        // Simulate processing
        time.Sleep(100 * time.Millisecond)
        fmt.Printf("Worker: Completed task %s\n", task.ID)

        // Send acknowledgment
        ackChannel <- true
    }()

    // Sender
    task := AutomationTask{
        ID:        "task-001",
        Type:      "file_processing",
        Data:      "process_data.csv",
        Priority:  1,
        CreatedAt: time.Now(),
    }

    fmt.Println("Main: Sending task...")
    taskChannel <- task // Blocks until receiver receives
    fmt.Println("Main: Task sent (guaranteed received)")

    // Wait for acknowledgment
    <-ackChannel
    fmt.Println("Main: Task processing confirmed")
}

// PART 3: Buffered Channels - No Guarantee Semantics
// Buffered channels provide no guarantee: sender doesn't know if receiver got the data
func demonstrateBufferedChannels() {
    fmt.Println("\n=== Buffered Channels (No Guarantee Semantics) ===")

    // Buffered channel - asynchronous communication without guarantee
    taskQueue := make(chan AutomationTask, 3) // Buffer size 3

    // Send multiple tasks without blocking (no guarantee they're processed)
    tasks := []AutomationTask{
        {ID: "task-001", Type: "email", Data: "send_notification.json", Priority: 1, CreatedAt: time.Now()},
        {ID: "task-002", Type: "backup", Data: "backup_database.sql", Priority: 2, CreatedAt: time.Now()},
        {ID: "task-003", Type: "report", Data: "generate_report.xml", Priority: 3, CreatedAt: time.Now()},
    }

    fmt.Printf("Queue capacity: %d, current length: %d\n", cap(taskQueue), len(taskQueue))

    // Send tasks (non-blocking due to buffer)
    for _, task := range tasks {
        taskQueue <- task
        fmt.Printf("Queued task %s (no guarantee of processing)\n", task.ID)
    }

    fmt.Printf("After sending - Queue length: %d\n", len(taskQueue))

    // Process tasks
    go func() {
        for i := 0; i < len(tasks); i++ {
            task := <-taskQueue
            fmt.Printf("Processing task %s (type: %s)\n", task.ID, task.Type)
            time.Sleep(50 * time.Millisecond)
        }
    }()

    time.Sleep(200 * time.Millisecond) // Wait for processing
    fmt.Printf("Final queue length: %d\n", len(taskQueue))
}

// PART 4: Channel Operations and States
func demonstrateChannelOperations() {
    fmt.Println("\n=== Channel Operations and States ===")

    // Channel states: nil, open, closed
    var nilChannel chan string
    openChannel := make(chan string, 2)

    // Demonstrate channel operations
    fmt.Println("--- Channel Send/Receive Operations ---")

    // Send to open channel
    openChannel <- "message1"
    openChannel <- "message2"
    fmt.Printf("Sent 2 messages, channel length: %d\n", len(openChannel))

    // Receive from open channel
    msg1 := <-openChannel
    msg2 := <-openChannel
    fmt.Printf("Received: %s, %s\n", msg1, msg2)

    // Close channel
    close(openChannel)
    fmt.Println("Channel closed")

    // Receive from closed channel (returns zero value)
    msg3, ok := <-openChannel
    fmt.Printf("Receive from closed channel: '%s', ok=%t\n", msg3, ok)

    // Demonstrate nil channel behavior (would block forever)
    fmt.Println("--- Nil Channel Behavior ---")
    fmt.Printf("Nil channel: %v (operations would block forever)\n", nilChannel)

    // Use select with nil channel to show non-blocking behavior
    select {
    case <-nilChannel:
        fmt.Println("This will never execute")
    default:
        fmt.Println("Nil channel operations block, so default case executes")
    }
}

// PART 5: Channel Direction and Type Safety
func demonstrateChannelDirections() {
    fmt.Println("\n=== Channel Directions and Type Safety ===")

    // Bidirectional channel
    taskChan := make(chan AutomationTask, 2)

    // Start sender (send-only channel)
    go taskSender(taskChan)

    // Start receiver (receive-only channel)
    go taskReceiver(taskChan)

    time.Sleep(300 * time.Millisecond)
}

// taskSender accepts send-only channel
func taskSender(tasks chan<- AutomationTask) {
    defer close(tasks)

    for i := 1; i <= 3; i++ {
        task := AutomationTask{
            ID:        fmt.Sprintf("directed-task-%d", i),
            Type:      "processing",
            Data:      fmt.Sprintf("data-%d", i),
            Priority:  i,
            CreatedAt: time.Now(),
        }

        tasks <- task
        fmt.Printf("Sender: Sent task %s\n", task.ID)
        time.Sleep(50 * time.Millisecond)
    }

    fmt.Println("Sender: Finished sending tasks")
}

// taskReceiver accepts receive-only channel
func taskReceiver(tasks <-chan AutomationTask) {
    fmt.Println("Receiver: Starting to receive tasks...")

    for task := range tasks {
        fmt.Printf("Receiver: Processing task %s (type: %s)\n", task.ID, task.Type)
        time.Sleep(30 * time.Millisecond)
    }

    fmt.Println("Receiver: All tasks processed")
}

// PART 6: Select Statement for Non-blocking Operations
func demonstrateSelectStatement() {
    fmt.Println("\n=== Select Statement for Non-blocking Operations ===")

    ch1 := make(chan string, 1)
    ch2 := make(chan string, 1)
    timeout := time.After(500 * time.Millisecond)

    // Send data to channels at different times
    go func() {
        time.Sleep(100 * time.Millisecond)
        ch1 <- "Data from automation system 1"
    }()

    go func() {
        time.Sleep(200 * time.Millisecond)
        ch2 <- "Data from automation system 2"
    }()

    // Use select to handle multiple channels non-blockingly
    for i := 0; i < 4; i++ {
        select {
        case msg1 := <-ch1:
            fmt.Printf("Received from system 1: %s\n", msg1)
        case msg2 := <-ch2:
            fmt.Printf("Received from system 2: %s\n", msg2)
        case <-timeout:
            fmt.Println("Timeout: No more data received")
            return
        default:
            fmt.Printf("No channels ready (iteration %d), doing other work...\n", i+1)
            time.Sleep(75 * time.Millisecond)
        }
    }
}

func main() {
    fmt.Printf("Starting with GOMAXPROCS=%d\n", runtime.GOMAXPROCS(0))
    fmt.Println("=== Channel Signaling Semantics Demo ===")

    // Demonstrate unbuffered channels (guarantee semantics)
    demonstrateUnbufferedChannels()

    // Demonstrate buffered channels (no guarantee semantics)
    demonstrateBufferedChannels()

    // Demonstrate channel operations and states
    demonstrateChannelOperations()

    // Demonstrate channel directions
    demonstrateChannelDirections()

    // Demonstrate select statement
    demonstrateSelectStatement()

    fmt.Println("\n=== Channel Signaling Key Insights ===")
    fmt.Println("✅ UNBUFFERED: Guarantee semantics - sender knows receiver got data")
    fmt.Println("✅ BUFFERED: No guarantee semantics - sender doesn't know if data is processed")
    fmt.Println("✅ CHANNEL STATES: nil (blocks), open (normal), closed (returns zero value)")
    fmt.Println("✅ DIRECTIONS: Send-only (chan<-), receive-only (<-chan), bidirectional (chan)")
    fmt.Println("✅ SELECT: Non-blocking operations with multiple channels")
    fmt.Println("✅ CLOSE: Signals no more data, receivers can detect with ok value")
}
```

**Hands-on Exercise 2: Worker Pool and Pooling Patterns**:

Implementing worker pools and pooling patterns for concurrent task processing:

```go
// Worker pool and pooling patterns in automation systems
package main

import (
    "fmt"
    "math/rand"
    "runtime"
    "sync"
    "sync/atomic"
    "time"
)

// PART 1: Worker Pool Pattern
// Worker pools manage a fixed number of goroutines to process tasks

// Task represents work to be processed in automation system
type Task struct {
    ID          string
    Type        string
    Data        string
    Priority    int
    CreatedAt   time.Time
    Timeout     time.Duration
}

// Result represents the outcome of task processing
type Result struct {
    TaskID      string
    Success     bool
    Output      string
    Error       error
    Duration    time.Duration
    ProcessedBy int
    CompletedAt time.Time
}

// WorkerPool manages a pool of workers for task processing
type WorkerPool struct {
    Name         string
    WorkerCount  int
    TaskQueue    chan Task
    ResultQueue  chan Result
    workers      []Worker
    wg           sync.WaitGroup
    shutdown     chan struct{}

    // Metrics
    tasksProcessed int64
    tasksSucceeded int64
    tasksFailed    int64
}

// Worker represents a single worker in the pool
type Worker struct {
    ID       int
    Pool     *WorkerPool
    TaskChan chan Task
    quit     chan struct{}
}

// NewWorkerPool creates a new worker pool
func NewWorkerPool(name string, workerCount, queueSize int) *WorkerPool {
    return &WorkerPool{
        Name:        name,
        WorkerCount: workerCount,
        TaskQueue:   make(chan Task, queueSize),
        ResultQueue: make(chan Result, queueSize),
        workers:     make([]Worker, workerCount),
        shutdown:    make(chan struct{}),
    }
}

// Start initializes and starts all workers in the pool
func (wp *WorkerPool) Start() {
    fmt.Printf("Starting worker pool '%s' with %d workers\n", wp.Name, wp.WorkerCount)

    for i := 0; i < wp.WorkerCount; i++ {
        worker := Worker{
            ID:       i,
            Pool:     wp,
            TaskChan: make(chan Task),
            quit:     make(chan struct{}),
        }

        wp.workers[i] = worker
        wp.wg.Add(1)
        go worker.Start()
    }

    // Start task dispatcher
    go wp.dispatch()
}

// dispatch distributes tasks from the main queue to worker channels
func (wp *WorkerPool) dispatch() {
    for {
        select {
        case task := <-wp.TaskQueue:
            // Find available worker (round-robin)
            workerIndex := int(atomic.LoadInt64(&wp.tasksProcessed)) % wp.WorkerCount

            select {
            case wp.workers[workerIndex].TaskChan <- task:
                // Task dispatched successfully
            case <-wp.shutdown:
                return
            }

        case <-wp.shutdown:
            // Close all worker task channels
            for i := range wp.workers {
                close(wp.workers[i].TaskChan)
            }
            return
        }
    }
}

// Start begins the worker's task processing loop
func (w *Worker) Start() {
    defer w.Pool.wg.Done()

    fmt.Printf("Worker %d started\n", w.ID)

    for {
        select {
        case task, ok := <-w.TaskChan:
            if !ok {
                fmt.Printf("Worker %d shutting down\n", w.ID)
                return
            }

            w.processTask(task)

        case <-w.quit:
            fmt.Printf("Worker %d received quit signal\n", w.ID)
            return
        }
    }
}

// processTask handles individual task processing
func (w *Worker) processTask(task Task) {
    start := time.Now()

    fmt.Printf("Worker %d: Processing task %s (type: %s)\n", w.ID, task.ID, task.Type)

    // Simulate different types of processing
    var success bool
    var output string
    var err error

    switch task.Type {
    case "file_processing":
        success, output, err = w.processFile(task)
    case "email_sending":
        success, output, err = w.sendEmail(task)
    case "data_analysis":
        success, output, err = w.analyzeData(task)
    case "report_generation":
        success, output, err = w.generateReport(task)
    default:
        success, output, err = w.processGeneric(task)
    }

    duration := time.Since(start)

    // Update metrics
    atomic.AddInt64(&w.Pool.tasksProcessed, 1)
    if success {
        atomic.AddInt64(&w.Pool.tasksSucceeded, 1)
    } else {
        atomic.AddInt64(&w.Pool.tasksFailed, 1)
    }

    // Send result
    result := Result{
        TaskID:      task.ID,
        Success:     success,
        Output:      output,
        Error:       err,
        Duration:    duration,
        ProcessedBy: w.ID,
        CompletedAt: time.Now(),
    }

    select {
    case w.Pool.ResultQueue <- result:
        // Result sent successfully
    default:
        fmt.Printf("Warning: Result queue full, dropping result for task %s\n", task.ID)
    }
}

// Task processing methods for different types
func (w *Worker) processFile(task Task) (bool, string, error) {
    // Simulate file processing
    time.Sleep(time.Duration(rand.Intn(200)+100) * time.Millisecond)

    if rand.Float32() < 0.1 { // 10% failure rate
        return false, "", fmt.Errorf("file processing failed for %s", task.Data)
    }

    return true, fmt.Sprintf("File %s processed successfully", task.Data), nil
}

func (w *Worker) sendEmail(task Task) (bool, string, error) {
    // Simulate email sending
    time.Sleep(time.Duration(rand.Intn(150)+50) * time.Millisecond)

    if rand.Float32() < 0.05 { // 5% failure rate
        return false, "", fmt.Errorf("email sending failed for %s", task.Data)
    }

    return true, fmt.Sprintf("Email sent to %s", task.Data), nil
}

func (w *Worker) analyzeData(task Task) (bool, string, error) {
    // Simulate data analysis (CPU intensive)
    time.Sleep(time.Duration(rand.Intn(300)+200) * time.Millisecond)

    if rand.Float32() < 0.15 { // 15% failure rate
        return false, "", fmt.Errorf("data analysis failed for %s", task.Data)
    }

    return true, fmt.Sprintf("Data analysis completed for %s", task.Data), nil
}

func (w *Worker) generateReport(task Task) (bool, string, error) {
    // Simulate report generation
    time.Sleep(time.Duration(rand.Intn(250)+150) * time.Millisecond)

    if rand.Float32() < 0.08 { // 8% failure rate
        return false, "", fmt.Errorf("report generation failed for %s", task.Data)
    }

    return true, fmt.Sprintf("Report generated for %s", task.Data), nil
}

func (w *Worker) processGeneric(task Task) (bool, string, error) {
    // Generic processing
    time.Sleep(time.Duration(rand.Intn(100)+50) * time.Millisecond)

    if rand.Float32() < 0.1 { // 10% failure rate
        return false, "", fmt.Errorf("generic processing failed for %s", task.Data)
    }

    return true, fmt.Sprintf("Generic processing completed for %s", task.Data), nil
}

// SubmitTask adds a task to the worker pool queue
func (wp *WorkerPool) SubmitTask(task Task) bool {
    select {
    case wp.TaskQueue <- task:
        return true
    default:
        return false // Queue is full
    }
}

// Shutdown gracefully stops the worker pool
func (wp *WorkerPool) Shutdown() {
    fmt.Printf("Shutting down worker pool '%s'\n", wp.Name)

    close(wp.shutdown)
    close(wp.TaskQueue)

    // Wait for all workers to finish
    wp.wg.Wait()
    close(wp.ResultQueue)

    fmt.Printf("Worker pool '%s' shutdown complete\n", wp.Name)
}

// GetMetrics returns current pool metrics
func (wp *WorkerPool) GetMetrics() (int64, int64, int64) {
    processed := atomic.LoadInt64(&wp.tasksProcessed)
    succeeded := atomic.LoadInt64(&wp.tasksSucceeded)
    failed := atomic.LoadInt64(&wp.tasksFailed)
    return processed, succeeded, failed
}

// PART 2: Demonstration Functions

func demonstrateWorkerPool() {
    fmt.Println("=== Worker Pool Pattern Demonstration ===")

    // Create worker pool
    pool := NewWorkerPool("AutomationPool", 4, 20)
    pool.Start()

    // Generate tasks
    taskTypes := []string{"file_processing", "email_sending", "data_analysis", "report_generation"}
    const numTasks = 15

    fmt.Printf("Submitting %d tasks to worker pool...\n", numTasks)

    // Submit tasks
    for i := 0; i < numTasks; i++ {
        task := Task{
            ID:        fmt.Sprintf("task-%03d", i),
            Type:      taskTypes[i%len(taskTypes)],
            Data:      fmt.Sprintf("data-item-%d", i),
            Priority:  rand.Intn(10),
            CreatedAt: time.Now(),
            Timeout:   5 * time.Second,
        }

        if pool.SubmitTask(task) {
            fmt.Printf("Submitted task %s (type: %s)\n", task.ID, task.Type)
        } else {
            fmt.Printf("Failed to submit task %s (queue full)\n", task.ID)
        }

        time.Sleep(10 * time.Millisecond) // Small delay between submissions
    }

    // Collect results
    fmt.Println("\nCollecting results...")
    var results []Result

    // Collect results with timeout
    timeout := time.After(10 * time.Second)

    for len(results) < numTasks {
        select {
        case result := <-pool.ResultQueue:
            results = append(results, result)

            status := "SUCCESS"
            if !result.Success {
                status = "FAILED"
            }

            fmt.Printf("Result: Task %s - %s (Worker %d, Duration: %v)\n",
                result.TaskID, status, result.ProcessedBy, result.Duration)

            if result.Error != nil {
                fmt.Printf("  Error: %v\n", result.Error)
            }

        case <-timeout:
            fmt.Println("Timeout waiting for results")
            break
        }
    }

    // Get final metrics
    processed, succeeded, failed := pool.GetMetrics()
    fmt.Printf("\nFinal Metrics:\n")
    fmt.Printf("  Tasks Processed: %d\n", processed)
    fmt.Printf("  Tasks Succeeded: %d\n", succeeded)
    fmt.Printf("  Tasks Failed: %d\n", failed)
    fmt.Printf("  Success Rate: %.2f%%\n", float64(succeeded)/float64(processed)*100)

    // Shutdown pool
    pool.Shutdown()
}

func main() {
    fmt.Printf("Starting with GOMAXPROCS=%d\n", runtime.GOMAXPROCS(0))
    fmt.Println("=== Worker Pool and Pooling Patterns Demo ===")

    // Demonstrate worker pool pattern
    demonstrateWorkerPool()

    fmt.Println("\n=== Worker Pool Key Insights ===")
    fmt.Println("✅ FIXED WORKERS: Pool maintains fixed number of goroutines")
    fmt.Println("✅ TASK QUEUE: Buffered channel distributes work to workers")
    fmt.Println("✅ RESULT COLLECTION: Separate channel collects processing results")
    fmt.Println("✅ GRACEFUL SHUTDOWN: Proper cleanup and resource management")
    fmt.Println("✅ METRICS: Track processing statistics and success rates")
    fmt.Println("✅ LOAD BALANCING: Tasks distributed evenly across workers")
}
```

**Hands-on Exercise 3: Fan-Out, Fan-In, and Pipeline Patterns**:

Implementing advanced channel patterns for complex automation workflows:

```go
// Fan-out, fan-in, and pipeline patterns in automation systems
package main

import (
    "fmt"
    "math/rand"
    "runtime"
    "sync"
    "time"
)

// PART 1: Fan-Out Pattern
// Fan-out distributes work from one source to multiple processors

// DataItem represents data flowing through automation pipeline
type DataItem struct {
    ID        string
    Content   string
    Type      string
    Priority  int
    Timestamp time.Time
}

// ProcessedData represents data after processing
type ProcessedData struct {
    OriginalID string
    Result     string
    Processor  string
    Duration   time.Duration
    Success    bool
}

// demonstrateFanOut shows how to distribute work to multiple processors
func demonstrateFanOut() {
    fmt.Println("=== Fan-Out Pattern: Data Distribution ===")

    // Input channel - single source of data
    input := make(chan DataItem, 10)

    // Output channels for different processors
    textProcessor := make(chan DataItem, 5)
    imageProcessor := make(chan DataItem, 5)
    videoProcessor := make(chan DataItem, 5)

    // Fan-out distributor
    go func() {
        defer close(textProcessor)
        defer close(imageProcessor)
        defer close(videoProcessor)

        for item := range input {
            fmt.Printf("Distributing item %s (type: %s)\n", item.ID, item.Type)

            // Route based on data type
            switch item.Type {
            case "text":
                textProcessor <- item
            case "image":
                imageProcessor <- item
            case "video":
                videoProcessor <- item
            default:
                // Default to text processor
                textProcessor <- item
            }
        }

        fmt.Println("Fan-out distribution complete")
    }()

    // Result collection channel
    results := make(chan ProcessedData, 15)

    var wg sync.WaitGroup

    // Text processor
    wg.Add(1)
    go func() {
        defer wg.Done()

        for item := range textProcessor {
            start := time.Now()

            // Simulate text processing
            time.Sleep(time.Duration(rand.Intn(100)+50) * time.Millisecond)

            result := ProcessedData{
                OriginalID: item.ID,
                Result:     fmt.Sprintf("Text processed: %s", item.Content),
                Processor:  "TextProcessor",
                Duration:   time.Since(start),
                Success:    rand.Float32() > 0.1, // 90% success rate
            }

            results <- result
            fmt.Printf("Text processor completed item %s\n", item.ID)
        }
    }()

    // Image processor
    wg.Add(1)
    go func() {
        defer wg.Done()

        for item := range imageProcessor {
            start := time.Now()

            // Simulate image processing (slower)
            time.Sleep(time.Duration(rand.Intn(200)+150) * time.Millisecond)

            result := ProcessedData{
                OriginalID: item.ID,
                Result:     fmt.Sprintf("Image processed: %s", item.Content),
                Processor:  "ImageProcessor",
                Duration:   time.Since(start),
                Success:    rand.Float32() > 0.15, // 85% success rate
            }

            results <- result
            fmt.Printf("Image processor completed item %s\n", item.ID)
        }
    }()

    // Video processor
    wg.Add(1)
    go func() {
        defer wg.Done()

        for item := range videoProcessor {
            start := time.Now()

            // Simulate video processing (slowest)
            time.Sleep(time.Duration(rand.Intn(300)+250) * time.Millisecond)

            result := ProcessedData{
                OriginalID: item.ID,
                Result:     fmt.Sprintf("Video processed: %s", item.Content),
                Processor:  "VideoProcessor",
                Duration:   time.Since(start),
                Success:    rand.Float32() > 0.2, // 80% success rate
            }

            results <- result
            fmt.Printf("Video processor completed item %s\n", item.ID)
        }
    }()

    // Generate test data
    testData := []DataItem{
        {ID: "item-001", Content: "document.txt", Type: "text", Priority: 1, Timestamp: time.Now()},
        {ID: "item-002", Content: "photo.jpg", Type: "image", Priority: 2, Timestamp: time.Now()},
        {ID: "item-003", Content: "report.txt", Type: "text", Priority: 1, Timestamp: time.Now()},
        {ID: "item-004", Content: "video.mp4", Type: "video", Priority: 3, Timestamp: time.Now()},
        {ID: "item-005", Content: "chart.png", Type: "image", Priority: 2, Timestamp: time.Now()},
        {ID: "item-006", Content: "presentation.txt", Type: "text", Priority: 1, Timestamp: time.Now()},
    }

    // Send data to input channel
    go func() {
        defer close(input)

        for _, item := range testData {
            input <- item
            time.Sleep(20 * time.Millisecond)
        }
    }()

    // Close results channel when all processors are done
    go func() {
        wg.Wait()
        close(results)
    }()

    // Collect and display results
    var allResults []ProcessedData
    for result := range results {
        allResults = append(allResults, result)
    }

    fmt.Printf("\nFan-out processing complete. Processed %d items:\n", len(allResults))
    for _, result := range allResults {
        status := "SUCCESS"
        if !result.Success {
            status = "FAILED"
        }
        fmt.Printf("  %s: %s (%s, %v)\n", result.OriginalID, status, result.Processor, result.Duration)
    }
}

// PART 2: Fan-In Pattern
// Fan-in combines multiple sources into a single stream

func demonstrateFanIn() {
    fmt.Println("\n=== Fan-In Pattern: Data Aggregation ===")

    // Multiple data sources
    source1 := make(chan string, 5)
    source2 := make(chan string, 5)
    source3 := make(chan string, 5)

    // Fan-in: combine multiple sources
    combined := fanIn(source1, source2, source3)

    // Start data sources
    go func() {
        defer close(source1)

        for i := 0; i < 4; i++ {
            message := fmt.Sprintf("DatabaseSystem-Record%d", i)
            source1 <- message
            fmt.Printf("Database source sent: %s\n", message)
            time.Sleep(100 * time.Millisecond)
        }
    }()

    go func() {
        defer close(source2)

        for i := 0; i < 3; i++ {
            message := fmt.Sprintf("APISystem-Response%d", i)
            source2 <- message
            fmt.Printf("API source sent: %s\n", message)
            time.Sleep(150 * time.Millisecond)
        }
    }()

    go func() {
        defer close(source3)

        for i := 0; i < 5; i++ {
            message := fmt.Sprintf("FileSystem-Document%d", i)
            source3 <- message
            fmt.Printf("File source sent: %s\n", message)
            time.Sleep(80 * time.Millisecond)
        }
    }()

    // Receive and process combined messages
    fmt.Println("\nReceiving combined messages:")
    for message := range combined {
        fmt.Printf("  Combined stream: %s\n", message)
    }

    fmt.Println("Fan-in aggregation complete")
}

// fanIn combines multiple string channels into one
func fanIn(channels ...<-chan string) <-chan string {
    out := make(chan string)
    var wg sync.WaitGroup

    // Start a goroutine for each input channel
    for i, ch := range channels {
        wg.Add(1)
        go func(channelID int, c <-chan string) {
            defer wg.Done()

            for msg := range c {
                // Add channel identifier to message
                enrichedMsg := fmt.Sprintf("[Source%d] %s", channelID+1, msg)
                out <- enrichedMsg
            }
        }(i, ch)
    }

    // Close output channel when all inputs are done
    go func() {
        wg.Wait()
        close(out)
    }()

    return out
}

// PART 3: Pipeline Pattern
// Pipeline processes data through multiple sequential stages

// AutomationPipeline represents a multi-stage processing pipeline
type AutomationPipeline struct {
    Name     string
    Stages   []PipelineStage
    Workers  int
}

// PipelineStage represents a single stage in the pipeline
type PipelineStage struct {
    Name      string
    Processor func(string) string
    Workers   int
}

// NewAutomationPipeline creates a new pipeline
func NewAutomationPipeline(name string) *AutomationPipeline {
    return &AutomationPipeline{
        Name:    name,
        Stages:  make([]PipelineStage, 0),
        Workers: 1,
    }
}

// AddStage adds a processing stage to the pipeline
func (ap *AutomationPipeline) AddStage(name string, processor func(string) string, workers int) {
    stage := PipelineStage{
        Name:      name,
        Processor: processor,
        Workers:   workers,
    }
    ap.Stages = append(ap.Stages, stage)
}

// Process executes the pipeline on input items
func (ap *AutomationPipeline) Process(items []string) []string {
    fmt.Printf("\n=== Pipeline: %s ===\n", ap.Name)
    fmt.Printf("Processing %d items through %d stages\n", len(items), len(ap.Stages))

    // Create channels for each stage
    channels := make([]chan string, len(ap.Stages)+1)
    for i := range channels {
        channels[i] = make(chan string, 10)
    }

    // Fill input channel
    go func() {
        defer close(channels[0])

        for _, item := range items {
            channels[0] <- item
        }
    }()

    // Start processing stages
    var wg sync.WaitGroup

    for i, stage := range ap.Stages {
        wg.Add(1)
        go func(stageIndex int, s PipelineStage) {
            defer wg.Done()
            defer close(channels[stageIndex+1])

            fmt.Printf("Starting stage: %s with %d workers\n", s.Name, s.Workers)

            // Process items through this stage
            for item := range channels[stageIndex] {
                start := time.Now()

                // Apply stage processor
                processed := s.Processor(item)

                duration := time.Since(start)
                fmt.Printf("Stage %s: %s -> %s (%v)\n", s.Name, item, processed, duration)

                channels[stageIndex+1] <- processed
            }

            fmt.Printf("Stage %s completed\n", s.Name)
        }(i, stage)
    }

    // Collect final results
    go func() {
        wg.Wait()
    }()

    var results []string
    for result := range channels[len(ap.Stages)] {
        results = append(results, result)
    }

    fmt.Printf("Pipeline processed %d items successfully\n", len(results))
    return results
}

func demonstratePipeline() {
    fmt.Println("\n=== Pipeline Pattern: Multi-Stage Processing ===")

    // Create automation pipeline
    pipeline := NewAutomationPipeline("DataProcessingPipeline")

    // Add processing stages
    pipeline.AddStage("Validate", func(item string) string {
        time.Sleep(30 * time.Millisecond) // Simulate validation
        return fmt.Sprintf("validated(%s)", item)
    }, 1)

    pipeline.AddStage("Transform", func(item string) string {
        time.Sleep(50 * time.Millisecond) // Simulate transformation
        return fmt.Sprintf("transformed(%s)", item)
    }, 2)

    pipeline.AddStage("Enrich", func(item string) string {
        time.Sleep(40 * time.Millisecond) // Simulate enrichment
        return fmt.Sprintf("enriched(%s)", item)
    }, 1)

    pipeline.AddStage("Store", func(item string) string {
        time.Sleep(60 * time.Millisecond) // Simulate storage
        return fmt.Sprintf("stored(%s)", item)
    }, 1)

    // Process test data
    testItems := []string{"data1", "data2", "data3", "data4", "data5"}
    results := pipeline.Process(testItems)

    fmt.Println("\nFinal Results:")
    for i, result := range results {
        fmt.Printf("  %d: %s\n", i+1, result)
    }
}

func main() {
    fmt.Printf("Starting with GOMAXPROCS=%d\n", runtime.GOMAXPROCS(0))
    fmt.Println("=== Advanced Channel Patterns Demo ===")

    // Demonstrate fan-out pattern
    demonstrateFanOut()

    // Demonstrate fan-in pattern
    demonstrateFanIn()

    // Demonstrate pipeline pattern
    demonstratePipeline()

    fmt.Println("\n=== Advanced Channel Pattern Insights ===")
    fmt.Println("✅ FAN-OUT: Distribute work from one source to multiple processors")
    fmt.Println("✅ FAN-IN: Combine multiple sources into single stream")
    fmt.Println("✅ PIPELINE: Sequential processing through multiple stages")
    fmt.Println("✅ COMPOSITION: Patterns can be combined for complex workflows")
    fmt.Println("✅ SCALABILITY: Each pattern supports concurrent processing")
    fmt.Println("✅ FLEXIBILITY: Processors can be added/removed dynamically")
}
```

**Prerequisites**: Module 29
