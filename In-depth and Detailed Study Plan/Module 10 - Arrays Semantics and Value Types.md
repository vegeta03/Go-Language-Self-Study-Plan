# Module 10: Arrays: Semantics and Value Types

**Duration**: 1 hour (45 min theory + 15 min hands-on)

**Learning Objectives**:

- Master array value semantics and copying behavior
- Understand when arrays are appropriate vs slices
- Learn array comparison and assignment rules
- Apply array semantics in automation contexts

**Videos Covered**:

- 3.4 Arrays Part 2 (Semantics) (0:16:43)

**Key Concepts**:

- Arrays are value types (copied on assignment)
- Array comparison and equality
- Passing arrays to functions (copy vs pointer)
- Array literals and initialization patterns
- When to choose arrays over slices

**Hands-on Exercise 1: Configuration Validation with Array Semantics**:

```go
// Configuration validation using array semantics
package main

import "fmt"

const MaxConfigItems = 10

type ConfigValidator struct {
    requiredFields [MaxConfigItems]string
    fieldCount     int
}

func NewConfigValidator(fields []string) ConfigValidator {
    var validator ConfigValidator

    // Copy fields into fixed array
    for i, field := range fields {
        if i >= MaxConfigItems {
            break
        }
        validator.requiredFields[i] = field
        validator.fieldCount++
    }

    return validator // Array is copied by value
}

func (cv ConfigValidator) Validate(config map[string]string) []string {
    var missing []string

    // Array iteration - working with copy
    for i := 0; i < cv.fieldCount; i++ {
        field := cv.requiredFields[i]
        if _, exists := config[field]; !exists {
            missing = append(missing, field)
        }
    }

    return missing
}

// Demonstrate array comparison
func (cv ConfigValidator) Equals(other ConfigValidator) bool {
    return cv.requiredFields == other.requiredFields &&
           cv.fieldCount == other.fieldCount
}

func main() {
    // Create validators with different required fields
    webValidator := NewConfigValidator([]string{
        "server_port", "database_url", "api_key", "log_level",
    })

    dbValidator := NewConfigValidator([]string{
        "host", "port", "username", "password", "database",
    })

    // Test configurations
    webConfig := map[string]string{
        "server_port":  "8080",
        "database_url": "localhost:5432",
        "api_key":      "secret123",
        // missing log_level
    }

    dbConfig := map[string]string{
        "host":     "localhost",
        "port":     "5432",
        "username": "admin",
        "password": "secret",
        "database": "myapp",
    }

    // Validate configurations
    if missing := webValidator.Validate(webConfig); len(missing) > 0 {
        fmt.Printf("Web config missing: %v\n", missing)
    } else {
        fmt.Println("Web config is valid")
    }

    if missing := dbValidator.Validate(dbConfig); len(missing) > 0 {
        fmt.Printf("DB config missing: %v\n", missing)
    } else {
        fmt.Println("DB config is valid")
    }

    // Demonstrate array comparison
    webValidator2 := NewConfigValidator([]string{
        "server_port", "database_url", "api_key", "log_level",
    })

    fmt.Printf("Validators equal: %t\n", webValidator.Equals(webValidator2))
}
```

**Hands-on Exercise 2: Array Value Semantics vs Pointer Semantics**:

```go
// Demonstrate array copying behavior and when to use pointers
package main

import (
    "fmt"
    "unsafe"
)

const (
    BufferSize = 1000
    DataPoints = 100
)

// Large array for demonstration
type DataBuffer [BufferSize]float64

// Processor that works with array values (copies)
type ValueProcessor struct {
    buffer DataBuffer
    count  int
}

// Processor that works with array pointers (shared)
type PointerProcessor struct {
    buffer *DataBuffer
    count  int
}

func NewValueProcessor() ValueProcessor {
    return ValueProcessor{}
}

func NewPointerProcessor() PointerProcessor {
    return PointerProcessor{
        buffer: &DataBuffer{},
    }
}

// Value receiver - operates on copy
func (vp ValueProcessor) ProcessData(data DataBuffer) DataBuffer {
    fmt.Printf("Value processor - buffer address: %p\n", &vp.buffer)

    // Modify the copy
    for i := 0; i < BufferSize && i < len(data); i++ {
        vp.buffer[i] = data[i] * 2.0
    }

    return vp.buffer // Returns a copy
}

// Pointer receiver - operates on shared data
func (pp *PointerProcessor) ProcessData(data *DataBuffer) {
    fmt.Printf("Pointer processor - buffer address: %p\n", pp.buffer)

    // Modify the original
    for i := 0; i < BufferSize; i++ {
        pp.buffer[i] = data[i] * 2.0
    }
}

func main() {
    fmt.Println("=== Array Value vs Pointer Semantics ===")

    // Create test data
    var originalData DataBuffer
    for i := 0; i < DataPoints; i++ {
        originalData[i] = float64(i + 1)
    }

    fmt.Printf("Original data address: %p\n", &originalData)
    fmt.Printf("Original data size: %d bytes\n", unsafe.Sizeof(originalData))

    // Test value semantics
    fmt.Println("\n--- Value Semantics Test ---")
    valueProc := NewValueProcessor()

    // This creates a copy of the entire array
    processedData := valueProc.ProcessData(originalData)
    fmt.Printf("Processed data address: %p\n", &processedData)

    // Original data unchanged
    fmt.Printf("Original[0]: %.1f, Processed[0]: %.1f\n",
        originalData[0], processedData[0])

    // Test pointer semantics
    fmt.Println("\n--- Pointer Semantics Test ---")
    pointerProc := NewPointerProcessor()

    // Create a copy for pointer processing
    var dataForPointer DataBuffer = originalData
    fmt.Printf("Data for pointer address: %p\n", &dataForPointer)

    // This works with the original data (no copy)
    pointerProc.ProcessData(&dataForPointer)

    // Original data is modified
    fmt.Printf("After pointer processing[0]: %.1f\n", dataForPointer[0])

    // Demonstrate array assignment (copy)
    fmt.Println("\n--- Array Assignment Test ---")
    var array1 DataBuffer
    var array2 DataBuffer

    // Initialize array1
    for i := 0; i < 10; i++ {
        array1[i] = float64(i * 10)
    }

    fmt.Printf("Array1 address: %p\n", &array1)
    fmt.Printf("Array2 address: %p\n", &array2)

    // Assignment creates a complete copy
    array2 = array1

    // Modify array1
    array1[0] = 999.0

    fmt.Printf("After modification - Array1[0]: %.1f, Array2[0]: %.1f\n",
        array1[0], array2[0])

    // Array comparison
    fmt.Println("\n--- Array Comparison Test ---")
    var array3, array4 [5]int

    // Initialize with same values
    for i := 0; i < 5; i++ {
        array3[i] = i
        array4[i] = i
    }

    fmt.Printf("Arrays equal: %t\n", array3 == array4)

    // Change one value
    array4[2] = 99
    fmt.Printf("After change, arrays equal: %t\n", array3 == array4)
}
```

**Hands-on Exercise 3: When to Choose Arrays vs Slices**:

```go
// Practical examples of when to use arrays vs slices
package main

import (
    "fmt"
    "time"
    "unsafe"
)

// Use arrays for fixed-size, known-at-compile-time data
const (
    DaysInWeek = 7
    MonthsInYear = 12
    MaxRetries = 3
)

// Calendar data - perfect for arrays (fixed size)
type WeeklySchedule struct {
    tasks [DaysInWeek]string
    hours [DaysInWeek]int
}

type YearlyBudget struct {
    monthlyBudgets [MonthsInYear]float64
    monthNames     [MonthsInYear]string
}

// Retry configuration - fixed maximum attempts
type RetryConfig struct {
    delays [MaxRetries]time.Duration
    count  int
}

// Use slices for dynamic, variable-size data
type TaskList struct {
    tasks []string
    priorities []int
}

type LogBuffer struct {
    entries []string
    maxSize int
}

func main() {
    demonstrateArrayUseCases()
    demonstrateSliceUseCases()
    comparePerformance()
}

func demonstrateArrayUseCases() {
    fmt.Println("=== Array Use Cases ===")

    // Weekly schedule - fixed 7 days
    schedule := WeeklySchedule{
        tasks: [DaysInWeek]string{
            "Planning", "Development", "Testing",
            "Review", "Deployment", "Monitoring", "Rest",
        },
        hours: [DaysInWeek]int{8, 8, 8, 8, 8, 4, 0},
    }

    fmt.Println("Weekly Schedule:")
    dayNames := [DaysInWeek]string{
        "Monday", "Tuesday", "Wednesday", "Thursday",
        "Friday", "Saturday", "Sunday",
    }

    for i := 0; i < DaysInWeek; i++ {
        fmt.Printf("  %s: %s (%d hours)\n",
            dayNames[i], schedule.tasks[i], schedule.hours[i])
    }

    // Yearly budget - fixed 12 months
    budget := YearlyBudget{
        monthlyBudgets: [MonthsInYear]float64{
            10000, 12000, 11000, 13000, 14000, 15000,
            16000, 15000, 14000, 13000, 12000, 18000,
        },
        monthNames: [MonthsInYear]string{
            "Jan", "Feb", "Mar", "Apr", "May", "Jun",
            "Jul", "Aug", "Sep", "Oct", "Nov", "Dec",
        },
    }

    total := 0.0
    for i := 0; i < MonthsInYear; i++ {
        total += budget.monthlyBudgets[i]
    }
    fmt.Printf("\nTotal yearly budget: $%.2f\n", total)

    // Retry configuration - fixed maximum attempts
    retryConfig := RetryConfig{
        delays: [MaxRetries]time.Duration{
            1 * time.Second,
            5 * time.Second,
            10 * time.Second,
        },
        count: MaxRetries,
    }

    fmt.Println("\nRetry configuration:")
    for i := 0; i < retryConfig.count; i++ {
        fmt.Printf("  Attempt %d: wait %v\n", i+1, retryConfig.delays[i])
    }
}

func demonstrateSliceUseCases() {
    fmt.Println("\n=== Slice Use Cases ===")

    // Dynamic task list
    taskList := TaskList{
        tasks:      make([]string, 0, 10),
        priorities: make([]int, 0, 10),
    }

    // Add tasks dynamically
    tasks := []struct {
        name     string
        priority int
    }{
        {"Setup environment", 5},
        {"Write tests", 7},
        {"Implement feature", 8},
        {"Code review", 6},
        {"Deploy to staging", 9},
    }

    for _, task := range tasks {
        taskList.tasks = append(taskList.tasks, task.name)
        taskList.priorities = append(taskList.priorities, task.priority)
    }

    fmt.Printf("Dynamic task list (%d tasks):\n", len(taskList.tasks))
    for i, task := range taskList.tasks {
        fmt.Printf("  %s (priority: %d)\n", task, taskList.priorities[i])
    }

    // Log buffer with dynamic growth
    logBuffer := LogBuffer{
        entries: make([]string, 0),
        maxSize: 5,
    }

    // Add log entries
    logEntries := []string{
        "Application started",
        "Database connected",
        "User logged in",
        "Processing request",
        "Request completed",
        "User logged out",
        "Application stopping",
    }

    for _, entry := range logEntries {
        logBuffer.entries = append(logBuffer.entries, entry)

        // Keep only recent entries
        if len(logBuffer.entries) > logBuffer.maxSize {
            logBuffer.entries = logBuffer.entries[1:]
        }
    }

    fmt.Printf("\nRecent log entries (max %d):\n", logBuffer.maxSize)
    for i, entry := range logBuffer.entries {
        fmt.Printf("  %d: %s\n", i+1, entry)
    }
}

func comparePerformance() {
    fmt.Println("\n=== Performance Comparison ===")

    // Array - stack allocated, predictable
    var arrayData [1000]int
    fmt.Printf("Array size: %d bytes (stack allocated)\n",
        unsafe.Sizeof(arrayData))

    // Slice - heap allocated header + backing array
    sliceData := make([]int, 1000)
    fmt.Printf("Slice header size: %d bytes (heap backing array)\n",
        unsafe.Sizeof(sliceData))

    // Memory usage
    fmt.Println("\nMemory characteristics:")
    fmt.Println("Arrays:")
    fmt.Println("  + Predictable memory usage")
    fmt.Println("  + Stack allocated (faster)")
    fmt.Println("  + No heap allocations")
    fmt.Println("  - Fixed size at compile time")
    fmt.Println("  - Large arrays can cause stack overflow")

    fmt.Println("\nSlices:")
    fmt.Println("  + Dynamic sizing")
    fmt.Println("  + Efficient append operations")
    fmt.Println("  + Can grow as needed")
    fmt.Println("  - Heap allocations (GC pressure)")
    fmt.Println("  - Indirection overhead")

    fmt.Println("\nChoose arrays when:")
    fmt.Println("  - Size is known at compile time")
    fmt.Println("  - Size is small to moderate")
    fmt.Println("  - Predictable memory usage is important")
    fmt.Println("  - Working with fixed collections (days, months, etc.)")

    fmt.Println("\nChoose slices when:")
    fmt.Println("  - Size varies at runtime")
    fmt.Println("  - Need to append/remove elements")
    fmt.Println("  - Working with user input or external data")
    fmt.Println("  - Size could be very large")
}
```

**Prerequisites**: Module 9
