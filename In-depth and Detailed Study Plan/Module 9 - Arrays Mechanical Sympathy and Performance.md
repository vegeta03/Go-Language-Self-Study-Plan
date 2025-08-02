# Module 9: Arrays: Mechanical Sympathy and Performance

**Duration**: 1 hour (45 min theory + 15 min hands-on)

**Learning Objectives**:

- Understand array fundamentals and memory layout
- Learn mechanical sympathy principles with arrays
- Master array performance characteristics
- Apply arrays effectively in automation scenarios

**Videos Covered**:

- 3.3 Arrays Part 1 (Mechanical Sympathy) (0:33:10)

**Key Concepts**:

- Array declaration and initialization
- Contiguous memory layout and cache performance
- Array vs slice performance trade-offs
- Fixed-size collections in automation
- Memory predictability and allocation

**Hands-on Exercise 1: High-Performance Metrics Collection**:

```go
// High-performance metrics collection using arrays
package main

import (
    "fmt"
    "time"
)

const (
    MetricsBufferSize = 1000
    SampleCount      = 100
)

// Fixed-size metrics buffer for predictable performance
type MetricsCollector struct {
    cpuUsage    [MetricsBufferSize]float64
    memoryUsage [MetricsBufferSize]float64
    timestamps  [MetricsBufferSize]int64
    index       int
    full        bool
}

func (mc *MetricsCollector) AddMetric(cpu, memory float64) {
    mc.cpuUsage[mc.index] = cpu
    mc.memoryUsage[mc.index] = memory
    mc.timestamps[mc.index] = time.Now().Unix()

    mc.index++
    if mc.index >= MetricsBufferSize {
        mc.index = 0
        mc.full = true
    }
}

func (mc *MetricsCollector) GetAverages() (float64, float64) {
    count := mc.index
    if mc.full {
        count = MetricsBufferSize
    }

    if count == 0 {
        return 0, 0
    }

    var cpuSum, memSum float64
    for i := 0; i < count; i++ {
        cpuSum += mc.cpuUsage[i]
        memSum += mc.memoryUsage[i]
    }

    return cpuSum / float64(count), memSum / float64(count)
}

func main() {
    collector := &MetricsCollector{}

    // Simulate collecting metrics
    fmt.Println("Collecting system metrics...")
    for i := 0; i < SampleCount; i++ {
        // Simulate CPU and memory readings
        cpu := 20.0 + float64(i%50)
        memory := 60.0 + float64(i%30)

        collector.AddMetric(cpu, memory)

        if i%20 == 0 {
            avgCPU, avgMem := collector.GetAverages()
            fmt.Printf("Sample %d - Avg CPU: %.2f%%, Avg Memory: %.2f%%\n",
                i, avgCPU, avgMem)
        }
    }

    finalCPU, finalMem := collector.GetAverages()
    fmt.Printf("\nFinal averages - CPU: %.2f%%, Memory: %.2f%%\n",
        finalCPU, finalMem)
}
```

**Hands-on Exercise 2: Cache Performance Analysis**:

```go
// Demonstrate mechanical sympathy with array traversal patterns
package main

import (
    "fmt"
    "time"
)

const (
    MatrixSize = 2000
    Iterations = 5
)

// Large matrix for cache performance testing
type Matrix [MatrixSize][MatrixSize]int

func main() {
    matrix := &Matrix{}

    // Initialize matrix with test data
    initializeMatrix(matrix)

    // Test different traversal patterns
    fmt.Println("=== Cache Performance Analysis ===")

    // Row-major traversal (cache-friendly)
    rowTime := benchmarkRowTraversal(matrix)
    fmt.Printf("Row-major traversal: %v\n", rowTime)

    // Column-major traversal (cache-unfriendly)
    colTime := benchmarkColumnTraversal(matrix)
    fmt.Printf("Column-major traversal: %v\n", colTime)

    // Performance difference
    ratio := float64(colTime) / float64(rowTime)
    fmt.Printf("Column/Row ratio: %.2fx slower\n", ratio)

    // Block traversal (cache-optimized)
    blockTime := benchmarkBlockTraversal(matrix)
    fmt.Printf("Block traversal: %v\n", blockTime)

    blockRatio := float64(blockTime) / float64(rowTime)
    fmt.Printf("Block/Row ratio: %.2fx\n", blockRatio)
}

func initializeMatrix(matrix *Matrix) {
    for i := 0; i < MatrixSize; i++ {
        for j := 0; j < MatrixSize; j++ {
            matrix[i][j] = i*MatrixSize + j
        }
    }
}

func benchmarkRowTraversal(matrix *Matrix) time.Duration {
    start := time.Now()

    for iter := 0; iter < Iterations; iter++ {
        sum := 0
        for i := 0; i < MatrixSize; i++ {
            for j := 0; j < MatrixSize; j++ {
                sum += matrix[i][j]
            }
        }
        _ = sum // Prevent optimization
    }

    return time.Since(start)
}

func benchmarkColumnTraversal(matrix *Matrix) time.Duration {
    start := time.Now()

    for iter := 0; iter < Iterations; iter++ {
        sum := 0
        for j := 0; j < MatrixSize; j++ {
            for i := 0; i < MatrixSize; i++ {
                sum += matrix[i][j]
            }
        }
        _ = sum // Prevent optimization
    }

    return time.Since(start)
}

func benchmarkBlockTraversal(matrix *Matrix) time.Duration {
    const BlockSize = 64
    start := time.Now()

    for iter := 0; iter < Iterations; iter++ {
        sum := 0
        for bi := 0; bi < MatrixSize; bi += BlockSize {
            for bj := 0; bj < MatrixSize; bj += BlockSize {
                // Process block
                maxI := min(bi+BlockSize, MatrixSize)
                maxJ := min(bj+BlockSize, MatrixSize)
                for i := bi; i < maxI; i++ {
                    for j := bj; j < maxJ; j++ {
                        sum += matrix[i][j]
                    }
                }
            }
        }
        _ = sum // Prevent optimization
    }

    return time.Since(start)
}

func min(a, b int) int {
    if a < b {
        return a
    }
    return b
}
```

**Hands-on Exercise 3: Fixed-Size Task Queue**:

```go
// Fixed-size task queue for predictable memory usage
package main

import (
    "fmt"
    "time"
)

const (
    QueueSize = 100
    TaskTypes = 5
)

// Task represents an automation task
type Task struct {
    ID       int
    Type     int
    Priority int
    Data     string
}

// Fixed-size circular queue using arrays
type TaskQueue struct {
    tasks [QueueSize]Task
    head  int
    tail  int
    count int
    full  bool
}

func NewTaskQueue() *TaskQueue {
    return &TaskQueue{}
}

func (tq *TaskQueue) Enqueue(task Task) bool {
    if tq.full {
        return false // Queue is full
    }

    tq.tasks[tq.tail] = task
    tq.tail = (tq.tail + 1) % QueueSize
    tq.count++

    if tq.tail == tq.head {
        tq.full = true
    }

    return true
}

func (tq *TaskQueue) Dequeue() (Task, bool) {
    if tq.count == 0 {
        return Task{}, false // Queue is empty
    }

    task := tq.tasks[tq.head]
    tq.head = (tq.head + 1) % QueueSize
    tq.count--
    tq.full = false

    return task, true
}

func (tq *TaskQueue) Size() int {
    return tq.count
}

func (tq *TaskQueue) IsFull() bool {
    return tq.full
}

func (tq *TaskQueue) IsEmpty() bool {
    return tq.count == 0
}

// Get statistics by task type
func (tq *TaskQueue) GetTypeStats() [TaskTypes]int {
    var stats [TaskTypes]int

    if tq.count == 0 {
        return stats
    }

    // Traverse the circular queue
    for i := 0; i < tq.count; i++ {
        index := (tq.head + i) % QueueSize
        taskType := tq.tasks[index].Type
        if taskType >= 0 && taskType < TaskTypes {
            stats[taskType]++
        }
    }

    return stats
}

func main() {
    queue := NewTaskQueue()

    fmt.Println("=== Fixed-Size Task Queue Demo ===")

    // Add tasks to queue
    tasks := []Task{
        {ID: 1, Type: 0, Priority: 5, Data: "Process file A"},
        {ID: 2, Type: 1, Priority: 3, Data: "Send notification"},
        {ID: 3, Type: 0, Priority: 7, Data: "Process file B"},
        {ID: 4, Type: 2, Priority: 1, Data: "Cleanup temp files"},
        {ID: 5, Type: 1, Priority: 4, Data: "Send alert"},
        {ID: 6, Type: 3, Priority: 6, Data: "Backup database"},
        {ID: 7, Type: 0, Priority: 2, Data: "Process file C"},
    }

    // Enqueue tasks
    for _, task := range tasks {
        if queue.Enqueue(task) {
            fmt.Printf("Enqueued task %d (type %d)\n", task.ID, task.Type)
        } else {
            fmt.Printf("Failed to enqueue task %d - queue full\n", task.ID)
        }
    }

    fmt.Printf("\nQueue size: %d\n", queue.Size())

    // Show type statistics
    stats := queue.GetTypeStats()
    fmt.Println("\nTask type statistics:")
    typeNames := []string{"File Processing", "Notifications", "Cleanup", "Backup", "Other"}
    for i, count := range stats {
        if count > 0 {
            fmt.Printf("  %s: %d tasks\n", typeNames[i], count)
        }
    }

    // Process some tasks
    fmt.Println("\nProcessing tasks:")
    for i := 0; i < 3; i++ {
        if task, ok := queue.Dequeue(); ok {
            fmt.Printf("Processing task %d: %s\n", task.ID, task.Data)
            time.Sleep(100 * time.Millisecond) // Simulate processing
        }
    }

    fmt.Printf("\nRemaining queue size: %d\n", queue.Size())

    // Add more tasks
    newTasks := []Task{
        {ID: 8, Type: 4, Priority: 8, Data: "Emergency task"},
        {ID: 9, Type: 2, Priority: 1, Data: "Regular cleanup"},
    }

    for _, task := range newTasks {
        if queue.Enqueue(task) {
            fmt.Printf("Added new task %d\n", task.ID)
        }
    }

    fmt.Printf("Final queue size: %d\n", queue.Size())
}
```

**Prerequisites**: Module 8
