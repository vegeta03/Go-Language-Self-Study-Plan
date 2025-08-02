# Module 33: Advanced Testing and Benchmarking

**Duration**: 1 hour (45 min theory + 15 min hands-on)

**Learning Objectives**:

- Master advanced testing patterns including sub-tests and example tests
- Understand code coverage analysis and interpretation
- Learn benchmarking fundamentals and performance measurement
- Implement sub-benchmarks for comprehensive performance testing
- Apply benchmark validation techniques to ensure accurate results
- Design performance-aware automation systems

**Videos Covered**:

- 12.5 Testing Internal Endpoints (0:07:22)
- 12.6 Example Tests (0:09:55)
- 12.7 Sub Tests (0:05:35)
- 12.8 Code Coverage (0:04:44)
- 13.1 Topics (0:00:46)
- 13.2 Basic Benchmarking (0:07:26)
- 13.3 Sub Benchmarks (0:03:35)
- 13.4 Validate Benchmarks (0:07:41)

**Key Concepts**:

- Sub-tests for organizing related test cases hierarchically
- Example tests for executable documentation and API examples
- Code coverage measurement and analysis techniques
- Testing internal vs exported APIs and their trade-offs
- Benchmarking fundamentals: timing, iterations, and accuracy
- Sub-benchmarks for testing performance across different parameters
- Memory allocation tracking in benchmarks (`-benchmem`)
- Performance regression detection and validation

**Hands-on Exercise 1: Sub-Tests and Test Organization**:

Implementing hierarchical test organization with sub-tests for better structure and reporting:

```go
// Advanced testing patterns for automation task queue system
package automation

import (
    "fmt"
    "sort"
    "sync"
    "time"
)

// TaskQueue manages a queue of automation tasks with advanced features
type TaskQueue struct {
    tasks    []*AutomationTask
    capacity int
    mu       sync.RWMutex
    stats    QueueStats
}

// QueueStats tracks queue performance metrics
type QueueStats struct {
    TotalAdded     int64
    TotalProcessed int64
    TotalDropped   int64
    AverageWaitTime time.Duration
    mu             sync.RWMutex
}

// AutomationTask represents a task in the automation system
type AutomationTask struct {
    ID          string
    Type        string
    Data        []byte
    Priority    int
    CreatedAt   time.Time
    ProcessedAt *time.Time
    Status      TaskStatus
    Metadata    map[string]string
}

type TaskStatus int

const (
    StatusPending TaskStatus = iota
    StatusProcessing
    StatusCompleted
    StatusFailed
    StatusCancelled
)

func (ts TaskStatus) String() string {
    switch ts {
    case StatusPending:
        return "pending"
    case StatusProcessing:
        return "processing"
    case StatusCompleted:
        return "completed"
    case StatusFailed:
        return "failed"
    case StatusCancelled:
        return "cancelled"
    default:
        return "unknown"
    }
}

func NewTaskQueue(capacity int) *TaskQueue {
    return &TaskQueue{
        tasks:    make([]*AutomationTask, 0, capacity),
        capacity: capacity,
    }
}

// Add adds a task to the queue with thread safety
func (tq *TaskQueue) Add(task *AutomationTask) error {
    tq.mu.Lock()
    defer tq.mu.Unlock()

    if len(tq.tasks) >= tq.capacity {
        tq.stats.mu.Lock()
        tq.stats.TotalDropped++
        tq.stats.mu.Unlock()
        return fmt.Errorf("queue is full (capacity: %d)", tq.capacity)
    }

    tq.tasks = append(tq.tasks, task)

    tq.stats.mu.Lock()
    tq.stats.TotalAdded++
    tq.stats.mu.Unlock()

    return nil
}

// Next retrieves the highest priority pending task
func (tq *TaskQueue) Next() *AutomationTask {
    tq.mu.Lock()
    defer tq.mu.Unlock()

    if len(tq.tasks) == 0 {
        return nil
    }

    // Find highest priority pending task
    var bestIndex = -1
    var bestPriority = -1

    for i, task := range tq.tasks {
        if task.Status == StatusPending && task.Priority > bestPriority {
            bestIndex = i
            bestPriority = task.Priority
        }
    }

    if bestIndex == -1 {
        return nil
    }

    task := tq.tasks[bestIndex]
    // Remove from queue
    tq.tasks = append(tq.tasks[:bestIndex], tq.tasks[bestIndex+1:]...)

    tq.stats.mu.Lock()
    tq.stats.TotalProcessed++
    tq.stats.mu.Unlock()

    return task
}

// Size returns the current queue size (thread-safe)
func (tq *TaskQueue) Size() int {
    tq.mu.RLock()
    defer tq.mu.RUnlock()
    return len(tq.tasks)
}

// PendingCount returns the number of pending tasks
func (tq *TaskQueue) PendingCount() int {
    tq.mu.RLock()
    defer tq.mu.RUnlock()

    count := 0
    for _, task := range tq.tasks {
        if task.Status == StatusPending {
            count++
        }
    }
    return count
}

// GetTasksByPriority returns tasks sorted by priority (highest first)
func (tq *TaskQueue) GetTasksByPriority() []*AutomationTask {
    tq.mu.RLock()
    defer tq.mu.RUnlock()

    // Create a copy to avoid race conditions
    tasks := make([]*AutomationTask, len(tq.tasks))
    copy(tasks, tq.tasks)

    sort.Slice(tasks, func(i, j int) bool {
        return tasks[i].Priority > tasks[j].Priority
    })

    return tasks
}

// GetTasksByType returns tasks filtered by type
func (tq *TaskQueue) GetTasksByType(taskType string) []*AutomationTask {
    tq.mu.RLock()
    defer tq.mu.RUnlock()

    var filtered []*AutomationTask
    for _, task := range tq.tasks {
        if task.Type == taskType {
            filtered = append(filtered, task)
        }
    }

    return filtered
}

// Clear removes all tasks from the queue
func (tq *TaskQueue) Clear() {
    tq.mu.Lock()
    defer tq.mu.Unlock()

    tq.tasks = tq.tasks[:0]
}

// GetStats returns current queue statistics
func (tq *TaskQueue) GetStats() QueueStats {
    tq.stats.mu.RLock()
    defer tq.stats.mu.RUnlock()

    return QueueStats{
        TotalAdded:     tq.stats.TotalAdded,
        TotalProcessed: tq.stats.TotalProcessed,
        TotalDropped:   tq.stats.TotalDropped,
        AverageWaitTime: tq.stats.AverageWaitTime,
    }
}
```

**Test File (queue_test.go):**

```go
package automation_test

import (
    "automation"
    "fmt"
    "sync"
    "testing"
    "time"
)

// Constants for consistent test output
const (
    checkMark = "\u2713" // ✓
    ballotX   = "\u2717" // ✗
)

// PART 1: Sub-Tests for TaskQueue
// Sub-tests organize related test cases under a common parent test

func TestTaskQueue(t *testing.T) {
    t.Log("Given the need to test task queue operations comprehensively")

    // Sub-test for basic queue operations
    t.Run("BasicOperations", func(t *testing.T) {
        t.Log("\tGiven the need to test basic queue operations")

        t.Run("Add and Size", func(t *testing.T) {
            t.Log("\t\tWhen adding a task to an empty queue")
            {
                // Given
                queue := automation.NewTaskQueue(5)
                task := &automation.AutomationTask{
                    ID:     "test-1",
                    Type:   "text",
                    Status: automation.StatusPending,
                }

                // When
                err := queue.Add(task)

                // Should
                if err != nil {
                    t.Fatalf("\t\t%s\tShould add task without error. Got: %v", ballotX, err)
                }
                t.Logf("\t\t%s\tShould add task without error", checkMark)

                if queue.Size() != 1 {
                    t.Fatalf("\t\t%s\tShould have size 1. Got: %d", ballotX, queue.Size())
                }
                t.Logf("\t\t%s\tShould have size 1", checkMark)

                if queue.PendingCount() != 1 {
                    t.Fatalf("\t\t%s\tShould have 1 pending task. Got: %d", ballotX, queue.PendingCount())
                }
                t.Logf("\t\t%s\tShould have 1 pending task", checkMark)
            }
        })

        t.Run("Empty Queue Next", func(t *testing.T) {
            t.Log("\t\tWhen calling Next on an empty queue")
            {
                // Given
                queue := automation.NewTaskQueue(5)

                // When
                task := queue.Next()

                // Should
                if task != nil {
                    t.Fatalf("\t\t%s\tShould return nil from empty queue. Got: %v", ballotX, task)
                }
                t.Logf("\t\t%s\tShould return nil from empty queue", checkMark)
            }
        })

        t.Run("Clear Queue", func(t *testing.T) {
            t.Log("\t\tWhen clearing a queue with multiple tasks")
            {
                // Given
                queue := automation.NewTaskQueue(5)

                // Add some tasks
                for i := 0; i < 3; i++ {
                    task := &automation.AutomationTask{
                        ID:     fmt.Sprintf("test-%d", i),
                        Status: automation.StatusPending,
                    }
                    queue.Add(task)
                }

                if queue.Size() != 3 {
                    t.Fatalf("\t\t%s\tShould have 3 tasks before clear. Got: %d", ballotX, queue.Size())
                }

                // When
                queue.Clear()

                // Should
                if queue.Size() != 0 {
                    t.Fatalf("\t\t%s\tShould have size 0 after clear. Got: %d", ballotX, queue.Size())
                }
                t.Logf("\t\t%s\tShould have size 0 after clear", checkMark)
            }
        })
    })

    // Sub-test for capacity management
    t.Run("CapacityManagement", func(t *testing.T) {
        t.Log("\tGiven the need to test queue capacity limits")

        t.Run("Capacity Limit", func(t *testing.T) {
            t.Log("\t\tWhen adding tasks up to and beyond capacity")
            {
                // Given
                queue := automation.NewTaskQueue(2)

                // Add tasks up to capacity
                for i := 0; i < 2; i++ {
                    task := &automation.AutomationTask{
                        ID:     fmt.Sprintf("test-%d", i),
                        Status: automation.StatusPending,
                    }
                    if err := queue.Add(task); err != nil {
                        t.Fatalf("\t\t%s\tShould add task %d without error. Got: %v", ballotX, i, err)
                    }
                }

                // Try to add one more (should fail)
                task := &automation.AutomationTask{
                    ID:     "overflow",
                    Status: automation.StatusPending,
                }
                err := queue.Add(task)

                // Should
                if err == nil {
                    t.Fatalf("\t\t%s\tShould return error when exceeding capacity", ballotX)
                }
                t.Logf("\t\t%s\tShould return error when exceeding capacity", checkMark)

                // Check stats
                stats := queue.GetStats()
                if stats.TotalAdded != 2 {
                    t.Fatalf("\t\t%s\tShould have 2 tasks added. Got: %d", ballotX, stats.TotalAdded)
                }
                t.Logf("\t\t%s\tShould have 2 tasks added", checkMark)

                if stats.TotalDropped != 1 {
                    t.Fatalf("\t\t%s\tShould have 1 task dropped. Got: %d", ballotX, stats.TotalDropped)
                }
                t.Logf("\t\t%s\tShould have 1 task dropped", checkMark)
            }
        })

        t.Run("Zero Capacity", func(t *testing.T) {
            t.Log("\t\tWhen creating queue with zero capacity")
            {
                // Given
                queue := automation.NewTaskQueue(0)
                task := &automation.AutomationTask{
                    ID:     "test",
                    Status: automation.StatusPending,
                }

                // When
                err := queue.Add(task)

                // Should
                if err == nil {
                    t.Fatalf("\t\t%s\tShould return error for zero capacity queue", ballotX)
                }
                t.Logf("\t\t%s\tShould return error for zero capacity queue", checkMark)
            }
        })
    })
}
```

**Hands-on Exercise 2: Example Tests and Code Coverage**:

Implementing example tests for documentation and measuring code coverage:

```go
// Example tests demonstrate API usage and serve as executable documentation
package automation_test

import (
    "automation"
    "fmt"
    "time"
)

// Example tests are executed as part of the test suite and appear in documentation
// They must have a function comment starting with the function name

// ExampleNewTaskQueue demonstrates basic task queue creation and usage
func ExampleNewTaskQueue() {
    // Create a new task queue with capacity of 10
    queue := automation.NewTaskQueue(10)

    // Create and add a task
    task := &automation.AutomationTask{
        ID:       "example-task-1",
        Type:     "text",
        Priority: 5,
        Status:   automation.StatusPending,
    }

    err := queue.Add(task)
    if err != nil {
        fmt.Printf("Error adding task: %v\n", err)
        return
    }

    fmt.Printf("Queue size: %d\n", queue.Size())
    fmt.Printf("Pending tasks: %d\n", queue.PendingCount())

    // Output:
    // Queue size: 1
    // Pending tasks: 1
}

// ExampleTaskQueue_Add demonstrates adding tasks with different priorities
func ExampleTaskQueue_Add() {
    queue := automation.NewTaskQueue(5)

    // Add tasks with different priorities
    priorities := []int{1, 5, 3}
    for i, priority := range priorities {
        task := &automation.AutomationTask{
            ID:       fmt.Sprintf("task-%d", i+1),
            Type:     "text",
            Priority: priority,
            Status:   automation.StatusPending,
        }
        queue.Add(task)
    }

    fmt.Printf("Total tasks: %d\n", queue.Size())

    // Get tasks sorted by priority
    sortedTasks := queue.GetTasksByPriority()
    fmt.Printf("Highest priority task: %s (priority: %d)\n",
        sortedTasks[0].ID, sortedTasks[0].Priority)

    // Output:
    // Total tasks: 3
    // Highest priority task: task-2 (priority: 5)
}

// ExampleTaskQueue_Next demonstrates task processing order
func ExampleTaskQueue_Next() {
    queue := automation.NewTaskQueue(10)

    // Add tasks with different priorities
    tasks := []*automation.AutomationTask{
        {ID: "low", Priority: 1, Status: automation.StatusPending},
        {ID: "high", Priority: 10, Status: automation.StatusPending},
        {ID: "medium", Priority: 5, Status: automation.StatusPending},
    }

    for _, task := range tasks {
        queue.Add(task)
    }

    // Process tasks - should get highest priority first
    for i := 0; i < 3; i++ {
        task := queue.Next()
        if task != nil {
            fmt.Printf("Processing task: %s (priority: %d)\n", task.ID, task.Priority)
        }
    }

    // Output:
    // Processing task: high (priority: 10)
    // Processing task: medium (priority: 5)
    // Processing task: low (priority: 1)
}
```

**Hands-on Exercise 3: Basic Benchmarking and Sub-Benchmarks**:

Implementing performance benchmarks with sub-benchmarks for comprehensive testing:

```go
// Benchmarking tests for automation task queue performance
package automation_test

import (
    "automation"
    "fmt"
    "testing"
    "time"
)

// PART 3: Basic Benchmarks
// Benchmarks measure performance and help identify bottlenecks

func BenchmarkTaskQueue_Add(b *testing.B) {
    queue := automation.NewTaskQueue(b.N + 1000) // Ensure capacity

    b.ResetTimer()
    for i := 0; i < b.N; i++ {
        task := &automation.AutomationTask{
            ID:     fmt.Sprintf("bench-task-%d", i),
            Type:   "text",
            Status: automation.StatusPending,
        }
        queue.Add(task)
    }
}

func BenchmarkTaskQueue_Next(b *testing.B) {
    queue := automation.NewTaskQueue(b.N + 1000)

    // Pre-populate queue
    for i := 0; i < b.N; i++ {
        task := &automation.AutomationTask{
            ID:       fmt.Sprintf("bench-task-%d", i),
            Priority: i % 10,
            Status:   automation.StatusPending,
        }
        queue.Add(task)
    }

    b.ResetTimer()
    for i := 0; i < b.N; i++ {
        queue.Next()
    }
}

func BenchmarkTaskQueue_Size(b *testing.B) {
    queue := automation.NewTaskQueue(1000)

    // Add some tasks
    for i := 0; i < 500; i++ {
        task := &automation.AutomationTask{
            ID:     fmt.Sprintf("bench-task-%d", i),
            Status: automation.StatusPending,
        }
        queue.Add(task)
    }

    b.ResetTimer()
    for i := 0; i < b.N; i++ {
        queue.Size()
    }
}

// PART 4: Sub-Benchmarks for Different Parameters
// Sub-benchmarks allow testing performance across different scenarios

func BenchmarkTaskQueue_Operations(b *testing.B) {
    capacities := []int{10, 100, 1000, 10000}

    for _, capacity := range capacities {
        b.Run(fmt.Sprintf("Add-capacity-%d", capacity), func(b *testing.B) {
            queue := automation.NewTaskQueue(capacity)

            b.ResetTimer()
            for i := 0; i < b.N && i < capacity; i++ {
                task := &automation.AutomationTask{
                    ID:     fmt.Sprintf("task-%d", i),
                    Status: automation.StatusPending,
                }
                queue.Add(task)
            }
        })
    }

    for _, capacity := range capacities {
        b.Run(fmt.Sprintf("Next-capacity-%d", capacity), func(b *testing.B) {
            queue := automation.NewTaskQueue(capacity)

            // Pre-populate with half capacity
            populateCount := capacity / 2
            for i := 0; i < populateCount; i++ {
                task := &automation.AutomationTask{
                    ID:       fmt.Sprintf("task-%d", i),
                    Priority: i % 10,
                    Status:   automation.StatusPending,
                }
                queue.Add(task)
            }

            b.ResetTimer()
            for i := 0; i < b.N && i < populateCount; i++ {
                queue.Next()
            }
        })
    }
}

func BenchmarkTaskQueue_PriorityScenarios(b *testing.B) {
    scenarios := []struct {
        name     string
        taskCount int
        maxPriority int
    }{
        {"small-low-priority", 10, 3},
        {"small-high-priority", 10, 100},
        {"medium-low-priority", 100, 3},
        {"medium-high-priority", 100, 100},
        {"large-low-priority", 1000, 3},
        {"large-high-priority", 1000, 100},
    }

    for _, scenario := range scenarios {
        b.Run(scenario.name, func(b *testing.B) {
            b.ResetTimer()
            for i := 0; i < b.N; i++ {
                b.StopTimer()
                queue := automation.NewTaskQueue(scenario.taskCount + 100)

                // Add tasks with varying priorities
                for j := 0; j < scenario.taskCount; j++ {
                    task := &automation.AutomationTask{
                        ID:       fmt.Sprintf("task-%d", j),
                        Priority: j % scenario.maxPriority,
                        Status:   automation.StatusPending,
                    }
                    queue.Add(task)
                }
                b.StartTimer()

                // Benchmark getting next task (highest priority)
                queue.Next()
            }
        })
    }
}

// PART 5: Memory Allocation Benchmarks
func BenchmarkTaskQueue_MemoryAllocation(b *testing.B) {
    b.Run("TaskCreation", func(b *testing.B) {
        b.ReportAllocs() // Report memory allocations

        b.ResetTimer()
        for i := 0; i < b.N; i++ {
            task := &automation.AutomationTask{
                ID:        fmt.Sprintf("memory-task-%d", i),
                Type:      "text",
                Data:      make([]byte, 256),
                Priority:  i % 10,
                CreatedAt: time.Now(),
                Status:    automation.StatusPending,
                Metadata:  make(map[string]string),
            }
            _ = task // Prevent optimization
        }
    })

    b.Run("QueueOperations", func(b *testing.B) {
        b.ReportAllocs()

        b.ResetTimer()
        for i := 0; i < b.N; i++ {
            queue := automation.NewTaskQueue(100)

            for j := 0; j < 50; j++ {
                task := &automation.AutomationTask{
                    ID:     fmt.Sprintf("queue-task-%d", j),
                    Status: automation.StatusPending,
                }
                queue.Add(task)
            }

            for j := 0; j < 25; j++ {
                queue.Next()
            }
        }
    })
}

// Example benchmark with custom metrics
func BenchmarkTaskQueue_WithMetrics(b *testing.B) {
    queue := automation.NewTaskQueue(10000)

    var totalTasks int64
    var totalOperations int64

    b.ResetTimer()
    start := time.Now()

    for i := 0; i < b.N; i++ {
        task := &automation.AutomationTask{
            ID:     fmt.Sprintf("metrics-task-%d", i),
            Status: automation.StatusPending,
        }

        queue.Add(task)
        totalTasks++
        totalOperations++

        if i%2 == 0 {
            queue.Next()
            totalOperations++
        }
    }

    duration := time.Since(start)

    // Report custom metrics
    b.ReportMetric(float64(totalTasks)/duration.Seconds(), "tasks/sec")
    b.ReportMetric(float64(totalOperations)/duration.Seconds(), "ops/sec")
    b.ReportMetric(float64(queue.Size()), "final_queue_size")
}
```

**Running the Exercises:**

```bash
# Exercise 1: Sub-Tests and Test Organization
go test -v -run TestTaskQueue

# Exercise 2: Example Tests and Code Coverage
go test -v -run Example
go test -cover -coverprofile=coverage.out
go tool cover -html=coverage.out

# Exercise 3: Basic Benchmarking and Sub-Benchmarks
go test -bench=BenchmarkTaskQueue -benchmem
go test -bench=BenchmarkTaskQueue_Operations -benchmem
go test -bench=BenchmarkTaskQueue_PriorityScenarios -benchmem

# Run benchmarks multiple times for accuracy
go test -bench=. -count=5

# Run benchmarks for specific duration
go test -bench=. -benchtime=10s

# Generate CPU profile during benchmarking
go test -bench=BenchmarkTaskQueue_Operations -cpuprofile=cpu.prof
go tool pprof cpu.prof
```

**Prerequisites**: Module 32
