# Module 7: Constants and Type Safety

**Duration**: 1 hour (45 min theory + 15 min hands-on)

**Learning Objectives**:

- Master Go's sophisticated constant system and its unique characteristics
- Understand the critical difference between typed and untyped constants
- Learn how constants provide implicit conversion capabilities
- Master iota for generating constant sequences and enumerations
- Understand constant expressions and compile-time evaluation
- Apply constants effectively for type safety in automation systems
- Learn best practices for configuration and enumeration constants

**Videos Covered**:

- 2.9 Constants (0:15:29)

**Key Concepts**:

**Constants vs Variables**:

- Constants exist only at compile time, not at runtime
- Constants have no memory address - they're replaced by their values
- Constants can be typed or untyped
- Constants enable implicit conversions that variables cannot
- Constant expressions are evaluated at compile time

**Untyped Constants**:

- Untyped constants have a "kind" but no specific type
- They can implicitly convert to compatible types
- Provide flexibility in numeric operations and assignments
- Examples: `const x = 42` (untyped integer), `const pi = 3.14159` (untyped float)

**Typed Constants**:

- Explicitly typed constants behave like variables for type checking
- No implicit conversion - explicit conversion required
- Examples: `const x int = 42`, `const pi float64 = 3.14159`

**Constant Arithmetic and Precision**:

- Constant arithmetic is performed with arbitrary precision
- Results are truncated/rounded only when assigned to variables
- Enables precise calculations at compile time
- Integer constants can be arbitrarily large during compilation

**iota - The Constant Generator**:

- iota is a predeclared identifier that generates sequential constants
- Resets to 0 at each `const` declaration
- Increments by 1 for each constant in the same declaration block
- Enables powerful enumeration and bit flag patterns
- Can be used in expressions for complex constant generation

**Constant Expressions**:

- Only certain operations are allowed in constant expressions
- Arithmetic, comparison, logical, and some built-in functions
- All operands must be constants
- Evaluated at compile time for maximum efficiency

**Hands-on Exercise 1: Typed vs Untyped Constants**:

```go
// Comprehensive demonstration of Go's constant system
package main

import (
    "fmt"
    "math"
)

func main() {
    demonstrateUntypedConstants()
    demonstrateTypedConstants()
    demonstrateConstantArithmetic()
    demonstrateImplicitConversion()
}

func demonstrateUntypedConstants() {
    fmt.Println("=== Untyped Constants ===")

    // Untyped constants - have kind but no specific type
    const untypedInt = 42
    const untypedFloat = 3.14159
    const untypedString = "Hello"
    const untypedBool = true

    // These can be assigned to compatible types
    var i8 int8 = untypedInt
    var i16 int16 = untypedInt
    var i32 int32 = untypedInt
    var i64 int64 = untypedInt

    var f32 float32 = untypedFloat
    var f64 float64 = untypedFloat

    fmt.Printf("Untyped int assigned to different types:\n")
    fmt.Printf("  int8: %d, int16: %d, int32: %d, int64: %d\n", i8, i16, i32, i64)
    fmt.Printf("Untyped float assigned to different types:\n")
    fmt.Printf("  float32: %f, float64: %f\n", f32, f64)

    // Untyped constants can participate in expressions with different types
    var result1 = untypedInt + i64  // untypedInt becomes int64
    var result2 = untypedFloat + f32 // untypedFloat becomes float32

    fmt.Printf("Mixed expressions: %d, %f\n", result1, result2)
    fmt.Println()
}

func demonstrateTypedConstants() {
    fmt.Println("=== Typed Constants ===")

    // Typed constants - have specific types
    const typedInt int = 42
    const typedFloat float64 = 3.14159
    const typedString string = "Hello"

    // These behave like variables for type checking
    var i64 int64
    var f32 float32

    // This would cause a compile error:
    // i64 = typedInt  // Cannot use typedInt (type int) as type int64

    // Explicit conversion required
    i64 = int64(typedInt)
    f32 = float32(typedFloat)

    fmt.Printf("Typed constants require explicit conversion:\n")
    fmt.Printf("  int64: %d, float32: %f\n", i64, f32)
    fmt.Println()
}

func demonstrateConstantArithmetic() {
    fmt.Println("=== Constant Arithmetic and Precision ===")

    // Constants have arbitrary precision during compilation
    const huge = 1000000000000000000000000000000000000000000
    const precise = 1.23456789012345678901234567890123456789

    // Arithmetic is performed with full precision
    const calculated = huge * 2
    const division = precise / 3

    fmt.Printf("Huge constant: %e\n", float64(huge))
    fmt.Printf("Precise constant: %.30f\n", precise)
    fmt.Printf("Calculated: %e\n", float64(calculated))
    fmt.Printf("Division result: %.30f\n", division)

    // Demonstrate compile-time evaluation
    const compiletime = 2 * 3 * 4 * 5 * 6 * 7 * 8 * 9 * 10
    fmt.Printf("Compile-time calculation: %d\n", compiletime)

    // Complex constant expressions
    const complex = (1 << 10) + (1 << 20) + (1 << 30)
    fmt.Printf("Complex expression: %d\n", complex)
    fmt.Println()
}

func demonstrateImplicitConversion() {
    fmt.Println("=== Implicit Conversion with Constants ===")

    const factor = 2.5

    // Untyped constant can be used with different numeric types
    var i int = 10
    var f float64 = 20.0

    // These work because factor is untyped
    result1 := float64(i) * factor  // factor becomes float64
    result2 := f * factor           // factor becomes float64

    fmt.Printf("Implicit conversion results: %f, %f\n", result1, result2)

    // Demonstrate with function calls
    processInt(42)      // untyped constant
    processFloat(42)    // same untyped constant, different type
    processFloat(42.0)  // untyped float constant

    fmt.Println()
}

func processInt(x int) {
    fmt.Printf("Processing int: %d\n", x)
}

func processFloat(x float64) {
    fmt.Printf("Processing float: %f\n", x)
}
```

**Hands-on Exercise 2: iota and Enumeration Patterns**:

```go
// Advanced iota patterns for automation systems
package main

import "fmt"

// Basic enumeration using iota
type TaskStatus int

const (
    TaskPending TaskStatus = iota  // 0
    TaskRunning                    // 1
    TaskCompleted                  // 2
    TaskFailed                     // 3
    TaskCancelled                  // 4
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

// Skip values with iota
type Priority int

const (
    Low Priority = iota * 10  // 0
    Medium                    // 10
    High                      // 20
    Critical                  // 30
)

// Bit flags using iota
type Permission int

const (
    Read Permission = 1 << iota  // 1 (binary: 001)
    Write                        // 2 (binary: 010)
    Execute                      // 4 (binary: 100)
)

// Complex iota expressions
type Size int

const (
    _        = iota             // ignore first value
    KB Size  = 1 << (10 * iota) // 1024
    MB                          // 1048576
    GB                          // 1073741824
    TB                          // 1099511627776
)

// Multiple constant blocks reset iota
const (
    First = iota   // 0
    Second         // 1
)

const (
    Third = iota   // 0 (reset)
    Fourth         // 1
)

// Configuration constants for automation
const (
    DefaultTimeout     = 30  // seconds
    MaxRetries        = 3
    DefaultBufferSize = 1024
    MaxWorkers        = 100
)

// String constants for configuration
const (
    ConfigFile     = "automation.yaml"
    LogFile        = "automation.log"
    DefaultLogLevel = "INFO"
)

func main() {
    demonstrateBasicIota()
    demonstrateAdvancedIota()
    demonstrateBitFlags()
    demonstrateConfigConstants()
}

func demonstrateBasicIota() {
    fmt.Println("=== Basic iota Enumeration ===")

    statuses := []TaskStatus{TaskPending, TaskRunning, TaskCompleted, TaskFailed, TaskCancelled}

    for _, status := range statuses {
        fmt.Printf("Status %d: %s\n", int(status), status.String())
    }

    priorities := []Priority{Low, Medium, High, Critical}
    for _, priority := range priorities {
        fmt.Printf("Priority value: %d\n", int(priority))
    }

    fmt.Println()
}

func demonstrateAdvancedIota() {
    fmt.Println("=== Advanced iota Patterns ===")

    sizes := []Size{KB, MB, GB, TB}
    sizeNames := []string{"KB", "MB", "GB", "TB"}

    for i, size := range sizes {
        fmt.Printf("1 %s = %d bytes\n", sizeNames[i], int(size))
    }

    fmt.Printf("First: %d, Second: %d\n", First, Second)
    fmt.Printf("Third: %d, Fourth: %d\n", Third, Fourth)

    fmt.Println()
}

func demonstrateBitFlags() {
    fmt.Println("=== Bit Flags with iota ===")

    fmt.Printf("Read: %d (binary: %08b)\n", Read, Read)
    fmt.Printf("Write: %d (binary: %08b)\n", Write, Write)
    fmt.Printf("Execute: %d (binary: %08b)\n", Execute, Execute)

    // Combine permissions
    readWrite := Read | Write
    fullAccess := Read | Write | Execute

    fmt.Printf("Read+Write: %d (binary: %08b)\n", readWrite, readWrite)
    fmt.Printf("Full access: %d (binary: %08b)\n", fullAccess, fullAccess)

    // Check permissions
    fmt.Printf("Has read permission: %t\n", hasPermission(fullAccess, Read))
    fmt.Printf("Has write permission: %t\n", hasPermission(readWrite, Write))
    fmt.Printf("Has execute permission: %t\n", hasPermission(readWrite, Execute))

    fmt.Println()
}

func hasPermission(permissions, check Permission) bool {
    return permissions&check != 0
}

func demonstrateConfigConstants() {
    fmt.Println("=== Configuration Constants ===")

    fmt.Printf("Default timeout: %d seconds\n", DefaultTimeout)
    fmt.Printf("Max retries: %d\n", MaxRetries)
    fmt.Printf("Buffer size: %d bytes\n", DefaultBufferSize)
    fmt.Printf("Max workers: %d\n", MaxWorkers)

    fmt.Printf("Config file: %s\n", ConfigFile)
    fmt.Printf("Log file: %s\n", LogFile)
    fmt.Printf("Log level: %s\n", DefaultLogLevel)

    // Constants in expressions
    totalBufferSize := DefaultBufferSize * MaxWorkers
    fmt.Printf("Total buffer size: %d bytes\n", totalBufferSize)
}
```

**Hands-on Exercise 3: Automation System Configuration with Constants**:

```go
// Real-world automation system using constants effectively
package main

import (
    "fmt"
    "time"
)

// System-wide configuration constants
const (
    // Application metadata
    AppName    = "AutomationEngine"
    AppVersion = "2.1.0"

    // Timeouts and intervals (using time.Duration for type safety)
    DefaultTimeout    = 30 * time.Second
    HeartbeatInterval = 5 * time.Second
    RetryDelay        = 2 * time.Second

    // Limits and thresholds
    MaxConcurrentTasks = 50
    MaxRetryAttempts   = 3
    MaxLogFileSize     = 100 * 1024 * 1024 // 100MB

    // Network configuration
    DefaultPort     = 8080
    MaxConnections  = 1000
    ReadBufferSize  = 4096
    WriteBufferSize = 4096
)

// Environment types
type Environment int

const (
    Development Environment = iota
    Testing
    Staging
    Production
)

func (e Environment) String() string {
    switch e {
    case Development:
        return "development"
    case Testing:
        return "testing"
    case Staging:
        return "staging"
    case Production:
        return "production"
    default:
        return "unknown"
    }
}

// Log levels with explicit values
type LogLevel int

const (
    LogTrace LogLevel = iota
    LogDebug
    LogInfo
    LogWarn
    LogError
    LogFatal
)

func (ll LogLevel) String() string {
    levels := []string{"TRACE", "DEBUG", "INFO", "WARN", "ERROR", "FATAL"}
    if int(ll) < len(levels) {
        return levels[ll]
    }
    return "UNKNOWN"
}

// Task types using string constants for clarity
const (
    TaskTypeDataSync     = "data_sync"
    TaskTypeFileProcess  = "file_process"
    TaskTypeNotification = "notification"
    TaskTypeCleanup      = "cleanup"
    TaskTypeBackup       = "backup"
)

// Error codes using iota with gaps for future expansion
type ErrorCode int

const (
    ErrSuccess ErrorCode = 0

    // Client errors (1000-1999)
    ErrInvalidInput ErrorCode = 1000 + iota
    ErrMissingParameter
    ErrInvalidFormat
    ErrAuthenticationFailed

    // Server errors (2000-2999)
    ErrInternalServer ErrorCode = 2000 + iota
    ErrDatabaseConnection
    ErrServiceUnavailable
    ErrTimeout

    // System errors (3000-3999)
    ErrSystemOverload ErrorCode = 3000 + iota
    ErrDiskFull
    ErrMemoryExhausted
    ErrNetworkFailure
)

func (ec ErrorCode) String() string {
    switch {
    case ec == ErrSuccess:
        return "Success"
    case ec >= 1000 && ec < 2000:
        return "Client Error"
    case ec >= 2000 && ec < 3000:
        return "Server Error"
    case ec >= 3000 && ec < 4000:
        return "System Error"
    default:
        return "Unknown Error"
    }
}

// Configuration structure using constants
type Config struct {
    Environment     Environment
    LogLevel        LogLevel
    Port            int
    MaxConnections  int
    Timeout         time.Duration
    RetryAttempts   int
    BufferSize      int
}

// Factory function using constants for defaults
func NewConfig(env Environment) Config {
    config := Config{
        Environment:    env,
        Port:          DefaultPort,
        MaxConnections: MaxConnections,
        Timeout:       DefaultTimeout,
        RetryAttempts: MaxRetryAttempts,
        BufferSize:    ReadBufferSize,
    }

    // Environment-specific overrides
    switch env {
    case Development:
        config.LogLevel = LogDebug
        config.MaxConnections = 10
    case Testing:
        config.LogLevel = LogInfo
        config.MaxConnections = 50
        config.Timeout = 10 * time.Second
    case Staging:
        config.LogLevel = LogInfo
        config.MaxConnections = 500
    case Production:
        config.LogLevel = LogWarn
        config.Timeout = 60 * time.Second
    }

    return config
}

// Task structure using constant types
type Task struct {
    ID       string
    Type     string
    Status   TaskStatus
    Priority Priority
    Created  time.Time
    Config   Config
}

func main() {
    demonstrateConfigurationSystem()
    demonstrateErrorHandling()
    demonstrateTaskManagement()
}

func demonstrateConfigurationSystem() {
    fmt.Println("=== Configuration System with Constants ===")

    environments := []Environment{Development, Testing, Staging, Production}

    for _, env := range environments {
        config := NewConfig(env)
        fmt.Printf("\n%s Configuration:\n", env.String())
        fmt.Printf("  Log Level: %s\n", config.LogLevel.String())
        fmt.Printf("  Port: %d\n", config.Port)
        fmt.Printf("  Max Connections: %d\n", config.MaxConnections)
        fmt.Printf("  Timeout: %v\n", config.Timeout)
        fmt.Printf("  Retry Attempts: %d\n", config.RetryAttempts)
        fmt.Printf("  Buffer Size: %d bytes\n", config.BufferSize)
    }

    fmt.Printf("\nApplication: %s v%s\n", AppName, AppVersion)
    fmt.Printf("Heartbeat Interval: %v\n", HeartbeatInterval)
    fmt.Printf("Max Log File Size: %d MB\n", MaxLogFileSize/(1024*1024))
}

func demonstrateErrorHandling() {
    fmt.Println("\n=== Error Handling with Constants ===")

    errors := []ErrorCode{
        ErrSuccess,
        ErrInvalidInput,
        ErrMissingParameter,
        ErrInternalServer,
        ErrDatabaseConnection,
        ErrSystemOverload,
        ErrDiskFull,
    }

    for _, err := range errors {
        fmt.Printf("Error %d: %s (Category: %s)\n",
            int(err), getErrorMessage(err), err.String())
    }
}

func getErrorMessage(code ErrorCode) string {
    switch code {
    case ErrSuccess:
        return "Operation completed successfully"
    case ErrInvalidInput:
        return "Invalid input provided"
    case ErrMissingParameter:
        return "Required parameter missing"
    case ErrInternalServer:
        return "Internal server error"
    case ErrDatabaseConnection:
        return "Database connection failed"
    case ErrSystemOverload:
        return "System is overloaded"
    case ErrDiskFull:
        return "Disk space exhausted"
    default:
        return "Unknown error occurred"
    }
}

func demonstrateTaskManagement() {
    fmt.Println("\n=== Task Management with Constants ===")

    config := NewConfig(Production)

    tasks := []Task{
        {
            ID:       "task-001",
            Type:     TaskTypeDataSync,
            Status:   TaskRunning,
            Priority: High,
            Created:  time.Now(),
            Config:   config,
        },
        {
            ID:       "task-002",
            Type:     TaskTypeFileProcess,
            Status:   TaskPending,
            Priority: Medium,
            Created:  time.Now(),
            Config:   config,
        },
        {
            ID:       "task-003",
            Type:     TaskTypeBackup,
            Status:   TaskCompleted,
            Priority: Low,
            Created:  time.Now().Add(-time.Hour),
            Config:   config,
        },
    }

    for _, task := range tasks {
        fmt.Printf("\nTask %s:\n", task.ID)
        fmt.Printf("  Type: %s\n", task.Type)
        fmt.Printf("  Status: %s\n", task.Status.String())
        fmt.Printf("  Priority: %d\n", int(task.Priority))
        fmt.Printf("  Created: %s\n", task.Created.Format(time.RFC3339))
        fmt.Printf("  Environment: %s\n", task.Config.Environment.String())
    }

    // Demonstrate constant-based logic
    fmt.Println("\n=== Task Processing Logic ===")
    for _, task := range tasks {
        processTask(task)
    }
}

func processTask(task Task) {
    fmt.Printf("Processing task %s (%s)...\n", task.ID, task.Type)

    // Use constants for decision making
    switch task.Type {
    case TaskTypeDataSync:
        fmt.Printf("  Syncing data with timeout %v\n", task.Config.Timeout)
    case TaskTypeFileProcess:
        fmt.Printf("  Processing files with buffer size %d\n", task.Config.BufferSize)
    case TaskTypeBackup:
        fmt.Printf("  Creating backup with %d retry attempts\n", task.Config.RetryAttempts)
    case TaskTypeNotification:
        fmt.Printf("  Sending notifications\n")
    case TaskTypeCleanup:
        fmt.Printf("  Cleaning up temporary files\n")
    default:
        fmt.Printf("  Unknown task type: %s\n", task.Type)
    }
}
```

**Key Benefits of Using Constants**:

1. **Type Safety**: Typed constants prevent accidental misuse
2. **Compile-time Optimization**: Constants are replaced by values at compile time
3. **Code Clarity**: Named constants make code self-documenting
4. **Maintainability**: Centralized configuration values
5. **Performance**: No runtime overhead for constant values
6. **Flexibility**: Untyped constants provide implicit conversion capabilities

**Best Practices**:

1. Use untyped constants for flexibility when possible
2. Group related constants together
3. Use iota for sequential values and enumerations
4. Prefer typed constants for domain-specific values
5. Use descriptive names that indicate purpose and units
6. Consider using custom types for better type safety
7. Document complex constant expressions

**Prerequisites**: Module 6

```go
// Configuration constants for automation system
package main

import "fmt"

// Service status constants using iota
type ServiceStatus int

const (
    StatusUnknown ServiceStatus = iota
    StatusStarting
    StatusRunning
    StatusStopping
    StatusStopped
    StatusError
)

// Configuration constants
const (
    DefaultTimeout = 30 // untyped constant
    MaxRetries     = 3
    BufferSize     = 1024
)

// String representation for status
func (s ServiceStatus) String() string {
    switch s {
    case StatusUnknown:
        return "Unknown"
    case StatusStarting:
        return "Starting"
    case StatusRunning:
        return "Running"
    case StatusStopping:
        return "Stopping"
    case StatusStopped:
        return "Stopped"
    case StatusError:
        return "Error"
    default:
        return "Invalid"
    }
}

type AutomationService struct {
    Name   string
    Status ServiceStatus
}

func main() {
    services := []AutomationService{
        {"Database Monitor", StatusRunning},
        {"Log Processor", StatusStarting},
        {"Alert Manager", StatusError},
    }

    fmt.Println("Service Status Report:")
    for _, service := range services {
        fmt.Printf("- %s: %s (%d)\n", service.Name, service.Status, service.Status)
    }

    // Demonstrate constant usage
    fmt.Printf("\nConfiguration:\n")
    fmt.Printf("- Default Timeout: %d seconds\n", DefaultTimeout)
    fmt.Printf("- Max Retries: %d\n", MaxRetries)
    fmt.Printf("- Buffer Size: %d bytes\n", BufferSize)
}
```

**Prerequisites**: Module 6
