# Module 15: Methods Part 1: Declaration and Receiver Behavior

**Duration**: 1 hour (45 min theory + 15 min hands-on)

**Learning Objectives**:

- Understand the transition from concrete data to behavior-driven design
- Master method declaration syntax and receiver concepts
- Learn when data should have behavior vs using functions
- Apply method concepts to automation service design
- Understand the importance of semantic consistency

**Videos Covered**:

- 4.1 Topics (0:00:56)
- 4.2 Methods Part 1 (Declare & Receiver Behavior) (0:10:45)

**Key Concepts**:

- Decoupling through behavior: moving from concrete to abstract
- Methods vs functions: when to choose each approach
- Receiver types: value receivers vs pointer receivers
- Semantic consistency: data drives the semantic model
- Method naming conventions and receiver naming
- Go's automatic receiver adjustment during method calls

**Hands-on Exercise 1: Automation Service with Method-Based Behavior**:

```go
// Automation service with method-based behavior
package main

import (
    "fmt"
    "log"
    "time"
)

// ServiceConfig represents automation service configuration
type ServiceConfig struct {
    Name        string
    Port        int
    MaxRetries  int
    Timeout     time.Duration
    EnableDebug bool
}

// Value receiver - operates on copy, used for read-only operations
func (sc ServiceConfig) GetConnectionString() string {
    return fmt.Sprintf("%s:localhost:%d", sc.Name, sc.Port)
}

// Value receiver - validation doesn't need to modify data
func (sc ServiceConfig) IsValid() bool {
    return sc.Name != "" && sc.Port > 0 && sc.Port < 65536
}

// Pointer receiver - modifies the configuration
func (sc *ServiceConfig) UpdateTimeout(timeout time.Duration) {
    sc.Timeout = timeout
    if sc.EnableDebug {
        log.Printf("Updated timeout for %s to %v", sc.Name, timeout)
    }
}

// Pointer receiver - enables/disables debug mode
func (sc *ServiceConfig) SetDebugMode(enabled bool) {
    sc.EnableDebug = enabled
    if enabled {
        log.Printf("Debug mode enabled for service: %s", sc.Name)
    }
}

// AutomationTask represents a task in the automation pipeline
type AutomationTask struct {
    ID          string
    Description string
    Status      string
    CreatedAt   time.Time
    CompletedAt *time.Time
}

// Value receiver - read-only status check
func (at AutomationTask) IsCompleted() bool {
    return at.Status == "completed" && at.CompletedAt != nil
}

// Value receiver - calculate duration
func (at AutomationTask) Duration() time.Duration {
    if at.CompletedAt == nil {
        return time.Since(at.CreatedAt)
    }
    return at.CompletedAt.Sub(at.CreatedAt)
}

// Pointer receiver - mark task as completed
func (at *AutomationTask) MarkCompleted() {
    now := time.Now()
    at.Status = "completed"
    at.CompletedAt = &now
}

// Pointer receiver - update task status
func (at *AutomationTask) UpdateStatus(status string) {
    at.Status = status
    log.Printf("Task %s status updated to: %s", at.ID, status)
}

func main() {
    // Create service configuration using value semantics
    config := ServiceConfig{
        Name:        "DataProcessor",
        Port:        8080,
        MaxRetries:  3,
        Timeout:     30 * time.Second,
        EnableDebug: false,
    }

    // Demonstrate value receiver calls
    fmt.Printf("Service: %s\n", config.GetConnectionString())
    fmt.Printf("Config valid: %t\n", config.IsValid())

    // Demonstrate pointer receiver calls
    // Go automatically takes address of config for pointer receiver methods
    config.SetDebugMode(true)
    config.UpdateTimeout(45 * time.Second)

    // Create automation task
    task := AutomationTask{
        ID:          "TASK-001",
        Description: "Process daily reports",
        Status:      "running",
        CreatedAt:   time.Now().Add(-5 * time.Minute),
    }

    fmt.Printf("\nTask %s - Completed: %t\n", task.ID, task.IsCompleted())
    fmt.Printf("Task duration: %v\n", task.Duration())

    // Complete the task
    task.MarkCompleted()
    fmt.Printf("Task %s - Completed: %t\n", task.ID, task.IsCompleted())
    fmt.Printf("Final duration: %v\n", task.Duration())

    // Demonstrate working with pointer to task
    newTask := &AutomationTask{
        ID:          "TASK-002",
        Description: "Backup database",
        Status:      "pending",
        CreatedAt:   time.Now(),
    }

    // Go automatically dereferences pointer for value receiver methods
    fmt.Printf("\nNew task completed: %t\n", newTask.IsCompleted())
    newTask.UpdateStatus("running")
}
```

**Hands-on Exercise 2: Method vs Function Design Patterns**:

```go
// Comparing method-based vs function-based approaches
package main

import (
    "fmt"
    "strings"
    "time"
)

// LogEntry represents a log entry in automation system
type LogEntry struct {
    Timestamp time.Time
    Level     string
    Message   string
    Service   string
    RequestID string
}

// Method-based approach: LogEntry has behavior
func (le LogEntry) String() string {
    return fmt.Sprintf("[%s] %s %s: %s (req: %s)",
        le.Timestamp.Format("15:04:05"),
        le.Level,
        le.Service,
        le.Message,
        le.RequestID)
}

func (le LogEntry) IsError() bool {
    return strings.ToLower(le.Level) == "error"
}

func (le LogEntry) IsFromService(service string) bool {
    return strings.EqualFold(le.Service, service)
}

func (le *LogEntry) Sanitize() {
    le.Message = strings.ReplaceAll(le.Message, "password=", "password=***")
    le.Message = strings.ReplaceAll(le.Message, "token=", "token=***")
}

// Function-based approach: separate functions operate on data
func FormatLogEntry(entry LogEntry) string {
    return fmt.Sprintf("[%s] %s %s: %s (req: %s)",
        entry.Timestamp.Format("15:04:05"),
        entry.Level,
        entry.Service,
        entry.Message,
        entry.RequestID)
}

func IsErrorLevel(entry LogEntry) bool {
    return strings.ToLower(entry.Level) == "error"
}

func IsFromServiceFunc(entry LogEntry, service string) bool {
    return strings.EqualFold(entry.Service, service)
}

func SanitizeLogEntry(entry *LogEntry) {
    entry.Message = strings.ReplaceAll(entry.Message, "password=", "password=***")
    entry.Message = strings.ReplaceAll(entry.Message, "token=", "token=***")
}

// LogProcessor demonstrates when to use methods vs functions
type LogProcessor struct {
    entries []LogEntry
    filters map[string]bool
}

func NewLogProcessor() *LogProcessor {
    return &LogProcessor{
        entries: make([]LogEntry, 0),
        filters: make(map[string]bool),
    }
}

// Method: operates on LogProcessor state
func (lp *LogProcessor) AddEntry(entry LogEntry) {
    lp.entries = append(lp.entries, entry)
}

// Method: uses LogProcessor state
func (lp *LogProcessor) GetErrorCount() int {
    count := 0
    for _, entry := range lp.entries {
        if entry.IsError() { // Using method on LogEntry
            count++
        }
    }
    return count
}

// Method: modifies LogProcessor state
func (lp *LogProcessor) SetServiceFilter(service string, enabled bool) {
    lp.filters[service] = enabled
}

// Method: uses LogProcessor state to filter
func (lp *LogProcessor) GetFilteredEntries() []LogEntry {
    var filtered []LogEntry

    for _, entry := range lp.entries {
        if shouldInclude, exists := lp.filters[entry.Service]; exists {
            if shouldInclude {
                filtered = append(filtered, entry)
            }
        } else {
            // Include by default if no filter set
            filtered = append(filtered, entry)
        }
    }

    return filtered
}

func main() {
    demonstrateMethodVsFunction()
    demonstrateLogProcessor()
}

func demonstrateMethodVsFunction() {
    fmt.Println("=== Method vs Function Approaches ===")

    entry := LogEntry{
        Timestamp: time.Now(),
        Level:     "ERROR",
        Message:   "Database connection failed with password=secret123",
        Service:   "UserService",
        RequestID: "req-12345",
    }

    fmt.Println("Original entry:")
    fmt.Printf("  Method approach: %s\n", entry.String())
    fmt.Printf("  Function approach: %s\n", FormatLogEntry(entry))

    fmt.Printf("  Is error (method): %t\n", entry.IsError())
    fmt.Printf("  Is error (function): %t\n", IsErrorLevel(entry))

    fmt.Printf("  From UserService (method): %t\n", entry.IsFromService("UserService"))
    fmt.Printf("  From UserService (function): %t\n", IsFromServiceFunc(entry, "UserService"))

    // Demonstrate mutation
    fmt.Println("\nAfter sanitization:")

    // Method approach
    entryCopy1 := entry
    entryCopy1.Sanitize()
    fmt.Printf("  Method approach: %s\n", entryCopy1.String())

    // Function approach
    entryCopy2 := entry
    SanitizeLogEntry(&entryCopy2)
    fmt.Printf("  Function approach: %s\n", FormatLogEntry(entryCopy2))

    fmt.Println("\nWhen to use methods vs functions:")
    fmt.Println("  Use METHODS when:")
    fmt.Println("    - Data should have behavior")
    fmt.Println("    - Operations are core to the type's purpose")
    fmt.Println("    - You want to enable method chaining")
    fmt.Println("    - The operation is type-specific")

    fmt.Println("  Use FUNCTIONS when:")
    fmt.Println("    - Operating on multiple types")
    fmt.Println("    - Utility operations not core to type")
    fmt.Println("    - Stateless transformations")
    fmt.Println("    - Package-level operations")
}

func demonstrateLogProcessor() {
    fmt.Println("\n=== Log Processor Demo ===")

    processor := NewLogProcessor()

    // Add sample log entries
    entries := []LogEntry{
        {
            Timestamp: time.Now().Add(-5 * time.Minute),
            Level:     "INFO",
            Message:   "Service started successfully",
            Service:   "UserService",
            RequestID: "req-001",
        },
        {
            Timestamp: time.Now().Add(-4 * time.Minute),
            Level:     "ERROR",
            Message:   "Database connection timeout",
            Service:   "UserService",
            RequestID: "req-002",
        },
        {
            Timestamp: time.Now().Add(-3 * time.Minute),
            Level:     "INFO",
            Message:   "Processing payment request",
            Service:   "PaymentService",
            RequestID: "req-003",
        },
        {
            Timestamp: time.Now().Add(-2 * time.Minute),
            Level:     "ERROR",
            Message:   "Payment validation failed",
            Service:   "PaymentService",
            RequestID: "req-004",
        },
        {
            Timestamp: time.Now().Add(-1 * time.Minute),
            Level:     "INFO",
            Message:   "User logged in successfully",
            Service:   "AuthService",
            RequestID: "req-005",
        },
    }

    for _, entry := range entries {
        processor.AddEntry(entry)
    }

    fmt.Printf("Total entries: %d\n", len(processor.entries))
    fmt.Printf("Error entries: %d\n", processor.GetErrorCount())

    // Set up filters
    processor.SetServiceFilter("UserService", true)
    processor.SetServiceFilter("PaymentService", true)
    processor.SetServiceFilter("AuthService", false)

    filtered := processor.GetFilteredEntries()
    fmt.Printf("Filtered entries: %d\n", len(filtered))

    fmt.Println("\nFiltered log entries:")
    for _, entry := range filtered {
        fmt.Printf("  %s\n", entry.String())
    }
}
```

**Hands-on Exercise 3: Receiver Type Decision Patterns**:

```go
// Patterns for choosing value vs pointer receivers
package main

import (
    "fmt"
    "time"
)

// Example 1: Small, immutable data - use value receivers
type Point struct {
    X, Y float64
}

// Value receiver - Point is small and operations don't modify
func (p Point) Distance() float64 {
    return p.X*p.X + p.Y*p.Y
}

// Value receiver - returns new Point (immutable pattern)
func (p Point) Add(other Point) Point {
    return Point{X: p.X + other.X, Y: p.Y + other.Y}
}

// Value receiver - comparison operation
func (p Point) Equals(other Point) bool {
    return p.X == other.X && p.Y == other.Y
}

// Example 2: Large struct - use pointer receivers to avoid copying
type AutomationReport struct {
    ID          string
    Title       string
    GeneratedAt time.Time
    Data        [1000]float64 // Large array
    Metadata    map[string]interface{}
    Summary     string
}

// Pointer receiver - avoid copying large struct
func (ar *AutomationReport) AddDataPoint(index int, value float64) {
    if index >= 0 && index < len(ar.Data) {
        ar.Data[index] = value
    }
}

// Pointer receiver - modifies the report
func (ar *AutomationReport) UpdateSummary(summary string) {
    ar.Summary = summary
    ar.Metadata["last_updated"] = time.Now()
}

// Pointer receiver - even for read-only to maintain consistency
func (ar *AutomationReport) GetDataSum() float64 {
    var sum float64
    for _, value := range ar.Data {
        sum += value
    }
    return sum
}

// Example 3: Mixed semantics - when you need both
type Counter struct {
    value int
    name  string
}

// Value receiver - read-only operations that don't need to modify
func (c Counter) String() string {
    return fmt.Sprintf("%s: %d", c.name, c.value)
}

// Value receiver - comparison
func (c Counter) Equals(other Counter) bool {
    return c.value == other.value && c.name == other.name
}

// Pointer receiver - modifies the counter
func (c *Counter) Increment() {
    c.value++
}

// Pointer receiver - modifies the counter
func (c *Counter) Add(amount int) {
    c.value += amount
}

// Pointer receiver - resets the counter
func (c *Counter) Reset() {
    c.value = 0
}

// Example 4: Interface satisfaction considerations
type Processor interface {
    Process() error
    GetStatus() string
}

// DataProcessor with pointer receiver methods
type DataProcessor struct {
    status string
    data   []byte
}

// Pointer receiver - modifies status
func (dp *DataProcessor) Process() error {
    dp.status = "processing"
    // Simulate processing
    time.Sleep(10 * time.Millisecond)
    dp.status = "completed"
    return nil
}

// Pointer receiver - for consistency (even though read-only)
func (dp *DataProcessor) GetStatus() string {
    return dp.status
}

func main() {
    demonstrateValueReceivers()
    demonstratePointerReceivers()
    demonstrateMixedSemantics()
    demonstrateInterfaceConsiderations()
}

func demonstrateValueReceivers() {
    fmt.Println("=== Value Receivers (Small, Immutable Data) ===")

    p1 := Point{X: 3, Y: 4}
    p2 := Point{X: 1, Y: 2}

    fmt.Printf("Point 1: (%.1f, %.1f)\n", p1.X, p1.Y)
    fmt.Printf("Point 2: (%.1f, %.1f)\n", p2.X, p2.Y)
    fmt.Printf("Distance of p1: %.2f\n", p1.Distance())

    // Immutable operations
    p3 := p1.Add(p2)
    fmt.Printf("p1 + p2 = (%.1f, %.1f)\n", p3.X, p3.Y)
    fmt.Printf("p1 unchanged: (%.1f, %.1f)\n", p1.X, p1.Y)
    fmt.Printf("p1 equals p3: %t\n", p1.Equals(p3))

    // Value receivers work with both values and pointers
    pPtr := &Point{X: 5, Y: 6}
    fmt.Printf("Pointer to point distance: %.2f\n", pPtr.Distance())
}

func demonstratePointerReceivers() {
    fmt.Println("\n=== Pointer Receivers (Large Data, Mutations) ===")

    report := &AutomationReport{
        ID:          "RPT-001",
        Title:       "Daily Automation Report",
        GeneratedAt: time.Now(),
        Metadata:    make(map[string]interface{}),
    }

    // Add some data points
    for i := 0; i < 10; i++ {
        report.AddDataPoint(i, float64(i*10))
    }

    fmt.Printf("Report ID: %s\n", report.ID)
    fmt.Printf("Data sum (first 10 points): %.1f\n", report.GetDataSum())

    report.UpdateSummary("Automation tasks completed successfully")
    fmt.Printf("Summary: %s\n", report.Summary)

    if lastUpdated, exists := report.Metadata["last_updated"]; exists {
        fmt.Printf("Last updated: %v\n", lastUpdated)
    }
}

func demonstrateMixedSemantics() {
    fmt.Println("\n=== Mixed Semantics (Value + Pointer) ===")

    counter := Counter{value: 0, name: "TaskCounter"}

    // Value receiver methods work with value
    fmt.Printf("Initial: %s\n", counter.String())

    // Pointer receiver methods need address
    counter.Increment()
    counter.Add(5)
    fmt.Printf("After increment and add: %s\n", counter.String())

    // Create another counter for comparison
    counter2 := Counter{value: 6, name: "TaskCounter"}
    fmt.Printf("Counters equal: %t\n", counter.Equals(counter2))

    counter.Reset()
    fmt.Printf("After reset: %s\n", counter.String())

    // Working with pointer to counter
    counterPtr := &Counter{value: 100, name: "PointerCounter"}
    counterPtr.Increment() // Pointer receiver
    fmt.Printf("Pointer counter: %s\n", counterPtr.String()) // Value receiver
}

func demonstrateInterfaceConsiderations() {
    fmt.Println("\n=== Interface Considerations ===")

    // When using pointer receiver methods, you need pointer to satisfy interface
    var processor Processor

    // This works - pointer satisfies interface with pointer receiver methods
    dp := &DataProcessor{status: "idle"}
    processor = dp

    fmt.Printf("Initial status: %s\n", processor.GetStatus())

    err := processor.Process()
    if err != nil {
        fmt.Printf("Error: %v\n", err)
    } else {
        fmt.Printf("Final status: %s\n", processor.GetStatus())
    }

    // Note: If DataProcessor had value receiver methods, both
    // DataProcessor{} and &DataProcessor{} could satisfy the interface

    fmt.Println("\nReceiver type guidelines:")
    fmt.Println("  Use VALUE receivers when:")
    fmt.Println("    - Type is small (few fields, no large arrays/slices)")
    fmt.Println("    - Type is immutable by design")
    fmt.Println("    - Type is a basic type (int, string, bool)")
    fmt.Println("    - Operations don't modify the receiver")

    fmt.Println("  Use POINTER receivers when:")
    fmt.Println("    - Type is large (would be expensive to copy)")
    fmt.Println("    - Method needs to modify the receiver")
    fmt.Println("    - Type contains slices, maps, or channels")
    fmt.Println("    - Consistency: if any method uses pointer, use pointer for all")
}
```

**Prerequisites**: Module 14
