# Module 20: Embedding, Exporting, and Composition Patterns

**Duration**: 1 hour (45 min theory + 15 min hands-on)

**Learning Objectives**:

- Master embedding for composition and method promotion
- Understand exporting rules and package visibility
- Learn composition patterns vs class-based inheritance
- Apply embedding in automation system architecture
- Design clean APIs with proper encapsulation

**Videos Covered**:

- 4.8 Embedding (0:07:30)
- 4.9 Exporting (0:08:29)

**Key Concepts**:

- Embedding: composition through anonymous fields
- Method promotion: embedded type methods become available
- Inner type promotion and method set expansion
- Exporting: uppercase names are public, lowercase are private
- Package-level encapsulation and API design
- Composition over class-based inheritance patterns

**Hands-on Exercise 1: Basic Embedding and Method Promotion**:

```go
// Basic embedding and method promotion in automation systems
package main

import (
    "fmt"
    "time"
)

// Base types for embedding

// Logger provides basic logging capability
type Logger struct {
    prefix string
    level  string
}

func (l *Logger) Log(message string) {
    timestamp := time.Now().Format("15:04:05")
    fmt.Printf("[%s] %s %s: %s\n", timestamp, l.prefix, l.level, message)
}

func (l *Logger) SetLevel(level string) {
    l.level = level
}

// Metrics provides performance tracking
type Metrics struct {
    startTime time.Time
    counter   int64
}

func (m *Metrics) Start() {
    m.startTime = time.Now()
    m.counter = 0
}

func (m *Metrics) Increment() {
    m.counter++
}

func (m *Metrics) GetStats() (time.Duration, int64) {
    return time.Since(m.startTime), m.counter
}

// AutomationTask demonstrates embedding
type AutomationTask struct {
    // Embedded types - anonymous fields create inner/outer type relationship
    *Logger  // Pointer embedding
    Metrics  // Value embedding

    // Own fields
    ID          string
    Description string
    Status      string
}

func NewAutomationTask(id, description string) *AutomationTask {
    return &AutomationTask{
        Logger: &Logger{
            prefix: fmt.Sprintf("TASK-%s", id),
            level:  "INFO",
        },
        Metrics:     Metrics{},
        ID:          id,
        Description: description,
        Status:      "created",
    }
}

// Own method that uses embedded functionality
func (at *AutomationTask) Execute() error {
    // Inner type promotion allows direct access to embedded methods
    at.Log("Starting task execution")  // From embedded Logger
    at.Start()                        // From embedded Metrics

    at.Status = "running"
    at.Log(fmt.Sprintf("Executing: %s", at.Description))

    // Simulate work
    for i := 0; i < 5; i++ {
        time.Sleep(50 * time.Millisecond)
        at.Increment() // From embedded Metrics
        at.Log(fmt.Sprintf("Progress: %d/5", i+1))
    }

    at.Status = "completed"
    duration, count := at.GetStats() // From embedded Metrics
    at.Log(fmt.Sprintf("Task completed in %v with %d operations", duration, count))

    return nil
}

// Method shadowing - outer type overrides inner type method
func (at *AutomationTask) Log(message string) {
    // Can still access inner type method explicitly
    at.Logger.Log(fmt.Sprintf("[%s] %s", at.Status, message))
}

// Demonstrate interface satisfaction through embedding
type Notifier interface {
    Notify(message string)
}

// User type implements Notifier
type User struct {
    Name  string
    Email string
}

func (u *User) Notify(message string) {
    fmt.Printf("Notifying %s (%s): %s\n", u.Name, u.Email, message)
}

// Admin embeds User - gets Notifier interface satisfaction through promotion
type Admin struct {
    *User // Embedded pointer to User
    Level string
}

func NewAdmin(name, email, level string) *Admin {
    return &Admin{
        User:  &User{Name: name, Email: email},
        Level: level,
    }
}

// Admin can override User's Notify method
func (a *Admin) Notify(message string) {
    fmt.Printf("ADMIN ALERT to %s (%s) [Level: %s]: %s\n",
        a.Name, a.Email, a.Level, message)
}

// Polymorphic function that accepts any Notifier
func SendNotification(n Notifier, message string) {
    n.Notify(message)
}

func main() {
    fmt.Println("=== Basic Embedding and Method Promotion ===")

    // Create automation task
    task := NewAutomationTask("001", "Process daily reports")

    // Embedded methods are promoted and directly accessible
    fmt.Printf("Task ID: %s\n", task.ID)
    task.SetLevel("DEBUG") // From embedded Logger (promoted)

    // Execute task using embedded functionality
    if err := task.Execute(); err != nil {
        fmt.Printf("Task execution failed: %v\n", err)
    }

    fmt.Println("\n=== Interface Satisfaction Through Embedding ===")

    // Create user and admin
    user := &User{Name: "John Doe", Email: "john@example.com"}
    admin := NewAdmin("Jane Smith", "jane@example.com", "SuperAdmin")

    // Both satisfy Notifier interface
    SendNotification(user, "System maintenance scheduled")
    SendNotification(admin, "Critical security alert")

    // Direct method calls show promotion and shadowing
    fmt.Println("\nDirect method calls:")
    user.Notify("Direct user notification")
    admin.Notify("Direct admin notification")

    // Access inner type method explicitly
    fmt.Println("\nAccessing inner type method explicitly:")
    admin.User.Notify("Calling User.Notify through admin")

    fmt.Println("\n=== Embedding Mechanics Summary ===")
    fmt.Println("- Embedding creates inner/outer type relationship")
    fmt.Println("- Inner type methods are promoted to outer type")
    fmt.Println("- Outer type can override inner type methods")
    fmt.Println("- Interface satisfaction works through promotion")
    fmt.Println("- No subtyping relationship is created")
}
```

**Hands-on Exercise 2: Exporting Rules and Package Visibility**:

```go
// Exporting rules and encapsulation patterns
package main

import (
    "fmt"
    "time"
)

// EXPORTED TYPES (start with capital letter)

// ServiceConfig represents automation service configuration
// Exported type with mixed field visibility
type ServiceConfig struct {
    Name        string        // Exported field
    Port        int          // Exported field
    timeout     time.Duration // unexported field - package-level encapsulation
    apiKey      string       // unexported field - sensitive data
    IsActive    bool         // Exported field
}

// Exported constructor function
func NewServiceConfig(name string, port int) *ServiceConfig {
    return &ServiceConfig{
        Name:     name,
        Port:     port,
        timeout:  30 * time.Second, // Set default for unexported field
        apiKey:   generateAPIKey(),  // Internal logic for unexported field
        IsActive: true,
    }
}

// Exported methods for accessing unexported fields
func (sc *ServiceConfig) GetTimeout() time.Duration {
    return sc.timeout
}

func (sc *ServiceConfig) SetTimeout(timeout time.Duration) {
    if timeout > 0 && timeout <= 5*time.Minute {
        sc.timeout = timeout
    }
}

func (sc *ServiceConfig) GetAPIKey() string {
    // Return masked version for security
    if len(sc.apiKey) > 8 {
        return sc.apiKey[:4] + "****" + sc.apiKey[len(sc.apiKey)-4:]
    }
    return "****"
}

// unexported method - internal use only
func (sc *ServiceConfig) validateConfig() bool {
    return sc.Name != "" && sc.Port > 0 && sc.Port < 65536
}

// unexported helper function
func generateAPIKey() string {
    return fmt.Sprintf("key_%d", time.Now().Unix())
}

// UNEXPORTED TYPES (start with lowercase letter)

// automationMetrics is unexported - only accessible within package
type automationMetrics struct {
    RequestCount int64         // Exported field (for JSON marshaling)
    ErrorCount   int64         // Exported field
    startTime    time.Time     // unexported field
    lastRequest  time.Time     // unexported field
}

// Exported constructor for unexported type
func NewMetrics() *automationMetrics {
    return &automationMetrics{
        RequestCount: 0,
        ErrorCount:   0,
        startTime:    time.Now(),
        lastRequest:  time.Now(),
    }
}

// Exported methods on unexported type
func (am *automationMetrics) RecordRequest() {
    am.RequestCount++
    am.lastRequest = time.Now()
}

func (am *automationMetrics) RecordError() {
    am.ErrorCount++
}

func (am *automationMetrics) GetUptime() time.Duration {
    return time.Since(am.startTime)
}

// Mixed visibility with embedding

// baseService is unexported type
type baseService struct {
    ID          string    // Exported field
    CreatedAt   time.Time // Exported field
    version     string    // unexported field
}

func (bs *baseService) GetVersion() string {
    return bs.version
}

// unexported method
func (bs *baseService) updateVersion(version string) {
    bs.version = version
}

// AutomationService embeds unexported type
type AutomationService struct {
    baseService              // Embedded unexported type
    Config      *ServiceConfig // Exported field
    metrics     *automationMetrics // unexported field
}

func NewAutomationService(name string, port int) *AutomationService {
    return &AutomationService{
        baseService: baseService{
            ID:        fmt.Sprintf("svc_%d", time.Now().Unix()),
            CreatedAt: time.Now(),
            version:   "1.0.0",
        },
        Config:  NewServiceConfig(name, port),
        metrics: NewMetrics(),
    }
}

// Exported method
func (as *AutomationService) Start() error {
    if !as.Config.validateConfig() { // Can access unexported method
        return fmt.Errorf("invalid configuration")
    }

    as.metrics.RecordRequest()
    fmt.Printf("Starting service %s on port %d\n", as.Config.Name, as.Config.Port)
    return nil
}

// Exported method that provides controlled access to unexported field
func (as *AutomationService) GetMetrics() (int64, int64, time.Duration) {
    return as.metrics.RequestCount, as.metrics.ErrorCount, as.metrics.GetUptime()
}

// Exported method
func (as *AutomationService) ProcessRequest() error {
    as.metrics.RecordRequest()

    // Simulate processing
    time.Sleep(10 * time.Millisecond)

    // Simulate occasional errors
    if as.metrics.RequestCount%10 == 0 {
        as.metrics.RecordError()
        return fmt.Errorf("simulated processing error")
    }

    return nil
}

func main() {
    fmt.Println("=== Exporting Rules Demonstration ===")

    // Create service using exported constructor
    service := NewAutomationService("DataProcessor", 8080)

    // Access exported fields directly
    fmt.Printf("Service ID: %s\n", service.ID)           // From embedded baseService
    fmt.Printf("Service Name: %s\n", service.Config.Name) // From exported Config field
    fmt.Printf("Service Port: %d\n", service.Config.Port)

    // Cannot access unexported fields directly:
    // fmt.Println(service.Config.timeout)  // Compile error!
    // fmt.Println(service.Config.apiKey)   // Compile error!
    // fmt.Println(service.metrics)         // Compile error!

    // Must use exported methods to access unexported data
    fmt.Printf("Timeout: %v\n", service.Config.GetTimeout())
    fmt.Printf("API Key: %s\n", service.Config.GetAPIKey())

    // Start service
    if err := service.Start(); err != nil {
        fmt.Printf("Failed to start service: %v\n", err)
        return
    }

    fmt.Println("\n=== Processing Requests ===")

    // Process some requests
    for i := 0; i < 15; i++ {
        if err := service.ProcessRequest(); err != nil {
            fmt.Printf("Request %d failed: %v\n", i+1, err)
        } else {
            fmt.Printf("Request %d processed successfully\n", i+1)
        }
    }

    // Get metrics through exported method
    requests, errors, uptime := service.GetMetrics()
    fmt.Printf("\nService Statistics:\n")
    fmt.Printf("  Total Requests: %d\n", requests)
    fmt.Printf("  Total Errors: %d\n", errors)
    fmt.Printf("  Uptime: %v\n", uptime)
    fmt.Printf("  Success Rate: %.1f%%\n", float64(requests-errors)/float64(requests)*100)

    fmt.Println("\n=== Encapsulation Summary ===")
    fmt.Println("- Exported identifiers (Capital): accessible outside package")
    fmt.Println("- unexported identifiers (lowercase): package-level encapsulation")
    fmt.Println("- Encapsulation is about identifier accessibility, not data privacy")
    fmt.Println("- Use exported methods to provide controlled access to unexported data")
    fmt.Println("- Embedded unexported types promote their exported fields")
}
```

**Hands-on Exercise 3: Advanced Composition Patterns with Embedding**:

```go
// Advanced composition patterns using embedding and interfaces
package main

import (
    "fmt"
    "time"
)

// Define behavior contracts (interfaces)
type Processor interface {
    Process(data string) (string, error)
}

type Validator interface {
    Validate(data string) error
}

type Logger interface {
    Log(message string)
}

type Metrics interface {
    RecordOperation(operation string, duration time.Duration)
    GetStats() map[string]interface{}
}

// Concrete implementations

// FileProcessor implements Processor
type FileProcessor struct {
    name      string
    directory string
}

func (fp *FileProcessor) Process(data string) (string, error) {
    // Simulate file processing
    time.Sleep(10 * time.Millisecond)
    return fmt.Sprintf("file_processed_%s_%s", fp.name, data), nil
}

// DataValidator implements Validator
type DataValidator struct {
    rules []string
}

func (dv *DataValidator) Validate(data string) error {
    if len(data) == 0 {
        return fmt.Errorf("data cannot be empty")
    }
    if len(data) > 100 {
        return fmt.Errorf("data too long (max 100 characters)")
    }
    return nil
}

// SimpleLogger implements Logger
type SimpleLogger struct {
    prefix string
}

func (sl *SimpleLogger) Log(message string) {
    timestamp := time.Now().Format("15:04:05")
    fmt.Printf("[%s] %s: %s\n", timestamp, sl.prefix, message)
}

// PerformanceMetrics implements Metrics
type PerformanceMetrics struct {
    operations map[string]int
    totalTime  map[string]time.Duration
}

func NewPerformanceMetrics() *PerformanceMetrics {
    return &PerformanceMetrics{
        operations: make(map[string]int),
        totalTime:  make(map[string]time.Duration),
    }
}

func (pm *PerformanceMetrics) RecordOperation(operation string, duration time.Duration) {
    pm.operations[operation]++
    pm.totalTime[operation] += duration
}

func (pm *PerformanceMetrics) GetStats() map[string]interface{} {
    stats := make(map[string]interface{})
    for op, count := range pm.operations {
        avgTime := pm.totalTime[op] / time.Duration(count)
        stats[op] = map[string]interface{}{
            "count":        count,
            "total_time":   pm.totalTime[op],
            "average_time": avgTime,
        }
    }
    return stats
}

// Composition through embedding interfaces

// AutomationEngine composes behavior through embedded interfaces
type AutomationEngine struct {
    // Embed interfaces - composition through behavior contracts
    Processor
    Validator
    Logger
    Metrics

    // Own state
    Name     string
    isActive bool
}

// Constructor that accepts interface implementations
func NewAutomationEngine(name string, p Processor, v Validator, l Logger, m Metrics) *AutomationEngine {
    return &AutomationEngine{
        Processor: p,
        Validator: v,
        Logger:    l,
        Metrics:   m,
        Name:      name,
        isActive:  true,
    }
}

// High-level method that orchestrates embedded behaviors
func (ae *AutomationEngine) ProcessData(data string) (string, error) {
    start := time.Now()

    ae.Log(fmt.Sprintf("Starting data processing for: %s", data))

    // Use embedded Validator
    if err := ae.Validate(data); err != nil {
        ae.Log(fmt.Sprintf("Validation failed: %v", err))
        ae.RecordOperation("validation_failed", time.Since(start))
        return "", err
    }

    // Use embedded Processor
    result, err := ae.Process(data)
    if err != nil {
        ae.Log(fmt.Sprintf("Processing failed: %v", err))
        ae.RecordOperation("processing_failed", time.Since(start))
        return "", err
    }

    duration := time.Since(start)
    ae.RecordOperation("successful_processing", duration)
    ae.Log(fmt.Sprintf("Processing completed successfully in %v", duration))

    return result, nil
}

// Method to replace embedded components (composition flexibility)
func (ae *AutomationEngine) SetProcessor(p Processor) {
    ae.Processor = p
    ae.Log("Processor component replaced")
}

func (ae *AutomationEngine) SetValidator(v Validator) {
    ae.Validator = v
    ae.Log("Validator component replaced")
}

// Advanced embedding with concrete types and interfaces

// Enhanced automation service that embeds both concrete types and interfaces
type EnhancedAutomationService struct {
    // Embed concrete type for base functionality
    *SimpleLogger

    // Embed interfaces for flexible composition
    Processor
    Validator
    Metrics

    // Own fields
    ServiceID   string
    Config      map[string]interface{}
    startTime   time.Time
}

func NewEnhancedService(id string, processor Processor, validator Validator) *EnhancedAutomationService {
    return &EnhancedAutomationService{
        SimpleLogger: &SimpleLogger{prefix: fmt.Sprintf("SVC-%s", id)},
        Processor:    processor,
        Validator:    validator,
        Metrics:      NewPerformanceMetrics(),
        ServiceID:    id,
        Config:       make(map[string]interface{}),
        startTime:    time.Now(),
    }
}

// Override embedded Logger method to add service context
func (eas *EnhancedAutomationService) Log(message string) {
    // Call embedded SimpleLogger method with additional context
    eas.SimpleLogger.Log(fmt.Sprintf("[%s] %s", eas.ServiceID, message))
}

// Batch processing method that demonstrates composition
func (eas *EnhancedAutomationService) ProcessBatch(dataItems []string) []string {
    eas.Log(fmt.Sprintf("Starting batch processing of %d items", len(dataItems)))

    var results []string
    successCount := 0

    for i, data := range dataItems {
        start := time.Now()

        // Use embedded interfaces
        if err := eas.Validate(data); err != nil {
            eas.Log(fmt.Sprintf("Item %d validation failed: %v", i+1, err))
            eas.RecordOperation("batch_validation_failed", time.Since(start))
            continue
        }

        result, err := eas.Process(data)
        if err != nil {
            eas.Log(fmt.Sprintf("Item %d processing failed: %v", i+1, err))
            eas.RecordOperation("batch_processing_failed", time.Since(start))
            continue
        }

        results = append(results, result)
        successCount++
        eas.RecordOperation("batch_item_success", time.Since(start))
    }

    eas.Log(fmt.Sprintf("Batch processing completed: %d/%d successful", successCount, len(dataItems)))
    return results
}

func (eas *EnhancedAutomationService) GetServiceStats() map[string]interface{} {
    stats := eas.GetStats() // From embedded Metrics
    stats["service_id"] = eas.ServiceID
    stats["uptime"] = time.Since(eas.startTime)
    return stats
}

func main() {
    fmt.Println("=== Advanced Composition with Embedding ===")

    // Create concrete implementations
    processor := &FileProcessor{name: "MainProcessor", directory: "/data"}
    validator := &DataValidator{rules: []string{"not_empty", "max_length"}}
    logger := &SimpleLogger{prefix: "ENGINE"}
    metrics := NewPerformanceMetrics()

    // Create automation engine through composition
    engine := NewAutomationEngine("DataEngine", processor, validator, logger, metrics)

    // Test data processing
    testData := []string{"sample_data", "test_input", "", "very_long_data_that_might_exceed_limits"}

    fmt.Println("Processing individual items:")
    for _, data := range testData {
        result, err := engine.ProcessData(data)
        if err != nil {
            fmt.Printf("  Failed to process '%s': %v\n", data, err)
        } else {
            fmt.Printf("  Successfully processed '%s' -> '%s'\n", data, result)
        }
    }

    // Show composition flexibility - replace components
    fmt.Println("\n=== Composition Flexibility ===")

    newProcessor := &FileProcessor{name: "AdvancedProcessor", directory: "/advanced"}
    engine.SetProcessor(newProcessor)

    result, _ := engine.ProcessData("flexibility_test")
    fmt.Printf("With new processor: %s\n", result)

    // Display metrics
    fmt.Println("\n=== Performance Metrics ===")
    stats := engine.GetStats()
    for operation, data := range stats {
        fmt.Printf("Operation: %s\n", operation)
        if opStats, ok := data.(map[string]interface{}); ok {
            for key, value := range opStats {
                fmt.Printf("  %s: %v\n", key, value)
            }
        }
        fmt.Println()
    }

    fmt.Println("=== Enhanced Service with Mixed Embedding ===")

    // Create enhanced service
    enhancedService := NewEnhancedService("ENH001", processor, validator)

    // Batch processing
    batchData := []string{"item1", "item2", "", "item4", "item5"}
    results := enhancedService.ProcessBatch(batchData)

    fmt.Printf("Batch results: %v\n", results)

    // Service statistics
    fmt.Println("\nService Statistics:")
    serviceStats := enhancedService.GetServiceStats()
    for key, value := range serviceStats {
        fmt.Printf("  %s: %v\n", key, value)
    }

    fmt.Println("\n=== Composition Patterns Summary ===")
    fmt.Println("- Embedding interfaces enables flexible composition")
    fmt.Println("- Components can be replaced at runtime")
    fmt.Println("- Concrete embedding provides base functionality")
    fmt.Println("- Interface embedding provides behavioral contracts")
    fmt.Println("- Method shadowing allows customization of embedded behavior")
    fmt.Println("- Composition over class-based inheritance creates flexible, testable designs")
}
```

**Prerequisites**: Module 19
