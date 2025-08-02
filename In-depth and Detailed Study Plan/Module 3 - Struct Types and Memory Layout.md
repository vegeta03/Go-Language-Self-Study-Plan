# Module 3: Struct Types and Memory Layout

**Duration**: 1 hour (45 min theory + 15 min hands-on)

**Learning Objectives**:

- Master user-defined struct types as Go's primary data organization mechanism
- Understand memory layout, alignment, and padding in structs
- Learn the critical difference between named and literal (anonymous) types
- Understand Go's stance on implicit vs explicit conversion
- Apply struct design principles to automation and integration like scenarios
- Optimize for correctness first, performance second

**Videos Covered**:

- 2.3 Struct Types (0:23:27)

**Key Concepts**:

**User-Defined Types and Structs**:

- Structs are Go's primary mechanism for creating composite types
- Go uses `struct` keyword instead of `class` - focuses on data, not behavior
- Struct types define the shape and layout of data in memory
- Construction tells you nothing about cost - sharing tells you everything

**Memory Layout and Alignment**:

- Hardware requires data alignment for efficient memory access
- Alignment prevents values from crossing word boundaries
- Padding is inserted to maintain proper alignment
- 1-byte values (bool): can be placed anywhere
- 2-byte values (int16): must be 2-byte aligned (addresses 0, 2, 4, 6...)
- 4-byte values (int32, float32): must be 4-byte aligned (addresses 0, 4, 8...)
- 8-byte values (int64, float64): must be 8-byte aligned (addresses 0, 8, 16...)

**Struct Size Calculation**:

- Struct size = sum of field sizes + padding
- Padding is added between fields and at the end for alignment
- Struct alignment is determined by its largest field
- Understanding struct size is crucial for understanding memory cost

**Named vs Literal Types**:

- Named types: explicitly declared with `type` keyword
- Literal types: anonymous, defined inline
- Named types have no implicit conversion - explicit conversion required
- Literal types allow more flexible assignment compatibility
- This design prevents accidental type mixing and improves code safety

**Optimization Philosophy**:

- Always optimize for correctness first
- Don't prematurely optimize struct layout for memory
- Only reorder fields for memory efficiency after profiling shows it's necessary
- Readability and logical grouping should drive field ordering by default

**Hands-on Exercise 1: Memory Layout and Alignment**:

```go
// Memory layout analysis for automation data structures
package main

import (
    "fmt"
    "unsafe"
)

// Example struct with padding
type AutomationTask struct {
    ID          int64     // 8 bytes, 8-byte aligned
    Name        string    // 16 bytes (2 words), 8-byte aligned
    Priority    int32     // 4 bytes, 4-byte aligned
    Enabled     bool      // 1 byte, 1-byte aligned
    RetryCount  int16     // 2 bytes, 2-byte aligned
    Timeout     float64   // 8 bytes, 8-byte aligned
}

// Memory-optimized version (largest to smallest)
type OptimizedTask struct {
    ID         int64     // 8 bytes
    Timeout    float64   // 8 bytes
    Name       string    // 16 bytes
    Priority   int32     // 4 bytes
    RetryCount int16     // 2 bytes
    Enabled    bool      // 1 byte
    // 1 byte padding at end for 8-byte struct alignment
}

// Poor layout example
type PoorTask struct {
    Enabled    bool      // 1 byte
    // 7 bytes padding
    ID         int64     // 8 bytes
    Priority   int32     // 4 bytes
    // 4 bytes padding
    Timeout    float64   // 8 bytes
    RetryCount int16     // 2 bytes
    // 6 bytes padding
    Name       string    // 16 bytes
}

func main() {
    analyzeStructSizes()
    demonstrateAlignment()
    showFieldOffsets()
}

func analyzeStructSizes() {
    fmt.Println("=== Struct Size Analysis ===")

    var regular AutomationTask
    var optimized OptimizedTask
    var poor PoorTask

    fmt.Printf("AutomationTask size: %d bytes\n", unsafe.Sizeof(regular))
    fmt.Printf("OptimizedTask size: %d bytes\n", unsafe.Sizeof(optimized))
    fmt.Printf("PoorTask size: %d bytes\n", unsafe.Sizeof(poor))

    // Calculate theoretical minimum (sum of field sizes)
    theoreticalMin := unsafe.Sizeof(int64(0)) + unsafe.Sizeof("") +
                     unsafe.Sizeof(int32(0)) + unsafe.Sizeof(bool(false)) +
                     unsafe.Sizeof(int16(0)) + unsafe.Sizeof(float64(0))

    fmt.Printf("Theoretical minimum: %d bytes\n", theoreticalMin)
    fmt.Printf("Regular padding: %d bytes\n", unsafe.Sizeof(regular) - theoreticalMin)
    fmt.Printf("Poor padding: %d bytes\n", unsafe.Sizeof(poor) - theoreticalMin)
    fmt.Println()
}

func demonstrateAlignment() {
    fmt.Println("=== Alignment Requirements ===")

    var b bool
    var i16 int16
    var i32 int32
    var i64 int64
    var f32 float32
    var f64 float64
    var s string

    fmt.Printf("bool alignment: %d bytes\n", unsafe.Alignof(b))
    fmt.Printf("int16 alignment: %d bytes\n", unsafe.Alignof(i16))
    fmt.Printf("int32 alignment: %d bytes\n", unsafe.Alignof(i32))
    fmt.Printf("int64 alignment: %d bytes\n", unsafe.Alignof(i64))
    fmt.Printf("float32 alignment: %d bytes\n", unsafe.Alignof(f32))
    fmt.Printf("float64 alignment: %d bytes\n", unsafe.Alignof(f64))
    fmt.Printf("string alignment: %d bytes\n", unsafe.Alignof(s))
    fmt.Println()
}

func showFieldOffsets() {
    fmt.Println("=== Field Offsets in AutomationTask ===")

    var task AutomationTask

    fmt.Printf("ID offset: %d\n", unsafe.Offsetof(task.ID))
    fmt.Printf("Name offset: %d\n", unsafe.Offsetof(task.Name))
    fmt.Printf("Priority offset: %d\n", unsafe.Offsetof(task.Priority))
    fmt.Printf("Enabled offset: %d\n", unsafe.Offsetof(task.Enabled))
    fmt.Printf("RetryCount offset: %d\n", unsafe.Offsetof(task.RetryCount))
    fmt.Printf("Timeout offset: %d\n", unsafe.Offsetof(task.Timeout))
    fmt.Println()
}
```

**Hands-on Exercise 2: Named vs Literal Types**:

```go
// Demonstration of named vs literal types and conversion rules
package main

import "fmt"

// Named types for automation system
type ServiceID string
type TaskID string

// Both have identical underlying types but are different named types
type DatabaseConfig struct {
    Host     string
    Port     int
    Username string
    Password string
}

type CacheConfig struct {
    Host     string
    Port     int
    Username string
    Password string
}

func main() {
    demonstrateNamedTypeConversion()
    demonstrateLiteralTypeFlexibility()
    showImplicitConversionDanger()
}

func demonstrateNamedTypeConversion() {
    fmt.Println("=== Named Type Conversion ===")

    // Named types require explicit conversion even when identical
    var serviceID ServiceID = "web-service"
    var taskID TaskID = "task-001"

    fmt.Printf("ServiceID: %s\n", serviceID)
    fmt.Printf("TaskID: %s\n", taskID)

    // This would cause a compile error:
    // taskID = serviceID  // Cannot use serviceID (type ServiceID) as type TaskID

    // Explicit conversion is required:
    taskID = TaskID(serviceID)
    fmt.Printf("Converted TaskID: %s\n", taskID)

    // Same with struct types
    dbConfig := DatabaseConfig{
        Host:     "localhost",
        Port:     5432,
        Username: "admin",
        Password: "secret",
    }

    var cacheConfig CacheConfig
    // This would cause a compile error:
    // cacheConfig = dbConfig  // Cannot use dbConfig (type DatabaseConfig) as type CacheConfig

    // Explicit conversion required:
    cacheConfig = CacheConfig(dbConfig)
    fmt.Printf("Converted cache config: %+v\n", cacheConfig)
    fmt.Println()
}

func demonstrateLiteralTypeFlexibility() {
    fmt.Println("=== Literal Type Flexibility ===")

    // Literal (anonymous) struct
    config1 := struct {
        Host string
        Port int
    }{
        Host: "localhost",
        Port: 8080,
    }

    // Another variable with the same literal type structure
    var config2 struct {
        Host string
        Port int
    }

    // This assignment works - both are literal types with same structure
    config2 = config1
    fmt.Printf("Config1: %+v\n", config1)
    fmt.Printf("Config2: %+v\n", config2)

    // Can also assign to named type from literal type
    var dbConfig DatabaseConfig
    literalConfig := struct {
        Host     string
        Port     int
        Username string
        Password string
    }{
        Host:     "db.example.com",
        Port:     5432,
        Username: "user",
        Password: "pass",
    }

    // This works - literal to named type assignment is allowed
    dbConfig = DatabaseConfig(literalConfig)
    fmt.Printf("DB Config from literal: %+v\n", dbConfig)
    fmt.Println()
}

func showImplicitConversionDanger() {
    fmt.Println("=== Why No Implicit Conversion? ===")

    // This demonstrates why implicit conversion can be dangerous
    type UserID int
    type ProductID int

    var userID UserID = 12345
    var productID ProductID = 67890

    fmt.Printf("UserID: %d\n", userID)
    fmt.Printf("ProductID: %d\n", productID)

    // If Go allowed implicit conversion, this bug would be possible:
    // processUser(productID)  // Accidentally passing product ID as user ID!

    // With explicit conversion, the intent is clear:
    processUser(UserID(productID))  // Clearly intentional conversion

    fmt.Println("Explicit conversion prevents accidental type mixing")
}

func processUser(id UserID) {
    fmt.Printf("Processing user with ID: %d\n", id)
}
```

**Hands-on Exercise 3: Automation System Design**:

```go
// Comprehensive automation system using proper struct design
package main

import (
    "encoding/json"
    "fmt"
    "time"
)

// Core automation types
type TaskStatus int

const (
    TaskPending TaskStatus = iota
    TaskRunning
    TaskCompleted
    TaskFailed
    TaskCancelled
)

func (ts TaskStatus) String() string {
    switch ts {
    case TaskPending:
        return "Pending"
    case TaskRunning:
        return "Running"
    case TaskCompleted:
        return "Completed"
    case TaskFailed:
        return "Failed"
    case TaskCancelled:
        return "Cancelled"
    default:
        return "Unknown"
    }
}

// Well-designed struct for automation tasks
type AutomationTask struct {
    // Group related fields together for readability
    ID          string    `json:"id"`
    Name        string    `json:"name"`
    Description string    `json:"description,omitempty"`

    // Status and timing information
    Status      TaskStatus `json:"status"`
    CreatedAt   time.Time  `json:"created_at"`
    StartedAt   *time.Time `json:"started_at,omitempty"`
    CompletedAt *time.Time `json:"completed_at,omitempty"`

    // Configuration
    Priority    int           `json:"priority"`
    MaxRetries  int           `json:"max_retries"`
    Timeout     time.Duration `json:"timeout"`

    // Runtime data
    CurrentRetry int                    `json:"current_retry"`
    LastError    string                 `json:"last_error,omitempty"`
    Metadata     map[string]interface{} `json:"metadata,omitempty"`
}

// API response structure
type TaskResponse struct {
    Success bool            `json:"success"`
    Message string          `json:"message"`
    Task    *AutomationTask `json:"task,omitempty"`
    Error   string          `json:"error,omitempty"`
}

// Batch operation structure
type BatchOperation struct {
    OperationID string           `json:"operation_id"`
    Tasks       []AutomationTask `json:"tasks"`
    TotalCount  int              `json:"total_count"`
    Completed   int              `json:"completed"`
    Failed      int              `json:"failed"`
    StartTime   time.Time        `json:"start_time"`
}

func main() {
    demonstrateStructUsage()
    showJSONSerialization()
    analyzeBatchOperations()
}

func demonstrateStructUsage() {
    fmt.Println("=== Automation Task Management ===")

    // Create a new task using literal construction
    task := AutomationTask{
        ID:          "task-001",
        Name:        "Database Backup",
        Description: "Daily backup of production database",
        Status:      TaskPending,
        CreatedAt:   time.Now(),
        Priority:    5,
        MaxRetries:  3,
        Timeout:     30 * time.Minute,
        Metadata: map[string]interface{}{
            "database": "production",
            "type":     "full_backup",
        },
    }

    fmt.Printf("Created task: %s\n", task.Name)
    fmt.Printf("Status: %s\n", task.Status)
    fmt.Printf("Priority: %d\n", task.Priority)
    fmt.Printf("Timeout: %v\n", task.Timeout)

    // Simulate task execution
    startTask(&task)
    completeTask(&task)

    fmt.Printf("Final status: %s\n", task.Status)
    if task.CompletedAt != nil {
        duration := task.CompletedAt.Sub(*task.StartedAt)
        fmt.Printf("Execution time: %v\n", duration)
    }
    fmt.Println()
}

func startTask(task *AutomationTask) {
    now := time.Now()
    task.Status = TaskRunning
    task.StartedAt = &now
}

func completeTask(task *AutomationTask) {
    now := time.Now()
    task.Status = TaskCompleted
    task.CompletedAt = &now
}

func showJSONSerialization() {
    fmt.Println("=== JSON Serialization ===")

    task := AutomationTask{
        ID:          "task-002",
        Name:        "Log Rotation",
        Description: "Rotate application logs",
        Status:      TaskCompleted,
        CreatedAt:   time.Now().Add(-1 * time.Hour),
        Priority:    3,
        MaxRetries:  2,
        Timeout:     10 * time.Minute,
    }

    now := time.Now()
    task.StartedAt = &now
    task.CompletedAt = &now

    response := TaskResponse{
        Success: true,
        Message: "Task completed successfully",
        Task:    &task,
    }

    jsonData, err := json.MarshalIndent(response, "", "  ")
    if err != nil {
        fmt.Printf("JSON marshaling error: %v\n", err)
        return
    }

    fmt.Printf("Task Response JSON:\n%s\n\n", jsonData)
}

func analyzeBatchOperations() {
    fmt.Println("=== Batch Operations ===")

    // Create multiple tasks
    tasks := []AutomationTask{
        {
            ID:         "batch-001",
            Name:       "Cleanup Temp Files",
            Status:     TaskCompleted,
            CreatedAt:  time.Now().Add(-30 * time.Minute),
            Priority:   2,
            MaxRetries: 1,
            Timeout:    5 * time.Minute,
        },
        {
            ID:         "batch-002",
            Name:       "Update Configuration",
            Status:     TaskFailed,
            CreatedAt:  time.Now().Add(-25 * time.Minute),
            Priority:   4,
            MaxRetries: 3,
            Timeout:    15 * time.Minute,
            LastError:  "Configuration file not found",
        },
        {
            ID:         "batch-003",
            Name:       "Send Notifications",
            Status:     TaskRunning,
            CreatedAt:  time.Now().Add(-10 * time.Minute),
            Priority:   1,
            MaxRetries: 2,
            Timeout:    2 * time.Minute,
        },
    }

    batch := BatchOperation{
        OperationID: "batch-op-001",
        Tasks:       tasks,
        TotalCount:  len(tasks),
        StartTime:   time.Now().Add(-30 * time.Minute),
    }

    // Calculate statistics
    for _, task := range batch.Tasks {
        switch task.Status {
        case TaskCompleted:
            batch.Completed++
        case TaskFailed:
            batch.Failed++
        }
    }

    fmt.Printf("Batch Operation: %s\n", batch.OperationID)
    fmt.Printf("Total tasks: %d\n", batch.TotalCount)
    fmt.Printf("Completed: %d\n", batch.Completed)
    fmt.Printf("Failed: %d\n", batch.Failed)
    fmt.Printf("Running: %d\n", batch.TotalCount - batch.Completed - batch.Failed)

    // Show memory usage
    fmt.Printf("\nMemory Analysis:\n")
    fmt.Printf("Single task size: %d bytes\n", unsafe.Sizeof(AutomationTask{}))
    fmt.Printf("Batch operation size: %d bytes\n", unsafe.Sizeof(batch))
    fmt.Printf("Total tasks memory: %d bytes\n",
        unsafe.Sizeof(AutomationTask{}) * uintptr(len(tasks)))
}
```

**Prerequisites**: Module 2
