# Module 22: Interface Conversions, Assertions, and Design Guidelines

**Duration**: 1 hour (45 min theory + 15 min hands-on)

**Learning Objectives**:

- Master interface conversions and type assertions
- Understand interface pollution and when to avoid it
- Learn interface design guidelines and best practices
- Apply clean interface design in automation systems
- Recognize and prevent interface anti-patterns

**Videos Covered**:

- 5.6 Conversion and Assertions (0:09:02)
- 5.7 Interface Pollution (0:06:45)
- 5.9 Design Guidelines (0:03:25)

**Key Concepts**:

- Type assertions: extracting concrete types from interfaces
- Interface conversions: moving between interface types
- Interface pollution: over-abstraction and premature interfaces
- Design guidelines: start concrete, move to abstract when needed
- Interface segregation: small, focused interfaces
- Avoid interfaces for the sake of interfaces

**Hands-on Exercise 1: Type Assertions and Interface Conversions**:

```go
// Demonstrating type assertions and interface conversions in automation systems
package main

import (
    "fmt"
    "io"
    "strings"
)

// Define behavior interfaces for automation components
type Mover interface {
    Move(destination string) error
}

type Locker interface {
    Lock() error
    Unlock() error
}

// Composed interface - moveLocker can move, lock, and unlock
type MoveLocker interface {
    Mover
    Locker
}

// Concrete automation component
type AutomationRobot struct {
    ID       string
    Location string
    IsLocked bool
}

func (ar *AutomationRobot) Move(destination string) error {
    if ar.IsLocked {
        return fmt.Errorf("robot %s is locked, cannot move", ar.ID)
    }
    fmt.Printf("Robot %s moving from %s to %s\n", ar.ID, ar.Location, destination)
    ar.Location = destination
    return nil
}

func (ar *AutomationRobot) Lock() error {
    ar.IsLocked = true
    fmt.Printf("Robot %s locked at %s\n", ar.ID, ar.Location)
    return nil
}

func (ar *AutomationRobot) Unlock() error {
    ar.IsLocked = false
    fmt.Printf("Robot %s unlocked at %s\n", ar.ID, ar.Location)
    return nil
}

// Additional behavior for some robots
type StatusReporter interface {
    GetStatus() string
}

func (ar *AutomationRobot) GetStatus() string {
    status := "unlocked"
    if ar.IsLocked {
        status = "locked"
    }
    return fmt.Sprintf("Robot %s at %s (%s)", ar.ID, ar.Location, status)
}

// Another concrete type that implements different interfaces
type AutomationDrone struct {
    ID       string
    Altitude int
    IsActive bool
}

func (ad *AutomationDrone) Move(destination string) error {
    if !ad.IsActive {
        return fmt.Errorf("drone %s is not active", ad.ID)
    }
    fmt.Printf("Drone %s flying to %s at altitude %d\n", ad.ID, destination, ad.Altitude)
    return nil
}

func (ad *AutomationDrone) GetStatus() string {
    status := "inactive"
    if ad.IsActive {
        status = "active"
    }
    return fmt.Sprintf("Drone %s at altitude %d (%s)", ad.ID, ad.Altitude, status)
}

// Demonstrate interface conversions
func demonstrateInterfaceConversions() {
    fmt.Println("=== Interface Conversions ===")

    // Create a robot that implements MoveLocker
    robot := &AutomationRobot{ID: "R001", Location: "Station A", IsLocked: false}

    // Store in MoveLocker interface
    var moveLocker MoveLocker = robot

    // Interface conversion: MoveLocker -> Mover (allowed - subset of behaviors)
    var mover Mover = moveLocker
    mover.Move("Station B")

    // Interface conversion: MoveLocker -> Locker (allowed - subset of behaviors)
    var locker Locker = moveLocker
    locker.Lock()

    // Cannot convert Mover -> MoveLocker (compile error - not enough behaviors)
    // var moveLocker2 MoveLocker = mover // This would fail!

    fmt.Println("Interface conversions allow moving from larger to smaller interfaces")
}

// Demonstrate type assertions
func demonstrateTypeAssertions() {
    fmt.Println("\n=== Type Assertions ===")

    // Create different automation components
    robot := &AutomationRobot{ID: "R002", Location: "Station C", IsLocked: false}
    drone := &AutomationDrone{ID: "D001", Altitude: 100, IsActive: true}

    // Store in interface slice
    movers := []Mover{robot, drone}

    for i, mover := range movers {
        fmt.Printf("\nProcessing mover %d:\n", i+1)

        // Type assertion with ok pattern (safe)
        if robot, ok := mover.(*AutomationRobot); ok {
            fmt.Printf("  Found robot: %s\n", robot.ID)
            // Can access robot-specific functionality
            robot.Lock()
            robot.Unlock()
        }

        if drone, ok := mover.(*AutomationDrone); ok {
            fmt.Printf("  Found drone: %s\n", drone.ID)
            // Can access drone-specific functionality
            drone.Altitude = 150
        }

        // Type assertion to interface (checking for additional behaviors)
        if reporter, ok := mover.(StatusReporter); ok {
            fmt.Printf("  Status: %s\n", reporter.GetStatus())
        }
    }
}

// Demonstrate type switch
func demonstrateTypeSwitch() {
    fmt.Println("\n=== Type Switch ===")

    components := []interface{}{
        &AutomationRobot{ID: "R003", Location: "Station D", IsLocked: true},
        &AutomationDrone{ID: "D002", Altitude: 200, IsActive: false},
        "invalid component", // Different type to show default case
    }

    for i, component := range components {
        fmt.Printf("\nAnalyzing component %d:\n", i+1)

        switch v := component.(type) {
        case *AutomationRobot:
            fmt.Printf("  Robot %s at %s (locked: %t)\n", v.ID, v.Location, v.IsLocked)
            if !v.IsLocked {
                v.Move("Maintenance Bay")
            }

        case *AutomationDrone:
            fmt.Printf("  Drone %s at altitude %d (active: %t)\n", v.ID, v.Altitude, v.IsActive)
            if v.IsActive {
                v.Move("Patrol Zone")
            }

        default:
            fmt.Printf("  Unknown component type: %T\n", v)
        }
    }
}

// Real-world example: optimized copy behavior using type assertions
func demonstrateOptimizedCopy() {
    fmt.Println("\n=== Optimized Copy Pattern (like io.Copy) ===")

    // Custom reader that can optimize copying
    type OptimizedReader struct {
        data string
    }

    func (or *OptimizedReader) Read(p []byte) (n int, err error) {
        if len(or.data) == 0 {
            return 0, io.EOF
        }
        n = copy(p, or.data)
        or.data = or.data[n:]
        return n, nil
    }

    // Optional optimization interface
    type WriterTo interface {
        WriteTo(w io.Writer) (n int64, err error)
    }

    func (or *OptimizedReader) WriteTo(w io.Writer) (n int64, err error) {
        fmt.Println("  Using optimized WriteTo method!")
        written, err := w.Write([]byte(or.data))
        or.data = ""
        return int64(written), err
    }

    // Custom copy function that checks for optimization
    customCopy := func(dst io.Writer, src io.Reader) (written int64, err error) {
        // Type assertion to check for optimization
        if wt, ok := src.(WriterTo); ok {
            return wt.WriteTo(dst)
        }

        // Default behavior
        fmt.Println("  Using default copy behavior")
        buf := make([]byte, 32*1024)
        for {
            nr, er := src.Read(buf)
            if nr > 0 {
                nw, ew := dst.Write(buf[0:nr])
                written += int64(nw)
                if ew != nil {
                    err = ew
                    break
                }
            }
            if er != nil {
                if er != io.EOF {
                    err = er
                }
                break
            }
        }
        return written, err
    }

    // Test with optimized reader
    optimizedReader := &OptimizedReader{data: "Hello from optimized reader!"}
    var output strings.Builder

    fmt.Println("Copying with optimization check:")
    customCopy(&output, optimizedReader)
    fmt.Printf("Result: %s\n", output.String())
}

func main() {
    fmt.Println("=== Type Assertions and Interface Conversions ===")

    demonstrateInterfaceConversions()
    demonstrateTypeAssertions()
    demonstrateTypeSwitch()
    demonstrateOptimizedCopy()

    fmt.Println("\n=== Key Concepts Summary ===")
    fmt.Println("1. Interface conversions: larger interface -> smaller interface")
    fmt.Println("2. Type assertions: extract concrete types from interfaces")
    fmt.Println("3. Type assertions with 'ok' pattern prevent panics")
    fmt.Println("4. Type switch handles multiple possible types")
    fmt.Println("5. Type assertions enable API optimization patterns")
    fmt.Println("6. All happen at runtime, not compile time")
}
```

**Hands-on Exercise 2: Interface Pollution and Design Anti-Patterns**:

```go
// Demonstrating interface pollution and how to avoid it
package main

import (
    "fmt"
    "time"
)

// BAD EXAMPLE: Interface pollution - this is what NOT to do

// SMELL 1: Interface named after a "thing" not a behavior
type Server interface {
    Start() error
    Stop() error
    Wait() error
}

// SMELL 2: Unexported concrete type with exported interface
type tcpServer struct {
    port     int
    isActive bool
}

func (ts *tcpServer) Start() error {
    ts.isActive = true
    fmt.Printf("TCP server starting on port %d\n", ts.port)
    return nil
}

func (ts *tcpServer) Stop() error {
    ts.isActive = false
    fmt.Printf("TCP server stopping\n")
    return nil
}

func (ts *tcpServer) Wait() error {
    fmt.Printf("TCP server waiting...\n")
    time.Sleep(100 * time.Millisecond)
    return nil
}

// SMELL 3: Factory function returns interface instead of concrete type
func NewServer(port int) Server {
    return &tcpServer{port: port}
}

// SMELL 4: Interface mirrors entire API - no real decoupling
func demonstratePollution() {
    fmt.Println("=== Interface Pollution Example (BAD) ===")

    // User gets interface, not concrete type
    server := NewServer(8080)

    // All interactions through interface
    server.Start()
    server.Wait()
    server.Stop()

    // Problems:
    // 1. Only one implementation exists
    // 2. User doesn't provide implementation details
    // 3. Interface doesn't decouple anything
    // 4. Unnecessary indirection and potential allocation
    // 5. Interface name describes a "thing" not behavior
}

// GOOD EXAMPLE: Proper design without pollution

// Exported concrete type - users work directly with it
type TCPServer struct {
    Port     int
    IsActive bool
    name     string
}

// Factory returns concrete type
func NewTCPServer(port int, name string) *TCPServer {
    return &TCPServer{
        Port: port,
        name: name,
    }
}

func (ts *TCPServer) Start() error {
    ts.IsActive = true
    fmt.Printf("TCP server '%s' starting on port %d\n", ts.name, ts.Port)
    return nil
}

func (ts *TCPServer) Stop() error {
    ts.IsActive = false
    fmt.Printf("TCP server '%s' stopping\n", ts.name)
    return nil
}

func (ts *TCPServer) Wait() error {
    fmt.Printf("TCP server '%s' waiting...\n", ts.name)
    time.Sleep(100 * time.Millisecond)
    return nil
}

func (ts *TCPServer) GetStats() map[string]interface{} {
    return map[string]interface{}{
        "port":      ts.Port,
        "is_active": ts.IsActive,
        "name":      ts.name,
    }
}

func demonstrateGoodDesign() {
    fmt.Println("\n=== Good Design Without Pollution ===")

    // User gets concrete type
    server := NewTCPServer(8080, "WebServer")

    // Direct interaction with concrete type
    server.Start()
    server.Wait()

    // Can access all functionality
    stats := server.GetStats()
    fmt.Printf("Server stats: %+v\n", stats)

    server.Stop()
}

// When interfaces ARE appropriate - multiple implementations needed

// GOOD: Behavior-focused interface when you have multiple implementations
type Notifier interface {
    Notify(message string) error
}

// Multiple concrete implementations
type EmailNotifier struct {
    smtpHost string
    from     string
}

func (en *EmailNotifier) Notify(message string) error {
    fmt.Printf("EMAIL: Sending '%s' from %s via %s\n", message, en.from, en.smtpHost)
    return nil
}

type SlackNotifier struct {
    webhook string
    channel string
}

func (sn *SlackNotifier) Notify(message string) error {
    fmt.Printf("SLACK: Posting '%s' to #%s\n", message, sn.channel)
    return nil
}

type SMSNotifier struct {
    apiKey string
    from   string
}

func (sms *SMSNotifier) Notify(message string) error {
    fmt.Printf("SMS: Sending '%s' from %s\n", message, sms.from)
    return nil
}

// Service that uses interface appropriately - multiple implementations exist
type AlertService struct {
    notifiers []Notifier
}

func NewAlertService() *AlertService {
    return &AlertService{
        notifiers: make([]Notifier, 0),
    }
}

func (as *AlertService) AddNotifier(n Notifier) {
    as.notifiers = append(as.notifiers, n)
}

func (as *AlertService) SendAlert(message string) {
    fmt.Printf("Sending alert: %s\n", message)
    for i, notifier := range as.notifiers {
        if err := notifier.Notify(message); err != nil {
            fmt.Printf("Notifier %d failed: %v\n", i, err)
        }
    }
}

func demonstrateAppropriateInterface() {
    fmt.Println("\n=== Appropriate Interface Usage ===")

    // Create multiple implementations
    email := &EmailNotifier{smtpHost: "smtp.company.com", from: "alerts@company.com"}
    slack := &SlackNotifier{webhook: "https://hooks.slack.com/...", channel: "alerts"}
    sms := &SMSNotifier{apiKey: "api-key-123", from: "+1234567890"}

    // Service uses interface for decoupling
    alertService := NewAlertService()
    alertService.AddNotifier(email)
    alertService.AddNotifier(slack)
    alertService.AddNotifier(sms)

    // Interface enables polymorphism with multiple implementations
    alertService.SendAlert("System maintenance starting in 10 minutes")
}

// Common pollution patterns to avoid

type BadAutomationInterface interface {
    // SMELL: Too many methods - entire API surface
    Initialize(config map[string]string) error
    Start() error
    Stop() error
    Pause() error
    Resume() error
    GetStatus() string
    GetMetrics() map[string]int64
    SetConfig(key, value string) error
    GetConfig(key string) string
    Reset() error
    Validate() error
    Backup() error
    Restore(backup []byte) error
    // ... and many more
}

func demonstratePollutionPatterns() {
    fmt.Println("\n=== Common Pollution Patterns to Avoid ===")

    fmt.Println("❌ Interface with too many methods (entire API)")
    fmt.Println("❌ Interface named after 'thing' not behavior")
    fmt.Println("❌ Only one implementation exists")
    fmt.Println("❌ User doesn't provide implementation details")
    fmt.Println("❌ Factory returns interface instead of concrete type")
    fmt.Println("❌ Interface exists 'because we have to use interfaces'")
    fmt.Println("❌ Interface exists only for testing/mocking")

    fmt.Println("✅… Good interface characteristics:")
    fmt.Println("✅… Small, focused on specific behavior")
    fmt.Println("✅… Multiple implementations exist or expected")
    fmt.Println("✅… User provides implementation details")
    fmt.Println("✅… Enables real decoupling")
    fmt.Println("✅… Named after behavior, not things")
}

func main() {
    fmt.Println("=== Interface Pollution and Anti-Patterns ===")

    demonstratePollution()
    demonstrateGoodDesign()
    demonstrateAppropriateInterface()
    demonstratePollutionPatterns()

    fmt.Println("\n=== Guidelines Summary ===")
    fmt.Println("1. Don't start with interfaces - start with concrete types")
    fmt.Println("2. Create interfaces only when you need decoupling")
    fmt.Println("3. Interfaces should describe behavior, not things")
    fmt.Println("4. Factory functions should return concrete types")
    fmt.Println("5. Question interfaces that exist 'just for testing'")
    fmt.Println("6. Remove interfaces that don't provide real value")
}
```

**Hands-on Exercise 3: Interface Design Guidelines and Best Practices**:

```go
// Demonstrating interface design guidelines and best practices
package main

import (
    "fmt"
    "io"
    "strings"
    "time"
)

// GUIDELINE 1: Start with concrete types, move to interfaces when needed

// Step 1: Start with concrete implementation
type FileLogger struct {
    filename string
    prefix   string
}

func NewFileLogger(filename, prefix string) *FileLogger {
    return &FileLogger{
        filename: filename,
        prefix:   prefix,
    }
}

func (fl *FileLogger) Log(message string) {
    timestamp := time.Now().Format("15:04:05")
    logEntry := fmt.Sprintf("[%s] %s %s: %s\n", timestamp, fl.prefix, "INFO", message)
    fmt.Printf("Writing to %s: %s", fl.filename, logEntry)
}

func (fl *FileLogger) LogError(message string) {
    timestamp := time.Now().Format("15:04:05")
    logEntry := fmt.Sprintf("[%s] %s %s: %s\n", timestamp, fl.prefix, "ERROR", message)
    fmt.Printf("Writing to %s: %s", fl.filename, logEntry)
}

// Step 2: When you need multiple implementations, introduce interface
type Logger interface {
    Log(message string)
    LogError(message string)
}

// Additional concrete implementation
type ConsoleLogger struct {
    prefix string
    color  bool
}

func NewConsoleLogger(prefix string, color bool) *ConsoleLogger {
    return &ConsoleLogger{
        prefix: prefix,
        color:  color,
    }
}

func (cl *ConsoleLogger) Log(message string) {
    timestamp := time.Now().Format("15:04:05")
    if cl.color {
        fmt.Printf("\033[32m[%s] %s INFO: %s\033[0m\n", timestamp, cl.prefix, message)
    } else {
        fmt.Printf("[%s] %s INFO: %s\n", timestamp, cl.prefix, message)
    }
}

func (cl *ConsoleLogger) LogError(message string) {
    timestamp := time.Now().Format("15:04:05")
    if cl.color {
        fmt.Printf("\033[31m[%s] %s ERROR: %s\033[0m\n", timestamp, cl.prefix, message)
    } else {
        fmt.Printf("[%s] %s ERROR: %s\n", timestamp, cl.prefix, message)
    }
}

// GUIDELINE 2: Keep interfaces small and focused

// GOOD: Small, focused interfaces
type Reader interface {
    Read(p []byte) (n int, err error)
}

type Writer interface {
    Write(p []byte) (n int, err error)
}

type Closer interface {
    Close() error
}

// GOOD: Compose interfaces when needed
type ReadWriter interface {
    Reader
    Writer
}

type ReadWriteCloser interface {
    Reader
    Writer
    Closer
}

// GUIDELINE 3: Interfaces should be defined by the consumer, not the producer

// Service that needs logging capability
type AutomationService struct {
    logger Logger // Consumer defines what it needs
    name   string
}

func NewAutomationService(logger Logger, name string) *AutomationService {
    return &AutomationService{
        logger: logger,
        name:   name,
    }
}

func (as *AutomationService) ProcessData(data string) error {
    as.logger.Log(fmt.Sprintf("Service %s processing data: %s", as.name, data))

    // Simulate processing
    time.Sleep(50 * time.Millisecond)

    if data == "invalid" {
        as.logger.LogError(fmt.Sprintf("Service %s failed to process invalid data", as.name))
        return fmt.Errorf("invalid data")
    }

    as.logger.Log(fmt.Sprintf("Service %s completed processing: %s", as.name, data))
    return nil
}

// GUIDELINE 4: Use interface composition for flexibility

// Base automation behaviors
type Processor interface {
    Process(data string) error
}

type Validator interface {
    Validate(data string) error
}

type Metrics interface {
    RecordMetric(name string, value float64)
    GetMetrics() map[string]float64
}

// Composed interface for full-featured processors
type AdvancedProcessor interface {
    Processor
    Validator
    Metrics
}

// Concrete implementation
type DataProcessor struct {
    name    string
    logger  Logger
    metrics map[string]float64
}

func NewDataProcessor(name string, logger Logger) *DataProcessor {
    return &DataProcessor{
        name:    name,
        logger:  logger,
        metrics: make(map[string]float64),
    }
}

func (dp *DataProcessor) Process(data string) error {
    dp.logger.Log(fmt.Sprintf("DataProcessor %s processing: %s", dp.name, data))

    // Simulate processing time
    start := time.Now()
    time.Sleep(30 * time.Millisecond)
    duration := time.Since(start)

    dp.RecordMetric("processing_time_ms", float64(duration.Milliseconds()))
    dp.RecordMetric("processed_count", dp.metrics["processed_count"]+1)

    return nil
}

func (dp *DataProcessor) Validate(data string) error {
    if len(data) == 0 {
        return fmt.Errorf("data cannot be empty")
    }
    if len(data) > 100 {
        return fmt.Errorf("data too long")
    }
    dp.logger.Log(fmt.Sprintf("DataProcessor %s validated: %s", dp.name, data))
    return nil
}

func (dp *DataProcessor) RecordMetric(name string, value float64) {
    dp.metrics[name] = value
}

func (dp *DataProcessor) GetMetrics() map[string]float64 {
    return dp.metrics
}

// GUIDELINE 5: Design APIs that are hard to misuse

// GOOD: Clear, focused interface
type ConfigReader interface {
    ReadConfig(key string) (string, error)
}

// BAD: Ambiguous interface
type ConfigManager interface {
    Get(key string) interface{}        // What type is returned?
    Set(key string, value interface{}) // What types are accepted?
    Save() error                       // Save to where?
    Load() error                       // Load from where?
}

// GOOD: Type-safe, clear interface
type TypedConfigReader interface {
    GetString(key string) (string, error)
    GetInt(key string) (int, error)
    GetBool(key string) (bool, error)
}

// GUIDELINE 6: Use the empty interface sparingly
func demonstrateEmptyInterface() {
    fmt.Println("=== Empty Interface Usage ===")

    // GOOD: Specific interface
    var logger Logger = NewConsoleLogger("DEMO", true)
    logger.Log("This is type-safe")

    // AVOID: Empty interface unless truly needed
    var anything interface{} = "could be anything"

    // Must use type assertion to do anything useful
    if str, ok := anything.(string); ok {
        fmt.Printf("Found string: %s\n", str)
    }

    fmt.Println("Prefer specific interfaces over interface{}")
}

// GUIDELINE 7: Interfaces enable testing without mocking frameworks
type SimpleConfigReader struct {
    config map[string]string
}

func (scr *SimpleConfigReader) ReadConfig(key string) (string, error) {
    if value, exists := scr.config[key]; exists {
        return value, nil
    }
    return "", fmt.Errorf("key %s not found", key)
}

// Service that depends on config reader
type ConfigurableService struct {
    configReader ConfigReader
    name         string
}

func NewConfigurableService(configReader ConfigReader, name string) *ConfigurableService {
    return &ConfigurableService{
        configReader: configReader,
        name:         name,
    }
}

func (cs *ConfigurableService) Initialize() error {
    endpoint, err := cs.configReader.ReadConfig("endpoint")
    if err != nil {
        return fmt.Errorf("failed to read endpoint: %w", err)
    }

    fmt.Printf("Service %s initialized with endpoint: %s\n", cs.name, endpoint)
    return nil
}

func demonstrateTestableDesign() {
    fmt.Println("\n=== Testable Design with Interfaces ===")

    // Production config reader
    prodConfig := &SimpleConfigReader{
        config: map[string]string{
            "endpoint": "https://api.production.com",
            "timeout":  "30s",
        },
    }

    prodService := NewConfigurableService(prodConfig, "ProductionService")
    prodService.Initialize()

    // Test config reader (no mocking framework needed)
    testConfig := &SimpleConfigReader{
        config: map[string]string{
            "endpoint": "https://api.test.com",
            "timeout":  "5s",
        },
    }

    testService := NewConfigurableService(testConfig, "TestService")
    testService.Initialize()

    fmt.Println("Same service, different configurations - no mocking needed!")
}

func main() {
    fmt.Println("=== Interface Design Guidelines and Best Practices ===")

    fmt.Println("=== Guideline 1: Start Concrete, Move to Abstract ===")

    // Start with concrete loggers
    fileLogger := NewFileLogger("app.log", "APP")
    consoleLogger := NewConsoleLogger("APP", true)

    // Use interface when you need polymorphism
    loggers := []Logger{fileLogger, consoleLogger}

    for i, logger := range loggers {
        fmt.Printf("\nLogger %d:\n", i+1)
        logger.Log("Application started")
        logger.LogError("Sample error message")
    }

    fmt.Println("\n=== Guideline 2: Consumer-Defined Interfaces ===")

    // Service defines what it needs
    service := NewAutomationService(consoleLogger, "DataService")
    service.ProcessData("sample_data")
    service.ProcessData("invalid")

    fmt.Println("\n=== Guideline 3: Interface Composition ===")

    processor := NewDataProcessor("MainProcessor", consoleLogger)

    // Use as different interface types
    var validator Validator = processor
    var metrics Metrics = processor
    var advancedProc AdvancedProcessor = processor

    validator.Validate("test data")
    advancedProc.Process("test data")

    fmt.Printf("Metrics: %+v\n", metrics.GetMetrics())

    demonstrateEmptyInterface()
    demonstrateTestableDesign()

    fmt.Println("\n=== Design Guidelines Summary ===")
    fmt.Println("1. Start with concrete types, introduce interfaces when needed")
    fmt.Println("2. Keep interfaces small and focused")
    fmt.Println("3. Let consumers define interfaces they need")
    fmt.Println("4. Use interface composition for flexibility")
    fmt.Println("5. Design APIs that are hard to misuse")
    fmt.Println("6. Use empty interface sparingly")
    fmt.Println("7. Interfaces enable testing without complex mocking")
    fmt.Println("8. 'A good API is not just easy to use but also hard to misuse'")
}
```

**Prerequisites**: Module 21
