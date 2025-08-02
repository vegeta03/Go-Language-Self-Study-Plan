# Module 29: Data Races and Synchronization

**Duration**: 1 hour (45 min theory + 15 min hands-on)

**Learning Objectives**:

- Understand data races and their dangers in concurrent programs
- Master synchronization primitives: mutexes, atomic operations
- Learn race detection tools and techniques
- Apply safe concurrency patterns in automation systems
- Design race-free automation components

**Videos Covered**:

- 9.1 Topics (0:00:53)
- 9.2 Cache Coherency and False Sharing (0:12:39)
- 9.3 Synchronization with Atomic Functions (0:11:30)
- 9.4 Synchronization with Mutexes (0:14:38)
- 9.5 Race Detection (0:04:48)
- 9.6 Map Data Race (0:04:01)
- 9.7 Interface-Based Race Condition (0:08:14)

**Key Concepts**:

- Data races: definition and consequences
- Cache coherency and false sharing
- Atomic operations: when and how to use them
- Mutexes: exclusive access to shared resources
- Read/write mutexes: optimizing for read-heavy workloads
- Race detection: tools and techniques
- Safe patterns for concurrent access

**Hands-on Exercise 1: Data Races and Cache Coherency**:

Understanding data races, their detection, and cache coherency issues:

```go
// Data races, cache coherency, and false sharing in automation systems
package main

import (
    "fmt"
    "runtime"
    "sync"
    "sync/atomic"
    "time"
    "unsafe"
)

// PART 1: Data Race Demonstration
// A data race occurs when two or more goroutines access the same memory location
// concurrently, and at least one of the accesses is a write

// UNSAFE: Counter with data race
type UnsafeCounter struct {
    value int64
}

func (c *UnsafeCounter) Increment() {
    c.value++ // DATA RACE! Read-modify-write is not atomic
}

func (c *UnsafeCounter) Value() int64 {
    return c.value // DATA RACE! Concurrent read while others write
}

// PART 2: Cache Coherency and False Sharing
// False sharing occurs when different goroutines access different variables
// that happen to be on the same cache line

// BAD: False sharing example - variables on same cache line
type FalseSharingExample struct {
    counter1 int64 // These will likely be on the same cache line
    counter2 int64 // causing false sharing between goroutines
    counter3 int64
    counter4 int64
}

// GOOD: Padding to prevent false sharing
type CacheLinePaddedCounters struct {
    counter1 int64
    _        [7]int64 // Padding to fill cache line (64 bytes on most systems)
    counter2 int64
    _        [7]int64 // Padding
    counter3 int64
    _        [7]int64 // Padding
    counter4 int64
}

// AutomationTaskTracker demonstrates cache coherency issues
type AutomationTaskTracker struct {
    // Bad design - false sharing likely
    BadCounters struct {
        fileProcessed   int64
        emailsSent      int64
        dbUpdates       int64
        reportsGenerated int64
    }

    // Good design - padded to prevent false sharing
    GoodCounters struct {
        fileProcessed   int64
        _               [7]int64 // Cache line padding
        emailsSent      int64
        _               [7]int64
        dbUpdates       int64
        _               [7]int64
        reportsGenerated int64
        _               [7]int64
    }
}

// PART 3: Race Detection and Analysis
func demonstrateDataRaces() {
    fmt.Println("=== Data Race Demonstration ===")
    fmt.Printf("GOMAXPROCS: %d\n", runtime.GOMAXPROCS(0))
    fmt.Printf("NumCPU: %d\n", runtime.NumCPU())

    const numGoroutines = 100
    const incrementsPerGoroutine = 1000
    expectedValue := int64(numGoroutines * incrementsPerGoroutine)

    // Test 1: Unsafe counter with data races
    fmt.Println("\n--- Test 1: Unsafe Counter (Data Races) ---")
    unsafeCounter := &UnsafeCounter{}

    var wg sync.WaitGroup
    start := time.Now()

    for i := 0; i < numGoroutines; i++ {
        wg.Add(1)
        go func() {
            defer wg.Done()
            for j := 0; j < incrementsPerGoroutine; j++ {
                unsafeCounter.Increment() // RACE CONDITION!
            }
        }()
    }

    wg.Wait()
    unsafeDuration := time.Since(start)
    actualValue := unsafeCounter.Value()

    fmt.Printf("Expected: %d\n", expectedValue)
    fmt.Printf("Actual: %d\n", actualValue)
    fmt.Printf("Correct: %t\n", expectedValue == actualValue)
    fmt.Printf("Lost increments: %d\n", expectedValue - actualValue)
    fmt.Printf("Duration: %v\n", unsafeDuration)

    if expectedValue != actualValue {
        fmt.Printf("⚙️ ï¸  DATA RACE DETECTED: Lost %d increments due to race conditions!\n",
            expectedValue - actualValue)
    }
}

// PART 4: False Sharing Performance Impact
func demonstrateFalseSharing() {
    fmt.Println("\n=== False Sharing Demonstration ===")

    const numGoroutines = 4
    const incrementsPerGoroutine = 10000000

    // Test with false sharing
    fmt.Println("\n--- Test with False Sharing ---")
    falseSharingCounters := &FalseSharingExample{}

    var wg sync.WaitGroup
    start := time.Now()

    // Each goroutine works on a different counter, but they're on the same cache line
    for i := 0; i < numGoroutines; i++ {
        wg.Add(1)
        go func(goroutineID int) {
            defer wg.Done()

            for j := 0; j < incrementsPerGoroutine; j++ {
                switch goroutineID {
                case 0:
                    atomic.AddInt64(&falseSharingCounters.counter1, 1)
                case 1:
                    atomic.AddInt64(&falseSharingCounters.counter2, 1)
                case 2:
                    atomic.AddInt64(&falseSharingCounters.counter3, 1)
                case 3:
                    atomic.AddInt64(&falseSharingCounters.counter4, 1)
                }
            }
        }(i)
    }

    wg.Wait()
    falseSharingDuration := time.Since(start)

    fmt.Printf("Duration with false sharing: %v\n", falseSharingDuration)
    fmt.Printf("Counter1: %d, Counter2: %d, Counter3: %d, Counter4: %d\n",
        falseSharingCounters.counter1, falseSharingCounters.counter2,
        falseSharingCounters.counter3, falseSharingCounters.counter4)

    // Test without false sharing (padded)
    fmt.Println("\n--- Test without False Sharing (Padded) ---")
    paddedCounters := &CacheLinePaddedCounters{}

    start = time.Now()

    for i := 0; i < numGoroutines; i++ {
        wg.Add(1)
        go func(goroutineID int) {
            defer wg.Done()

            for j := 0; j < incrementsPerGoroutine; j++ {
                switch goroutineID {
                case 0:
                    atomic.AddInt64(&paddedCounters.counter1, 1)
                case 1:
                    atomic.AddInt64(&paddedCounters.counter2, 1)
                case 2:
                    atomic.AddInt64(&paddedCounters.counter3, 1)
                case 3:
                    atomic.AddInt64(&paddedCounters.counter4, 1)
                }
            }
        }(i)
    }

    wg.Wait()
    paddedDuration := time.Since(start)

    fmt.Printf("Duration without false sharing: %v\n", paddedDuration)
    fmt.Printf("Counter1: %d, Counter2: %d, Counter3: %d, Counter4: %d\n",
        paddedCounters.counter1, paddedCounters.counter2,
        paddedCounters.counter3, paddedCounters.counter4)

    if falseSharingDuration > paddedDuration {
        improvement := float64(falseSharingDuration) / float64(paddedDuration)
        fmt.Printf("🔥š€ Performance improvement: %.2fx faster without false sharing\n", improvement)
    }
}

// PART 5: Cache Line Analysis
func analyzeCacheLines() {
    fmt.Println("\n=== Cache Line Analysis ===")

    var fs FalseSharingExample
    var padded CacheLinePaddedCounters

    fmt.Printf("FalseSharingExample size: %d bytes\n", unsafe.Sizeof(fs))
    fmt.Printf("CacheLinePaddedCounters size: %d bytes\n", unsafe.Sizeof(padded))

    // Show memory addresses to understand cache line alignment
    fmt.Printf("\nFalseSharingExample field addresses:\n")
    fmt.Printf("  counter1: %p\n", &fs.counter1)
    fmt.Printf("  counter2: %p (offset: %d)\n", &fs.counter2,
        uintptr(unsafe.Pointer(&fs.counter2)) - uintptr(unsafe.Pointer(&fs.counter1)))
    fmt.Printf("  counter3: %p (offset: %d)\n", &fs.counter3,
        uintptr(unsafe.Pointer(&fs.counter3)) - uintptr(unsafe.Pointer(&fs.counter1)))
    fmt.Printf("  counter4: %p (offset: %d)\n", &fs.counter4,
        uintptr(unsafe.Pointer(&fs.counter4)) - uintptr(unsafe.Pointer(&fs.counter1)))

    fmt.Printf("\nCacheLinePaddedCounters field addresses:\n")
    fmt.Printf("  counter1: %p\n", &padded.counter1)
    fmt.Printf("  counter2: %p (offset: %d)\n", &padded.counter2,
        uintptr(unsafe.Pointer(&padded.counter2)) - uintptr(unsafe.Pointer(&padded.counter1)))
    fmt.Printf("  counter3: %p (offset: %d)\n", &padded.counter3,
        uintptr(unsafe.Pointer(&padded.counter3)) - uintptr(unsafe.Pointer(&padded.counter1)))
    fmt.Printf("  counter4: %p (offset: %d)\n", &padded.counter4,
        uintptr(unsafe.Pointer(&padded.counter4)) - uintptr(unsafe.Pointer(&padded.counter1)))
}

func main() {
    fmt.Println("=== Data Races and Cache Coherency Demo ===")
    fmt.Println("⚙️ ï¸  Run with 'go run -race main.go' to detect data races")

    // Demonstrate data races
    demonstrateDataRaces()

    // Demonstrate false sharing performance impact
    demonstrateFalseSharing()

    // Analyze cache line layout
    analyzeCacheLines()

    fmt.Println("\n=== Key Insights ===")
    fmt.Println("✅… DATA RACES: Occur when concurrent access includes writes")
    fmt.Println("✅… RACE DETECTOR: Use -race flag to detect races during development")
    fmt.Println("✅… FALSE SHARING: Different variables on same cache line cause contention")
    fmt.Println("✅… CACHE LINES: Typically 64 bytes on modern processors")
    fmt.Println("✅… PADDING: Add padding to separate frequently accessed variables")
    fmt.Println("✅… PERFORMANCE: False sharing can significantly impact concurrent performance")
}
```

**Hands-on Exercise 2: Atomic Operations and Mutexes**:

Mastering atomic operations and mutex-based synchronization:

```go
// Atomic operations and mutex synchronization in automation systems
package main

import (
    "fmt"
    "runtime"
    "sync"
    "sync/atomic"
    "time"
)

// PART 1: Atomic Operations
// Atomic operations provide lock-free synchronization for simple operations

// SAFE: Counter using atomic operations
type AtomicCounter struct {
    value int64
}

func (c *AtomicCounter) Increment() {
    atomic.AddInt64(&c.value, 1)
}

func (c *AtomicCounter) Add(delta int64) {
    atomic.AddInt64(&c.value, delta)
}

func (c *AtomicCounter) Value() int64 {
    return atomic.LoadInt64(&c.value)
}

func (c *AtomicCounter) CompareAndSwap(old, new int64) bool {
    return atomic.CompareAndSwapInt64(&c.value, old, new)
}

// SAFE: Counter using mutex
type MutexCounter struct {
    mu    sync.Mutex
    value int64
}

func (c *MutexCounter) Increment() {
    c.mu.Lock()
    c.value++
    c.mu.Unlock()
}

func (c *MutexCounter) Add(delta int64) {
    c.mu.Lock()
    c.value += delta
    c.mu.Unlock()
}

func (c *MutexCounter) Value() int64 {
    c.mu.Lock()
    defer c.mu.Unlock()
    return c.value
}

// PART 2: Automation Metrics with Different Synchronization Approaches
type AutomationMetrics struct {
    // Atomic counters for simple metrics (best performance)
    tasksProcessed   int64
    errorsOccurred   int64
    bytesProcessed   int64
    requestsReceived int64

    // Mutex-protected complex state
    mu           sync.Mutex
    tasksByType  map[string]int64
    lastActivity time.Time
    activeWorkers map[int]bool

    // Read-heavy data with RWMutex (optimized for frequent reads)
    configMu sync.RWMutex
    config   map[string]string
}

func NewAutomationMetrics() *AutomationMetrics {
    return &AutomationMetrics{
        tasksByType:   make(map[string]int64),
        activeWorkers: make(map[int]bool),
        config:        make(map[string]string),
        lastActivity:  time.Now(),
    }
}

// PART 3: Atomic Operations for Simple Counters
func (am *AutomationMetrics) IncrementTasksProcessed() {
    atomic.AddInt64(&am.tasksProcessed, 1)
}

func (am *AutomationMetrics) IncrementErrors() {
    atomic.AddInt64(&am.errorsOccurred, 1)
}

func (am *AutomationMetrics) AddBytesProcessed(bytes int64) {
    atomic.AddInt64(&am.bytesProcessed, bytes)
}

func (am *AutomationMetrics) IncrementRequests() {
    atomic.AddInt64(&am.requestsReceived, 1)
}

func (am *AutomationMetrics) GetTasksProcessed() int64 {
    return atomic.LoadInt64(&am.tasksProcessed)
}

func (am *AutomationMetrics) GetErrors() int64 {
    return atomic.LoadInt64(&am.errorsOccurred)
}

func (am *AutomationMetrics) GetBytesProcessed() int64 {
    return atomic.LoadInt64(&am.bytesProcessed)
}

func (am *AutomationMetrics) GetRequests() int64 {
    return atomic.LoadInt64(&am.requestsReceived)
}

// PART 4: Mutex for Complex State Modifications
func (am *AutomationMetrics) RecordTaskByType(taskType string) {
    am.mu.Lock()
    am.tasksByType[taskType]++
    am.lastActivity = time.Now()
    am.mu.Unlock()
}

func (am *AutomationMetrics) RegisterWorker(workerID int) {
    am.mu.Lock()
    am.activeWorkers[workerID] = true
    am.mu.Unlock()
}

func (am *AutomationMetrics) UnregisterWorker(workerID int) {
    am.mu.Lock()
    delete(am.activeWorkers, workerID)
    am.mu.Unlock()
}

func (am *AutomationMetrics) GetTasksByType() map[string]int64 {
    am.mu.Lock()
    defer am.mu.Unlock()

    // Return a copy to prevent races
    result := make(map[string]int64)
    for k, v := range am.tasksByType {
        result[k] = v
    }
    return result
}

func (am *AutomationMetrics) GetActiveWorkerCount() int {
    am.mu.Lock()
    defer am.mu.Unlock()
    return len(am.activeWorkers)
}

func (am *AutomationMetrics) GetLastActivity() time.Time {
    am.mu.Lock()
    defer am.mu.Unlock()
    return am.lastActivity
}

// PART 5: RWMutex for Read-Heavy Configuration
func (am *AutomationMetrics) UpdateConfig(key, value string) {
    am.configMu.Lock()
    am.config[key] = value
    am.configMu.Unlock()
}

func (am *AutomationMetrics) GetConfig(key string) (string, bool) {
    am.configMu.RLock()
    defer am.configMu.RUnlock()
    value, exists := am.config[key]
    return value, exists
}

func (am *AutomationMetrics) GetAllConfig() map[string]string {
    am.configMu.RLock()
    defer am.configMu.RUnlock()

    result := make(map[string]string)
    for k, v := range am.config {
        result[k] = v
    }
    return result
}

// PART 6: Performance Comparison
func compareAtomicVsMutex() {
    fmt.Println("=== Atomic vs Mutex Performance Comparison ===")

    const numGoroutines = 100
    const incrementsPerGoroutine = 10000

    // Test atomic counter
    fmt.Println("\n--- Atomic Counter Performance ---")
    atomicCounter := &AtomicCounter{}

    var wg sync.WaitGroup
    start := time.Now()

    for i := 0; i < numGoroutines; i++ {
        wg.Add(1)
        go func() {
            defer wg.Done()
            for j := 0; j < incrementsPerGoroutine; j++ {
                atomicCounter.Increment()
            }
        }()
    }

    wg.Wait()
    atomicDuration := time.Since(start)
    atomicValue := atomicCounter.Value()

    fmt.Printf("Atomic - Value: %d, Duration: %v\n", atomicValue, atomicDuration)

    // Test mutex counter
    fmt.Println("\n--- Mutex Counter Performance ---")
    mutexCounter := &MutexCounter{}

    start = time.Now()

    for i := 0; i < numGoroutines; i++ {
        wg.Add(1)
        go func() {
            defer wg.Done()
            for j := 0; j < incrementsPerGoroutine; j++ {
                mutexCounter.Increment()
            }
        }()
    }

    wg.Wait()
    mutexDuration := time.Since(start)
    mutexValue := mutexCounter.Value()

    fmt.Printf("Mutex - Value: %d, Duration: %v\n", mutexValue, mutexDuration)

    // Performance comparison
    if atomicDuration < mutexDuration {
        speedup := float64(mutexDuration) / float64(atomicDuration)
        fmt.Printf("\n🔥š€ Atomic operations are %.2fx faster than mutex for simple counters\n", speedup)
    }
}

// PART 7: Automation Workload Simulation
func simulateAutomationWorkload() {
    fmt.Println("\n=== Automation Workload Simulation ===")

    metrics := NewAutomationMetrics()

    // Set initial configuration
    metrics.UpdateConfig("max_workers", "10")
    metrics.UpdateConfig("timeout", "30s")
    metrics.UpdateConfig("retry_count", "3")
    metrics.UpdateConfig("batch_size", "100")

    const numWorkers = 20
    const tasksPerWorker = 100

    var wg sync.WaitGroup

    // Start workers that process tasks
    for i := 0; i < numWorkers; i++ {
        wg.Add(1)
        go func(workerID int) {
            defer wg.Done()

            // Register worker
            metrics.RegisterWorker(workerID)
            defer metrics.UnregisterWorker(workerID)

            taskTypes := []string{"file_process", "email_send", "db_update", "report_gen", "data_sync"}

            for j := 0; j < tasksPerWorker; j++ {
                // Simulate task processing
                taskType := taskTypes[j%len(taskTypes)]

                // Record metrics using different synchronization methods
                metrics.IncrementTasksProcessed()    // Atomic
                metrics.IncrementRequests()          // Atomic
                metrics.AddBytesProcessed(1024)      // Atomic
                metrics.RecordTaskByType(taskType)   // Mutex

                // Simulate occasional errors
                if j%20 == 0 { // 5% error rate
                    metrics.IncrementErrors() // Atomic
                }

                // Simulate work
                time.Sleep(time.Millisecond)

                // Occasionally read configuration (read-heavy workload)
                if j%10 == 0 {
                    if timeout, exists := metrics.GetConfig("timeout"); exists {
                        _ = timeout // Use the config value
                    }
                    if batchSize, exists := metrics.GetConfig("batch_size"); exists {
                        _ = batchSize
                    }
                }
            }
        }(i)
    }

    // Start monitoring goroutine
    wg.Add(1)
    go func() {
        defer wg.Done()

        ticker := time.NewTicker(200 * time.Millisecond)
        defer ticker.Stop()

        for i := 0; i < 10; i++ {
            <-ticker.C

            processed := metrics.GetTasksProcessed()
            errors := metrics.GetErrors()
            bytes := metrics.GetBytesProcessed()
            requests := metrics.GetRequests()
            activeWorkers := metrics.GetActiveWorkerCount()
            lastActivity := metrics.GetLastActivity()

            fmt.Printf("Monitor: Tasks=%d, Errors=%d, Bytes=%d, Requests=%d, Workers=%d, LastActivity=%v ago\n",
                processed, errors, bytes, requests, activeWorkers, time.Since(lastActivity))
        }
    }()

    wg.Wait()

    // Final report
    fmt.Println("\nFinal Metrics:")
    fmt.Printf("Total tasks processed: %d\n", metrics.GetTasksProcessed())
    fmt.Printf("Total errors: %d\n", metrics.GetErrors())
    fmt.Printf("Total bytes processed: %d\n", metrics.GetBytesProcessed())
    fmt.Printf("Total requests: %d\n", metrics.GetRequests())

    tasksByType := metrics.GetTasksByType()
    fmt.Println("Tasks by type:")
    for taskType, count := range tasksByType {
        fmt.Printf("  %s: %d\n", taskType, count)
    }

    config := metrics.GetAllConfig()
    fmt.Println("Configuration:")
    for key, value := range config {
        fmt.Printf("  %s: %s\n", key, value)
    }
}

func main() {
    fmt.Printf("Starting with GOMAXPROCS=%d\n", runtime.GOMAXPROCS(0))

    // Compare atomic vs mutex performance
    compareAtomicVsMutex()

    // Simulate realistic automation workload
    simulateAutomationWorkload()

    fmt.Println("\n=== Synchronization Best Practices ===")
    fmt.Println("✅… ATOMIC: Use for simple counters and flags (fastest)")
    fmt.Println("✅… MUTEX: Use for complex state modifications")
    fmt.Println("✅… RWMUTEX: Use for read-heavy workloads")
    fmt.Println("✅… COPY DATA: Return copies from getters to prevent races")
    fmt.Println("✅… DEFER UNLOCK: Always defer mutex unlocks")
    fmt.Println("✅… MINIMIZE CRITICAL SECTIONS: Keep locked sections small")
}
```

**Hands-on Exercise 3: Advanced Synchronization and Map Safety**:

Covering read/write mutexes, map mutations, and interface-based race conditions:

```go
// Advanced synchronization patterns: RWMutex, map safety, and interface races
package main

import (
    "fmt"
    "math/rand"
    "runtime"
    "sync"
    "sync/atomic"
    "time"
)

// PART 1: Read/Write Mutex Optimization
// RWMutex allows multiple concurrent readers but exclusive writers

// ConfigurationManager demonstrates RWMutex usage for read-heavy workloads
type ConfigurationManager struct {
    mu     sync.RWMutex
    config map[string]string

    // Metrics to track read/write operations
    readCount  int64
    writeCount int64
}

func NewConfigurationManager() *ConfigurationManager {
    return &ConfigurationManager{
        config: make(map[string]string),
    }
}

// Write operations require exclusive access
func (cm *ConfigurationManager) Set(key, value string) {
    cm.mu.Lock()
    defer cm.mu.Unlock()

    cm.config[key] = value
    atomic.AddInt64(&cm.writeCount, 1)
}

func (cm *ConfigurationManager) Delete(key string) {
    cm.mu.Lock()
    defer cm.mu.Unlock()

    delete(cm.config, key)
    atomic.AddInt64(&cm.writeCount, 1)
}

// Read operations can be concurrent
func (cm *ConfigurationManager) Get(key string) (string, bool) {
    cm.mu.RLock()
    defer cm.mu.RUnlock()

    value, exists := cm.config[key]
    atomic.AddInt64(&cm.readCount, 1)
    return value, exists
}

func (cm *ConfigurationManager) GetAll() map[string]string {
    cm.mu.RLock()
    defer cm.mu.RUnlock()

    // Return a copy to prevent races
    result := make(map[string]string)
    for k, v := range cm.config {
        result[k] = v
    }
    atomic.AddInt64(&cm.readCount, 1)
    return result
}

func (cm *ConfigurationManager) GetStats() (int64, int64) {
    return atomic.LoadInt64(&cm.readCount), atomic.LoadInt64(&cm.writeCount)
}

// PART 2: Map Mutations and Race Conditions
// Maps are not safe for concurrent access - demonstrate the problem and solutions

// UNSAFE: Concurrent map access without synchronization
type UnsafeTaskRegistry struct {
    tasks map[string]string // DANGEROUS: No synchronization!
}

func NewUnsafeTaskRegistry() *UnsafeTaskRegistry {
    return &UnsafeTaskRegistry{
        tasks: make(map[string]string),
    }
}

func (tr *UnsafeTaskRegistry) Register(id, name string) {
    tr.tasks[id] = name // RACE CONDITION!
}

func (tr *UnsafeTaskRegistry) Get(id string) (string, bool) {
    name, exists := tr.tasks[id] // RACE CONDITION!
    return name, exists
}

// SAFE: Synchronized map access
type SafeTaskRegistry struct {
    mu    sync.RWMutex
    tasks map[string]string
}

func NewSafeTaskRegistry() *SafeTaskRegistry {
    return &SafeTaskRegistry{
        tasks: make(map[string]string),
    }
}

func (tr *SafeTaskRegistry) Register(id, name string) {
    tr.mu.Lock()
    tr.tasks[id] = name
    tr.mu.Unlock()
}

func (tr *SafeTaskRegistry) Get(id string) (string, bool) {
    tr.mu.RLock()
    defer tr.mu.RUnlock()
    name, exists := tr.tasks[id]
    return name, exists
}

func (tr *SafeTaskRegistry) GetAll() map[string]string {
    tr.mu.RLock()
    defer tr.mu.RUnlock()

    result := make(map[string]string)
    for k, v := range tr.tasks {
        result[k] = v
    }
    return result
}

// PART 3: Interface-Based Race Conditions
// Race conditions can occur when interfaces hide mutable state

// TaskProcessor interface
type TaskProcessor interface {
    Process(taskID string) error
    GetStats() ProcessorStats
}

type ProcessorStats struct {
    TasksProcessed int64
    ErrorsOccurred int64
    LastProcessed  time.Time
}

// UNSAFE: Interface implementation with race conditions
type UnsafeProcessor struct {
    stats ProcessorStats // RACE: Concurrent access to struct fields
}

func (p *UnsafeProcessor) Process(taskID string) error {
    // Simulate processing
    time.Sleep(time.Millisecond)

    // RACE CONDITION: Multiple goroutines modifying stats
    p.stats.TasksProcessed++
    p.stats.LastProcessed = time.Now()

    // Simulate occasional errors
    if rand.Float32() < 0.1 {
        p.stats.ErrorsOccurred++
        return fmt.Errorf("processing failed for task %s", taskID)
    }

    return nil
}

func (p *UnsafeProcessor) GetStats() ProcessorStats {
    return p.stats // RACE: Reading while others write
}

// SAFE: Interface implementation with proper synchronization
type SafeProcessor struct {
    mu    sync.RWMutex
    stats ProcessorStats
}

func (p *SafeProcessor) Process(taskID string) error {
    // Simulate processing
    time.Sleep(time.Millisecond)

    p.mu.Lock()
    p.stats.TasksProcessed++
    p.stats.LastProcessed = time.Now()

    // Simulate occasional errors
    if rand.Float32() < 0.1 {
        p.stats.ErrorsOccurred++
        p.mu.Unlock()
        return fmt.Errorf("processing failed for task %s", taskID)
    }
    p.mu.Unlock()

    return nil
}

func (p *SafeProcessor) GetStats() ProcessorStats {
    p.mu.RLock()
    defer p.mu.RUnlock()
    return p.stats
}

// PART 4: Demonstration Functions

func demonstrateRWMutexPerformance() {
    fmt.Println("=== RWMutex Performance Demonstration ===")

    cm := NewConfigurationManager()

    // Initialize some configuration
    cm.Set("database_url", "postgres://localhost:5432/automation")
    cm.Set("api_timeout", "30s")
    cm.Set("max_workers", "10")
    cm.Set("log_level", "info")

    const numReaders = 50
    const numWriters = 5
    const operationsPerGoroutine = 1000

    var wg sync.WaitGroup
    start := time.Now()

    // Start many readers (read-heavy workload)
    for i := 0; i < numReaders; i++ {
        wg.Add(1)
        go func(readerID int) {
            defer wg.Done()

            keys := []string{"database_url", "api_timeout", "max_workers", "log_level"}

            for j := 0; j < operationsPerGoroutine; j++ {
                key := keys[j%len(keys)]
                if value, exists := cm.Get(key); exists {
                    _ = value // Use the value
                }

                // Occasionally get all config
                if j%100 == 0 {
                    _ = cm.GetAll()
                }
            }
        }(i)
    }

    // Start fewer writers
    for i := 0; i < numWriters; i++ {
        wg.Add(1)
        go func(writerID int) {
            defer wg.Done()

            for j := 0; j < operationsPerGoroutine/10; j++ { // Fewer write operations
                key := fmt.Sprintf("dynamic_config_%d_%d", writerID, j)
                value := fmt.Sprintf("value_%d", j)
                cm.Set(key, value)

                // Occasionally delete
                if j%10 == 0 && j > 0 {
                    deleteKey := fmt.Sprintf("dynamic_config_%d_%d", writerID, j-1)
                    cm.Delete(deleteKey)
                }
            }
        }(i)
    }

    wg.Wait()
    duration := time.Since(start)

    reads, writes := cm.GetStats()
    fmt.Printf("Duration: %v\n", duration)
    fmt.Printf("Read operations: %d\n", reads)
    fmt.Printf("Write operations: %d\n", writes)
    fmt.Printf("Read/Write ratio: %.2f:1\n", float64(reads)/float64(writes))
    fmt.Printf("Operations per second: %.0f\n", float64(reads+writes)/duration.Seconds())
}

func demonstrateMapRaces() {
    fmt.Println("\n=== Map Race Conditions Demonstration ===")

    const numGoroutines = 20
    const operationsPerGoroutine = 100

    // Test unsafe map (will cause races)
    fmt.Println("\n--- Unsafe Map (Race Conditions) ---")
    unsafeRegistry := NewUnsafeTaskRegistry()

    var wg sync.WaitGroup
    start := time.Now()

    for i := 0; i < numGoroutines; i++ {
        wg.Add(1)
        go func(goroutineID int) {
            defer wg.Done()

            for j := 0; j < operationsPerGoroutine; j++ {
                taskID := fmt.Sprintf("task_%d_%d", goroutineID, j)
                taskName := fmt.Sprintf("Task %d-%d", goroutineID, j)

                // This will cause race conditions
                unsafeRegistry.Register(taskID, taskName)

                // This will also cause race conditions
                if j%10 == 0 {
                    _, _ = unsafeRegistry.Get(taskID)
                }
            }
        }(i)
    }

    wg.Wait()
    unsafeDuration := time.Since(start)
    fmt.Printf("Unsafe map duration: %v (⚙️ ï¸  Contains race conditions!)\n", unsafeDuration)

    // Test safe map
    fmt.Println("\n--- Safe Map (Synchronized) ---")
    safeRegistry := NewSafeTaskRegistry()

    start = time.Now()

    for i := 0; i < numGoroutines; i++ {
        wg.Add(1)
        go func(goroutineID int) {
            defer wg.Done()

            for j := 0; j < operationsPerGoroutine; j++ {
                taskID := fmt.Sprintf("task_%d_%d", goroutineID, j)
                taskName := fmt.Sprintf("Task %d-%d", goroutineID, j)

                safeRegistry.Register(taskID, taskName)

                if j%10 == 0 {
                    _, _ = safeRegistry.Get(taskID)
                }
            }
        }(i)
    }

    wg.Wait()
    safeDuration := time.Since(start)

    allTasks := safeRegistry.GetAll()
    fmt.Printf("Safe map duration: %v\n", safeDuration)
    fmt.Printf("Total tasks registered: %d\n", len(allTasks))

    if safeDuration > unsafeDuration {
        overhead := float64(safeDuration) / float64(unsafeDuration)
        fmt.Printf("Synchronization overhead: %.2fx (worth it for correctness!)\n", overhead)
    }
}

func demonstrateInterfaceRaces() {
    fmt.Println("\n=== Interface-Based Race Conditions ===")

    const numGoroutines = 20
    const tasksPerGoroutine = 100

    // Test unsafe processor
    fmt.Println("\n--- Unsafe Processor (Interface Races) ---")
    var unsafeProcessor TaskProcessor = &UnsafeProcessor{}

    var wg sync.WaitGroup
    start := time.Now()

    for i := 0; i < numGoroutines; i++ {
        wg.Add(1)
        go func(goroutineID int) {
            defer wg.Done()

            for j := 0; j < tasksPerGoroutine; j++ {
                taskID := fmt.Sprintf("task_%d_%d", goroutineID, j)
                _ = unsafeProcessor.Process(taskID)

                // Occasionally read stats
                if j%20 == 0 {
                    _ = unsafeProcessor.GetStats()
                }
            }
        }(i)
    }

    wg.Wait()
    unsafeDuration := time.Since(start)
    unsafeStats := unsafeProcessor.GetStats()

    fmt.Printf("Unsafe processor duration: %v\n", unsafeDuration)
    fmt.Printf("Tasks processed: %d (⚙️ ï¸  May be incorrect due to races!)\n", unsafeStats.TasksProcessed)
    fmt.Printf("Errors occurred: %d\n", unsafeStats.ErrorsOccurred)

    // Test safe processor
    fmt.Println("\n--- Safe Processor (Synchronized) ---")
    var safeProcessor TaskProcessor = &SafeProcessor{}

    start = time.Now()

    for i := 0; i < numGoroutines; i++ {
        wg.Add(1)
        go func(goroutineID int) {
            defer wg.Done()

            for j := 0; j < tasksPerGoroutine; j++ {
                taskID := fmt.Sprintf("task_%d_%d", goroutineID, j)
                _ = safeProcessor.Process(taskID)

                if j%20 == 0 {
                    _ = safeProcessor.GetStats()
                }
            }
        }(i)
    }

    wg.Wait()
    safeDuration := time.Since(start)
    safeStats := safeProcessor.GetStats()

    fmt.Printf("Safe processor duration: %v\n", safeDuration)
    fmt.Printf("Tasks processed: %d (✅… Correct!)\n", safeStats.TasksProcessed)
    fmt.Printf("Errors occurred: %d\n", safeStats.ErrorsOccurred)

    expectedTasks := int64(numGoroutines * tasksPerGoroutine)
    if safeStats.TasksProcessed == expectedTasks {
        fmt.Printf("✅… All tasks accounted for: %d/%d\n", safeStats.TasksProcessed, expectedTasks)
    } else {
        fmt.Printf("❌ Task count mismatch: %d/%d\n", safeStats.TasksProcessed, expectedTasks)
    }
}

func main() {
    fmt.Printf("Starting with GOMAXPROCS=%d\n", runtime.GOMAXPROCS(0))
    fmt.Println("⚙️ ï¸  Run with 'go run -race main.go' to detect race conditions")

    // Demonstrate RWMutex performance benefits
    demonstrateRWMutexPerformance()

    // Demonstrate map race conditions
    demonstrateMapRaces()

    // Demonstrate interface-based race conditions
    demonstrateInterfaceRaces()

    fmt.Println("\n=== Advanced Synchronization Best Practices ===")
    fmt.Println("✅… RWMUTEX: Use for read-heavy workloads (many readers, few writers)")
    fmt.Println("✅… MAP SAFETY: Maps are not safe for concurrent access")
    fmt.Println("✅… INTERFACE RACES: Hidden mutable state in interfaces can cause races")
    fmt.Println("✅… COPY RETURNS: Return copies of complex data to prevent races")
    fmt.Println("✅… RACE DETECTOR: Always test with -race flag during development")
    fmt.Println("✅… ATOMIC COUNTERS: Use atomic operations for simple metrics")
}
```

**Prerequisites**: Module 28
