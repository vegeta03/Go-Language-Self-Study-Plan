# Module 16: Methods Part 2: Value vs Pointer Semantics

**Duration**: 1 hour (45 min theory + 15 min hands-on)

**Learning Objectives**:

- Master semantic consistency principles for different type classes
- Understand when to use value vs pointer semantics for methods
- Learn standard library patterns and semantic guidelines
- Apply semantic consistency to automation system design
- Recognize exceptions and when they're appropriate

**Videos Covered**:

- 4.3 Methods Part 2 (Value & Pointer Semantics) (0:15:35)

**Key Concepts**:

- Three classes of types: built-in, reference, and user-defined
- Built-in types (numerics, strings, bools): always use value semantics
- Reference types (slices, maps, channels, interfaces): use value semantics
- User-defined struct types: choose based on data characteristics
- Type drives semantics, not the operation
- Mutation APIs can use value semantics (sandbox pattern)
- Standard library semantic consistency patterns

**Hands-on Exercise 1: Semantic Consistency in Automation Systems**:

```go
// Semantic consistency in automation monitoring system
package main

import (
    "fmt"
    "strings"
    "time"
)

// Built-in type wrapper - uses value semantics
type ServiceID string

// Value receiver - built-in types always use value semantics
func (sid ServiceID) IsValid() bool {
    return len(sid) > 0 && !strings.Contains(string(sid), " ")
}

// Value receiver - returns new value (immutable pattern)
func (sid ServiceID) WithPrefix(prefix string) ServiceID {
    return ServiceID(prefix + string(sid))
}

// Reference type wrapper - uses value semantics
type MetricValues []float64

// Value receiver - reference types use value semantics
func (mv MetricValues) Average() float64 {
    if len(mv) == 0 {
        return 0
    }

    var sum float64
    for _, value := range mv {
        sum += value
    }
    return sum / float64(len(mv))
}

// Value receiver - mutation API using value semantics (sandbox pattern)
func (mv MetricValues) Normalize() MetricValues {
    if len(mv) == 0 {
        return mv
    }

    // Create new slice (mutation in isolation)
    normalized := make(MetricValues, len(mv))
    max := mv.Max()

    for i, value := range mv {
        normalized[i] = value / max
    }

    return normalized // Return new value
}

// Value receiver - helper method
func (mv MetricValues) Max() float64 {
    if len(mv) == 0 {
        return 0
    }

    max := mv[0]
    for _, value := range mv {
        if value > max {
            max = value
        }
    }
    return max
}

// User-defined struct type - choose semantics based on data nature
type MonitoringSession struct {
    ID        ServiceID
    StartTime time.Time
    Metrics   MetricValues
    Active    bool
}

// Value receiver - read-only operations for small, simple structs
func (ms MonitoringSession) Duration() time.Duration {
    return time.Since(ms.StartTime)
}

// Value receiver - validation doesn't require mutation
func (ms MonitoringSession) IsHealthy() bool {
    if !ms.Active || len(ms.Metrics) == 0 {
        return false
    }

    avg := ms.Metrics.Average()
    return avg > 0.5 && avg < 0.95 // Healthy range
}

// Large struct type - uses pointer semantics for efficiency
type AutomationPipeline struct {
    ID          ServiceID
    Name        string
    Sessions    []MonitoringSession
    Config      map[string]string
    Statistics  struct {
        TotalRuns    int64
        SuccessCount int64
        FailureCount int64
        LastRun      time.Time
    }
    CreatedAt time.Time
    UpdatedAt time.Time
}

// Pointer receiver - large structs use pointer semantics
func (ap *AutomationPipeline) AddSession(session MonitoringSession) {
    ap.Sessions = append(ap.Sessions, session)
    ap.UpdatedAt = time.Now()
}

// Pointer receiver - mutation requires pointer semantics
func (ap *AutomationPipeline) RecordRun(success bool) {
    ap.Statistics.TotalRuns++
    if success {
        ap.Statistics.SuccessCount++
    } else {
        ap.Statistics.FailureCount++
    }
    ap.Statistics.LastRun = time.Now()
    ap.UpdatedAt = time.Now()
}

// Pointer receiver - consistent with struct semantics
func (ap *AutomationPipeline) SuccessRate() float64 {
    if ap.Statistics.TotalRuns == 0 {
        return 0
    }
    return float64(ap.Statistics.SuccessCount) / float64(ap.Statistics.TotalRuns)
}

func main() {
    // Built-in type semantics
    serviceID := ServiceID("web-service")
    fmt.Printf("Service ID valid: %t\n", serviceID.IsValid())
    prefixedID := serviceID.WithPrefix("prod-")
    fmt.Printf("Prefixed ID: %s\n", prefixedID)

    // Reference type semantics
    metrics := MetricValues{0.1, 0.8, 0.6, 0.9, 0.7}
    fmt.Printf("Average: %.2f\n", metrics.Average())

    // Mutation in isolation (value semantics)
    normalized := metrics.Normalize()
    fmt.Printf("Original max: %.2f\n", metrics.Max())
    fmt.Printf("Normalized max: %.2f\n", normalized.Max())

    // Small struct with value semantics
    session := MonitoringSession{
        ID:        serviceID,
        StartTime: time.Now().Add(-5 * time.Minute),
        Metrics:   metrics,
        Active:    true,
    }

    fmt.Printf("Session duration: %v\n", session.Duration())
    fmt.Printf("Session healthy: %t\n", session.IsHealthy())

    // Large struct with pointer semantics
    pipeline := &AutomationPipeline{
        ID:        ServiceID("pipeline-001"),
        Name:      "Daily Processing Pipeline",
        Config:    make(map[string]string),
        CreatedAt: time.Now(),
        UpdatedAt: time.Now(),
    }

    // Add sessions and record runs
    pipeline.AddSession(session)
    pipeline.RecordRun(true)
    pipeline.RecordRun(true)
    pipeline.RecordRun(false)

    fmt.Printf("Pipeline success rate: %.2f%%\n", pipeline.SuccessRate()*100)
    fmt.Printf("Total sessions: %d\n", len(pipeline.Sessions))
}
```

**Hands-on Exercise 2: Type Classification and Semantic Guidelines**:

```go
// Demonstrating semantic guidelines for different type classes
package main

import (
    "fmt"
    "time"
)

// ========== Built-in Type Wrappers (Always Value Semantics) ==========

type UserID int64
type Email string
type Temperature float64

// Built-in types: always use value receivers
func (uid UserID) IsValid() bool {
    return uid > 0
}

func (uid UserID) String() string {
    return fmt.Sprintf("user-%d", uid)
}

func (e Email) IsValid() bool {
    return len(e) > 0 && string(e)[0] != '@'
}

func (e Email) Domain() string {
    for i := len(e) - 1; i >= 0; i-- {
        if e[i] == '@' {
            return string(e[i+1:])
        }
    }
    return ""
}

func (t Temperature) Celsius() float64 {
    return float64(t)
}

func (t Temperature) Fahrenheit() float64 {
    return float64(t)*9/5 + 32
}

// ========== Reference Type Wrappers (Value Semantics) ==========

type TaskQueue []string
type ConfigMap map[string]string
type NotificationChannel chan string

// Reference types: use value receivers (even for mutations)
func (tq TaskQueue) Length() int {
    return len(tq)
}

func (tq TaskQueue) Contains(task string) bool {
    for _, t := range tq {
        if t == task {
            return true
        }
    }
    return false
}

// Mutation using value semantics (sandbox pattern)
func (tq TaskQueue) Add(task string) TaskQueue {
    return append(tq, task)
}

func (tq TaskQueue) Filter(predicate func(string) bool) TaskQueue {
    var filtered TaskQueue
    for _, task := range tq {
        if predicate(task) {
            filtered = append(filtered, task)
        }
    }
    return filtered
}

func (cm ConfigMap) Get(key string) (string, bool) {
    value, exists := cm[key]
    return value, exists
}

func (cm ConfigMap) Set(key, value string) ConfigMap {
    // Create new map (sandbox pattern)
    newMap := make(ConfigMap)
    for k, v := range cm {
        newMap[k] = v
    }
    newMap[key] = value
    return newMap
}

func (nc NotificationChannel) Send(message string) {
    select {
    case nc <- message:
    default:
        // Channel full, message dropped
    }
}

// ========== User-Defined Types (Choose Based on Data) ==========

// Small, simple struct - value semantics
type Coordinate struct {
    X, Y float64
}

func (c Coordinate) Distance() float64 {
    return c.X*c.X + c.Y*c.Y
}

func (c Coordinate) Add(other Coordinate) Coordinate {
    return Coordinate{X: c.X + other.X, Y: c.Y + other.Y}
}

// Large or complex struct - pointer semantics
type AutomationJob struct {
    ID          UserID
    Name        string
    Owner       Email
    Tasks       TaskQueue
    Config      ConfigMap
    Status      string
    CreatedAt   time.Time
    UpdatedAt   time.Time
    Logs        []string
    Metrics     map[string]float64
    Temperature Temperature
}

// Pointer semantics for large structs
func (aj *AutomationJob) AddTask(task string) {
    aj.Tasks = aj.Tasks.Add(task)
    aj.UpdatedAt = time.Now()
}

func (aj *AutomationJob) UpdateStatus(status string) {
    aj.Status = status
    aj.UpdatedAt = time.Now()
    aj.Logs = append(aj.Logs, fmt.Sprintf("Status changed to: %s", status))
}

func (aj *AutomationJob) SetConfig(key, value string) {
    aj.Config = aj.Config.Set(key, value)
    aj.UpdatedAt = time.Now()
}

func (aj *AutomationJob) GetSummary() string {
    return fmt.Sprintf("Job %s (%s) - %d tasks, status: %s",
        aj.Name, aj.Owner, aj.Tasks.Length(), aj.Status)
}

func main() {
    demonstrateBuiltinTypes()
    demonstrateReferenceTypes()
    demonstrateUserDefinedTypes()
    demonstrateSemanticConsistency()
}

func demonstrateBuiltinTypes() {
    fmt.Println("=== Built-in Type Wrappers (Value Semantics) ===")

    userID := UserID(12345)
    email := Email("user@example.com")
    temp := Temperature(25.5)

    fmt.Printf("User ID: %s, valid: %t\n", userID.String(), userID.IsValid())
    fmt.Printf("Email: %s, domain: %s, valid: %t\n",
        email, email.Domain(), email.IsValid())
    fmt.Printf("Temperature: %.1fÂ°C / %.1fÂ°F\n",
        temp.Celsius(), temp.Fahrenheit())

    // Built-in types always use value semantics
    fmt.Println("Built-in types always use value receivers")
}

func demonstrateReferenceTypes() {
    fmt.Println("\n=== Reference Type Wrappers (Value Semantics) ===")

    tasks := TaskQueue{"task1", "task2", "task3"}
    config := ConfigMap{"env": "prod", "debug": "false"}

    fmt.Printf("Tasks: %d items\n", tasks.Length())
    fmt.Printf("Contains 'task2': %t\n", tasks.Contains("task2"))

    // Mutation using value semantics (returns new value)
    newTasks := tasks.Add("task4")
    fmt.Printf("Original tasks: %d\n", tasks.Length())
    fmt.Printf("New tasks: %d\n", newTasks.Length())

    // Filter using value semantics
    filtered := newTasks.Filter(func(task string) bool {
        return task != "task2"
    })
    fmt.Printf("Filtered tasks: %d\n", filtered.Length())

    // Config operations
    if value, exists := config.Get("env"); exists {
        fmt.Printf("Environment: %s\n", value)
    }

    newConfig := config.Set("timeout", "30s")
    fmt.Printf("Original config size: %d\n", len(config))
    fmt.Printf("New config size: %d\n", len(newConfig))
}

func demonstrateUserDefinedTypes() {
    fmt.Println("\n=== User-Defined Types (Choose Semantics) ===")

    // Small struct - value semantics
    coord1 := Coordinate{X: 3, Y: 4}
    coord2 := Coordinate{X: 1, Y: 2}

    fmt.Printf("Coordinate 1: (%.1f, %.1f), distance: %.2f\n",
        coord1.X, coord1.Y, coord1.Distance())

    sum := coord1.Add(coord2)
    fmt.Printf("Sum: (%.1f, %.1f)\n", sum.X, sum.Y)
    fmt.Printf("Original unchanged: (%.1f, %.1f)\n", coord1.X, coord1.Y)

    // Large struct - pointer semantics
    job := &AutomationJob{
        ID:        UserID(1001),
        Name:      "Data Processing Job",
        Owner:     Email("admin@company.com"),
        Tasks:     TaskQueue{"validate", "process", "notify"},
        Config:    ConfigMap{"batch_size": "100"},
        Status:    "pending",
        CreatedAt: time.Now(),
        UpdatedAt: time.Now(),
        Logs:      make([]string, 0),
        Metrics:   make(map[string]float64),
    }

    fmt.Printf("Job summary: %s\n", job.GetSummary())

    job.AddTask("cleanup")
    job.UpdateStatus("running")
    job.SetConfig("timeout", "300s")

    fmt.Printf("Updated summary: %s\n", job.GetSummary())
    fmt.Printf("Log entries: %d\n", len(job.Logs))
}

func demonstrateSemanticConsistency() {
    fmt.Println("\n=== Semantic Consistency Guidelines ===")

    fmt.Println("Type Classification:")
    fmt.Println("1. Built-in types (int, string, bool, float64)")
    fmt.Println("   →’ Always use VALUE semantics")
    fmt.Println("   →’ Small, immutable, efficient to copy")

    fmt.Println("2. Reference types (slice, map, channel, interface)")
    fmt.Println("   →’ Use VALUE semantics")
    fmt.Println("   →’ Header is small, data is already referenced")
    fmt.Println("   →’ Mutations can use sandbox pattern")

    fmt.Println("3. User-defined struct types")
    fmt.Println("   →’ Choose based on data characteristics:")
    fmt.Println("   →’ Small, simple data →’ VALUE semantics")
    fmt.Println("   →’ Large, complex data →’ POINTER semantics")
    fmt.Println("   →’ Consistency: if any method uses pointer, all should")

    fmt.Println("\nSemantic Consistency Rules:")
    fmt.Println("- Type drives semantics, not the operation")
    fmt.Println("- Be consistent within a type")
    fmt.Println("- Follow standard library patterns")
    fmt.Println("- Consider the cost of copying")
    fmt.Println("- Mutation APIs can use value semantics (sandbox)")
}
```

**Hands-on Exercise 3: Standard Library Semantic Patterns**:

```go
// Following standard library semantic patterns
package main

import (
    "fmt"
    "net/url"
    "strings"
    "time"
)

// ========== Following time.Time Pattern (Value Semantics) ==========

// Duration wrapper following time.Duration pattern
type ProcessingDuration time.Duration

func (pd ProcessingDuration) String() string {
    return time.Duration(pd).String()
}

func (pd ProcessingDuration) Seconds() float64 {
    return time.Duration(pd).Seconds()
}

func (pd ProcessingDuration) Add(other ProcessingDuration) ProcessingDuration {
    return ProcessingDuration(time.Duration(pd) + time.Duration(other))
}

// ========== Following url.URL Pattern (Pointer Semantics) ==========

// ServiceEndpoint following url.URL pattern (complex struct)
type ServiceEndpoint struct {
    Scheme   string
    Host     string
    Port     int
    Path     string
    Query    map[string]string
    Fragment string
}

func (se *ServiceEndpoint) String() string {
    result := fmt.Sprintf("%s://%s", se.Scheme, se.Host)
    if se.Port != 0 {
        result += fmt.Sprintf(":%d", se.Port)
    }
    if se.Path != "" {
        result += se.Path
    }
    if len(se.Query) > 0 {
        result += "?"
        var parts []string
        for k, v := range se.Query {
            parts = append(parts, fmt.Sprintf("%s=%s", k, v))
        }
        result += strings.Join(parts, "&")
    }
    if se.Fragment != "" {
        result += "#" + se.Fragment
    }
    return result
}

func (se *ServiceEndpoint) SetQuery(key, value string) {
    if se.Query == nil {
        se.Query = make(map[string]string)
    }
    se.Query[key] = value
}

func (se *ServiceEndpoint) GetQuery(key string) string {
    if se.Query == nil {
        return ""
    }
    return se.Query[key]
}

// ========== Following strings.Builder Pattern (Pointer Semantics) ==========

// LogBuilder following strings.Builder pattern
type LogBuilder struct {
    entries []string
    level   string
    service string
}

func (lb *LogBuilder) WriteEntry(message string) {
    timestamp := time.Now().Format("15:04:05")
    entry := fmt.Sprintf("[%s] %s %s: %s",
        timestamp, lb.level, lb.service, message)
    lb.entries = append(lb.entries, entry)
}

func (lb *LogBuilder) SetLevel(level string) {
    lb.level = level
}

func (lb *LogBuilder) SetService(service string) {
    lb.service = service
}

func (lb *LogBuilder) String() string {
    return strings.Join(lb.entries, "\n")
}

func (lb *LogBuilder) Reset() {
    lb.entries = lb.entries[:0]
    lb.level = ""
    lb.service = ""
}

// ========== Following slice patterns (Value Semantics) ==========

// TaskList following slice patterns
type TaskList []Task

type Task struct {
    ID       string
    Name     string
    Priority int
}

func (tl TaskList) Len() int {
    return len(tl)
}

func (tl TaskList) Filter(predicate func(Task) bool) TaskList {
    var filtered TaskList
    for _, task := range tl {
        if predicate(task) {
            filtered = append(filtered, task)
        }
    }
    return filtered
}

func (tl TaskList) Sort() TaskList {
    // Create copy for sorting
    sorted := make(TaskList, len(tl))
    copy(sorted, tl)

    // Simple bubble sort by priority
    for i := 0; i < len(sorted); i++ {
        for j := 0; j < len(sorted)-1-i; j++ {
            if sorted[j].Priority < sorted[j+1].Priority {
                sorted[j], sorted[j+1] = sorted[j+1], sorted[j]
            }
        }
    }

    return sorted
}

func (tl TaskList) Find(id string) (Task, bool) {
    for _, task := range tl {
        if task.ID == id {
            return task, true
        }
    }
    return Task{}, false
}

func main() {
    demonstrateTimePattern()
    demonstrateURLPattern()
    demonstrateBuilderPattern()
    demonstrateSlicePattern()
}

func demonstrateTimePattern() {
    fmt.Println("=== Following time.Time Pattern (Value Semantics) ===")

    // ProcessingDuration follows time.Duration pattern
    duration1 := ProcessingDuration(5 * time.Second)
    duration2 := ProcessingDuration(3 * time.Second)

    fmt.Printf("Duration 1: %s (%.1f seconds)\n",
        duration1.String(), duration1.Seconds())

    total := duration1.Add(duration2)
    fmt.Printf("Total duration: %s\n", total.String())

    // Original values unchanged (value semantics)
    fmt.Printf("Original duration1: %s\n", duration1.String())

    fmt.Println("✅“ Follows time.Time pattern: value semantics, immutable")
}

func demonstrateURLPattern() {
    fmt.Println("\n=== Following url.URL Pattern (Pointer Semantics) ===")

    // ServiceEndpoint follows url.URL pattern
    endpoint := &ServiceEndpoint{
        Scheme: "https",
        Host:   "api.example.com",
        Port:   443,
        Path:   "/v1/automation",
    }

    fmt.Printf("Initial endpoint: %s\n", endpoint.String())

    // Mutations modify the original (pointer semantics)
    endpoint.SetQuery("version", "2.0")
    endpoint.SetQuery("format", "json")

    fmt.Printf("After adding query params: %s\n", endpoint.String())
    fmt.Printf("Version param: %s\n", endpoint.GetQuery("version"))

    fmt.Println("✅“ Follows url.URL pattern: pointer semantics, mutable")
}

func demonstrateBuilderPattern() {
    fmt.Println("\n=== Following strings.Builder Pattern (Pointer Semantics) ===")

    // LogBuilder follows strings.Builder pattern
    builder := &LogBuilder{}
    builder.SetLevel("INFO")
    builder.SetService("AutomationService")

    builder.WriteEntry("Service started")
    builder.WriteEntry("Processing batch 1")
    builder.WriteEntry("Processing batch 2")

    fmt.Println("Log output:")
    fmt.Println(builder.String())

    // Change level and add more entries
    builder.SetLevel("ERROR")
    builder.WriteEntry("Processing failed")

    fmt.Println("\nAfter adding error:")
    fmt.Println(builder.String())

    // Reset and reuse
    builder.Reset()
    builder.SetLevel("DEBUG")
    builder.SetService("TestService")
    builder.WriteEntry("Debug message")

    fmt.Println("\nAfter reset:")
    fmt.Println(builder.String())

    fmt.Println("✅“ Follows strings.Builder pattern: pointer semantics, efficient building")
}

func demonstrateSlicePattern() {
    fmt.Println("\n=== Following Slice Patterns (Value Semantics) ===")

    // TaskList follows slice patterns
    tasks := TaskList{
        {ID: "T1", Name: "Setup", Priority: 1},
        {ID: "T2", Name: "Process", Priority: 3},
        {ID: "T3", Name: "Validate", Priority: 2},
        {ID: "T4", Name: "Cleanup", Priority: 1},
    }

    fmt.Printf("Total tasks: %d\n", tasks.Len())

    // Filter high priority tasks (returns new slice)
    highPriority := tasks.Filter(func(t Task) bool {
        return t.Priority >= 2
    })

    fmt.Printf("High priority tasks: %d\n", highPriority.Len())
    for _, task := range highPriority {
        fmt.Printf("  %s: %s (priority %d)\n", task.ID, task.Name, task.Priority)
    }

    // Sort tasks (returns new slice)
    sorted := tasks.Sort()
    fmt.Println("\nSorted by priority:")
    for _, task := range sorted {
        fmt.Printf("  %s: %s (priority %d)\n", task.ID, task.Name, task.Priority)
    }

    // Find specific task
    if task, found := tasks.Find("T2"); found {
        fmt.Printf("\nFound task: %s - %s\n", task.ID, task.Name)
    }

    // Original slice unchanged
    fmt.Printf("\nOriginal tasks unchanged: %d items\n", tasks.Len())

    fmt.Println("✅“ Follows slice patterns: value semantics, functional style")
}
```

**Prerequisites**: Module 15
