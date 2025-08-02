# Module 24: Error Handling: Values, Variables, and Context

**Duration**: 1 hour (45 min theory + 15 min hands-on)

**Learning Objectives**:

- Master Go's error handling philosophy and patterns
- Understand error values, variables, and custom error types
- Learn to provide context through error types and behavior
- Apply robust error handling in automation systems
- Design APIs that respect error handling principles

**Videos Covered**:

- 6.1 Topics (0:00:51)
- 6.2 Default Error Values (0:11:33)
- 6.3 Error Variables (0:02:40)
- 6.4 Type as Context (0:07:04)
- 6.5 Behavior as Context (0:09:50)

**Key Concepts**:

- Error interface: simple yet powerful design
- Error values vs error variables vs error types
- Providing context through custom error types
- Error behavior: temporary errors, timeout errors
- API design: giving users enough context for decisions
- Error handling as part of the happy path

**Hands-on Exercise 1: Default Error Values and Error Variables**:

```go
// Demonstrating basic error handling patterns in automation systems
package main

import (
    "fmt"
    "time"
)

// PART 1: Default Error Values - simple string errors using fmt.Errorf

// AutomationTask represents a task in our automation system
type AutomationTask struct {
    ID          string
    Type        string
    Status      string
    CreatedAt   time.Time
    CompletedAt *time.Time
}

// Simple error handling with fmt.Errorf - most common pattern
func validateTaskID(taskID string) error {
    if taskID == "" {
        return fmt.Errorf("task ID cannot be empty")
    }

    if len(taskID) < 3 {
        return fmt.Errorf("task ID '%s' is too short, minimum 3 characters required", taskID)
    }

    if len(taskID) > 50 {
        return fmt.Errorf("task ID '%s' is too long, maximum 50 characters allowed", taskID)
    }

    // Check for invalid characters (simplified)
    for _, char := range taskID {
        if char == ' ' || char == '\t' || char == '\n' {
            return fmt.Errorf("task ID '%s' contains invalid whitespace characters", taskID)
        }
    }

    return nil
}

func validateTaskType(taskType string) error {
    validTypes := []string{"data-processing", "file-transfer", "notification", "backup", "monitoring"}

    if taskType == "" {
        return fmt.Errorf("task type cannot be empty")
    }

    for _, valid := range validTypes {
        if taskType == valid {
            return nil
        }
    }

    return fmt.Errorf("task type '%s' is not supported, valid types are: %v", taskType, validTypes)
}

func createTask(id, taskType string) (*AutomationTask, error) {
    // Validate inputs using simple error values
    if err := validateTaskID(id); err != nil {
        return nil, fmt.Errorf("task creation failed: %w", err)
    }

    if err := validateTaskType(taskType); err != nil {
        return nil, fmt.Errorf("task creation failed: %w", err)
    }

    task := &AutomationTask{
        ID:        id,
        Type:      taskType,
        Status:    "created",
        CreatedAt: time.Now(),
    }

    fmt.Printf("Task created successfully: %s (%s)\n", task.ID, task.Type)
    return task, nil
}

// PART 2: Error Variables - predefined errors for comparison and decision making

// Error variables - notice the "Err" prefix convention
var (
    ErrTaskNotFound      = fmt.Errorf("task not found")
    ErrTaskAlreadyExists = fmt.Errorf("task already exists")
    ErrTaskInvalidState  = fmt.Errorf("task is in invalid state")
    ErrServiceBusy       = fmt.Errorf("automation service is busy")
    ErrServiceOffline    = fmt.Errorf("automation service is offline")
    ErrInsufficientResources = fmt.Errorf("insufficient system resources")
    ErrPermissionDenied  = fmt.Errorf("permission denied")
)

// TaskRegistry simulates a task storage system
type TaskRegistry struct {
    tasks   map[string]*AutomationTask
    maxTasks int
    offline  bool
}

func NewTaskRegistry(maxTasks int) *TaskRegistry {
    return &TaskRegistry{
        tasks:    make(map[string]*AutomationTask),
        maxTasks: maxTasks,
        offline:  false,
    }
}

func (tr *TaskRegistry) SetOffline(offline bool) {
    tr.offline = offline
}

func (tr *TaskRegistry) RegisterTask(task *AutomationTask) error {
    if tr.offline {
        return ErrServiceOffline
    }

    if len(tr.tasks) >= tr.maxTasks {
        return ErrInsufficientResources
    }

    if _, exists := tr.tasks[task.ID]; exists {
        return ErrTaskAlreadyExists
    }

    tr.tasks[task.ID] = task
    fmt.Printf("Task registered: %s\n", task.ID)
    return nil
}

func (tr *TaskRegistry) FindTask(taskID string) (*AutomationTask, error) {
    if tr.offline {
        return nil, ErrServiceOffline
    }

    task, exists := tr.tasks[taskID]
    if !exists {
        return nil, ErrTaskNotFound
    }

    return task, nil
}

func (tr *TaskRegistry) UpdateTaskStatus(taskID, newStatus string) error {
    if tr.offline {
        return ErrServiceOffline
    }

    task, exists := tr.tasks[taskID]
    if !exists {
        return ErrTaskNotFound
    }

    // State validation
    validTransitions := map[string][]string{
        "created":   {"running", "cancelled"},
        "running":   {"completed", "failed", "cancelled"},
        "completed": {},
        "failed":    {"running"}, // Allow retry
        "cancelled": {},
    }

    validNext, hasTransitions := validTransitions[task.Status]
    if !hasTransitions {
        return ErrTaskInvalidState
    }

    for _, valid := range validNext {
        if newStatus == valid {
            task.Status = newStatus
            if newStatus == "completed" || newStatus == "failed" {
                now := time.Now()
                task.CompletedAt = &now
            }
            fmt.Printf("Task %s status updated: %s -> %s\n", taskID, task.Status, newStatus)
            return nil
        }
    }

    return ErrTaskInvalidState
}

// Error handling function that demonstrates decision making based on error variables
func handleTaskError(operation string, err error) {
    if err == nil {
        return
    }

    fmt.Printf("Operation '%s' failed: %v\n", operation, err)

    // Decision making based on specific error variables
    switch err {
    case ErrTaskNotFound:
        fmt.Println("  →’ Recovery: Task not found, will create new task")

    case ErrTaskAlreadyExists:
        fmt.Println("  →’ Recovery: Task exists, will update existing task")

    case ErrTaskInvalidState:
        fmt.Println("  →’ Recovery: Invalid state transition, will check current state")

    case ErrServiceBusy:
        fmt.Println("  →’ Recovery: Service busy, will retry after delay")

    case ErrServiceOffline:
        fmt.Println("  →’ Recovery: Service offline, will switch to backup service")

    case ErrInsufficientResources:
        fmt.Println("  →’ Recovery: Insufficient resources, will clean up old tasks")

    case ErrPermissionDenied:
        fmt.Println("  →’ Recovery: Permission denied, will escalate to admin")

    default:
        fmt.Println("  →’ Recovery: Unknown error, will log and continue")
    }
}

func main() {
    fmt.Println("=== Default Error Values and Error Variables ===")

    fmt.Println("\n=== Part 1: Default Error Values ===")

    // Test simple error values with validation
    testTaskData := []struct {
        id       string
        taskType string
    }{
        {"", "data-processing"},                    // Empty ID
        {"ab", "data-processing"},                  // Too short
        {"valid-task-123", "invalid-type"},         // Invalid type
        {"task with spaces", "data-processing"},    // Invalid characters
        {"valid-task-456", "notification"},         // Valid
    }

    for _, data := range testTaskData {
        fmt.Printf("\nCreating task: ID='%s', Type='%s'\n", data.id, data.taskType)
        task, err := createTask(data.id, data.taskType)
        if err != nil {
            fmt.Printf("  Error: %v\n", err)
        } else {
            fmt.Printf("  Success: Created task %s\n", task.ID)
        }
    }

    fmt.Println("\n=== Part 2: Error Variables ===")

    // Create task registry
    registry := NewTaskRegistry(2) // Small limit to test resource errors

    // Create some valid tasks
    task1, _ := createTask("task-001", "data-processing")
    task2, _ := createTask("task-002", "file-transfer")
    task3, _ := createTask("task-003", "backup")

    // Test various error scenarios
    fmt.Println("\nTesting task registration:")

    // Normal registration
    err := registry.RegisterTask(task1)
    handleTaskError("register task1", err)

    err = registry.RegisterTask(task2)
    handleTaskError("register task2", err)

    // Resource limit exceeded
    err = registry.RegisterTask(task3)
    handleTaskError("register task3", err)

    // Duplicate registration
    err = registry.RegisterTask(task1)
    handleTaskError("register duplicate task1", err)

    fmt.Println("\nTesting task lookup:")

    // Find existing task
    foundTask, err := registry.FindTask("task-001")
    if err != nil {
        handleTaskError("find task-001", err)
    } else {
        fmt.Printf("Found task: %s (%s)\n", foundTask.ID, foundTask.Status)
    }

    // Find non-existent task
    _, err = registry.FindTask("non-existent")
    handleTaskError("find non-existent", err)

    fmt.Println("\nTesting status updates:")

    // Valid state transitions
    err = registry.UpdateTaskStatus("task-001", "running")
    handleTaskError("start task-001", err)

    err = registry.UpdateTaskStatus("task-001", "completed")
    handleTaskError("complete task-001", err)

    // Invalid state transition
    err = registry.UpdateTaskStatus("task-001", "running")
    handleTaskError("restart completed task-001", err)

    fmt.Println("\nTesting offline service:")

    // Set service offline
    registry.SetOffline(true)

    _, err = registry.FindTask("task-002")
    handleTaskError("find task while offline", err)

    err = registry.UpdateTaskStatus("task-002", "running")
    handleTaskError("update task while offline", err)

    fmt.Println("\n=== Key Principles ===")
    fmt.Println("1. Use fmt.Errorf for simple, descriptive error messages")
    fmt.Println("2. Error variables enable specific error identification and recovery")
    fmt.Println("3. Error variables should be declared at package level with 'Err' prefix")
    fmt.Println("4. Use == comparison to check for specific error variables")
    fmt.Println("5. Error handling should provide context for informed decisions")
    fmt.Println("6. Design APIs to give users enough context to recover or fail gracefully")
}
```

**Hands-on Exercise 2: Type as Context with Custom Error Types**:

```go
// Demonstrating custom error types for rich context in automation systems
package main

import (
    "fmt"
    "reflect"
    "strconv"
    "time"
)

// PART 1: Custom Error Types - when error variables aren't enough

// ValidationError provides detailed context about validation failures
type ValidationError struct {
    Field     string      // Which field failed validation
    Value     interface{} // The actual value that failed
    Rule      string      // Which validation rule was violated
    Message   string      // Human-readable description
    Timestamp time.Time   // When the error occurred
}

// Error implements the error interface - notice pointer semantics
func (ve *ValidationError) Error() string {
    // Include ALL fields in the error message for logging
    return fmt.Sprintf("validation error [%s]: field '%s' with value '%v' violated rule '%s' - %s",
        ve.Timestamp.Format("15:04:05"), ve.Field, ve.Value, ve.Rule, ve.Message)
}

// NetworkError provides context about network-related failures
type NetworkError struct {
    Operation string        // What operation was being performed
    Endpoint  string        // Which endpoint was involved
    StatusCode int          // HTTP status code (if applicable)
    Latency   time.Duration // How long the operation took
    Cause     error         // Underlying error
    Timestamp time.Time     // When the error occurred
    Retryable bool          // Whether this error can be retried
}

func (ne *NetworkError) Error() string {
    return fmt.Sprintf("network error [%s]: %s to '%s' failed with status %d after %v - %s (retryable: %t)",
        ne.Timestamp.Format("15:04:05"), ne.Operation, ne.Endpoint, ne.StatusCode,
        ne.Latency, ne.Cause.Error(), ne.Retryable)
}

// Unwrap allows error unwrapping (Go 1.13+)
func (ne *NetworkError) Unwrap() error {
    return ne.Cause
}

// ConfigurationError provides context about configuration issues
type ConfigurationError struct {
    ConfigFile string                 // Which config file
    Section    string                 // Which section of config
    Key        string                 // Which configuration key
    Expected   string                 // What was expected
    Actual     interface{}            // What was actually found
    Suggestions []string              // Possible fixes
}

func (ce *ConfigurationError) Error() string {
    suggestions := ""
    if len(ce.Suggestions) > 0 {
        suggestions = fmt.Sprintf(" (suggestions: %v)", ce.Suggestions)
    }
    return fmt.Sprintf("configuration error in %s[%s.%s]: expected %s, got '%v'%s",
        ce.ConfigFile, ce.Section, ce.Key, ce.Expected, ce.Actual, suggestions)
}

// PART 2: Factory Functions for Error Creation

func NewValidationError(field string, value interface{}, rule, message string) *ValidationError {
    return &ValidationError{
        Field:     field,
        Value:     value,
        Rule:      rule,
        Message:   message,
        Timestamp: time.Now(),
    }
}

func NewNetworkError(operation, endpoint string, statusCode int, latency time.Duration, cause error) *NetworkError {
    retryable := statusCode >= 500 || statusCode == 429 || statusCode == 408
    return &NetworkError{
        Operation:  operation,
        Endpoint:   endpoint,
        StatusCode: statusCode,
        Latency:    latency,
        Cause:      cause,
        Timestamp:  time.Now(),
        Retryable:  retryable,
    }
}

func NewConfigurationError(file, section, key, expected string, actual interface{}) *ConfigurationError {
    var suggestions []string

    // Provide helpful suggestions based on the error
    switch expected {
    case "positive integer":
        suggestions = []string{"use a number > 0", "check for typos in numeric values"}
    case "valid URL":
        suggestions = []string{"ensure URL starts with http:// or https://", "check for missing protocol"}
    case "duration":
        suggestions = []string{"use format like '30s', '5m', '1h'", "check time unit suffix"}
    }

    return &ConfigurationError{
        ConfigFile:  file,
        Section:     section,
        Key:         key,
        Expected:    expected,
        Actual:      actual,
        Suggestions: suggestions,
    }
}

// PART 3: Automation Service with Custom Error Types

type AutomationConfig struct {
    ServiceName     string        `json:"service_name"`
    MaxWorkers      int           `json:"max_workers"`
    ProcessTimeout  time.Duration `json:"process_timeout"`
    APIEndpoint     string        `json:"api_endpoint"`
    RetryAttempts   int           `json:"retry_attempts"`
    EnableMetrics   bool          `json:"enable_metrics"`
}

type AutomationService struct {
    config *AutomationConfig
    name   string
}

func NewAutomationService(name string) *AutomationService {
    return &AutomationService{
        name: name,
    }
}

// ValidateConfig demonstrates validation with custom error types
func (as *AutomationService) ValidateConfig(configData map[string]interface{}) error {
    // Validate service name
    if name, ok := configData["service_name"].(string); !ok || name == "" {
        return NewValidationError("service_name", configData["service_name"],
            "required_string", "service name must be a non-empty string")
    }

    // Validate max workers
    if workers, ok := configData["max_workers"].(float64); !ok || workers <= 0 || workers > 100 {
        return NewValidationError("max_workers", configData["max_workers"],
            "positive_integer_range", "max workers must be between 1 and 100")
    }

    // Validate timeout
    if timeoutStr, ok := configData["process_timeout"].(string); !ok {
        return NewValidationError("process_timeout", configData["process_timeout"],
            "duration_string", "process timeout must be a duration string like '30s' or '5m'")
    } else {
        if _, err := time.ParseDuration(timeoutStr); err != nil {
            return NewValidationError("process_timeout", timeoutStr,
                "valid_duration", "process timeout must be a valid duration format")
        }
    }

    // Validate API endpoint
    if endpoint, ok := configData["api_endpoint"].(string); !ok || endpoint == "" {
        return NewValidationError("api_endpoint", configData["api_endpoint"],
            "required_url", "API endpoint must be a valid URL")
    }

    return nil
}

// LoadConfig demonstrates configuration errors
func (as *AutomationService) LoadConfig(configFile string, configData map[string]interface{}) error {
    // Simulate reading from different config sections

    // Check service section
    if serviceName, exists := configData["service_name"]; !exists {
        return NewConfigurationError(configFile, "service", "name", "string", nil)
    } else if name, ok := serviceName.(string); !ok || name == "" {
        return NewConfigurationError(configFile, "service", "name", "non-empty string", serviceName)
    }

    // Check worker section
    if maxWorkers, exists := configData["max_workers"]; !exists {
        return NewConfigurationError(configFile, "workers", "max_count", "positive integer", nil)
    } else if workers, ok := maxWorkers.(float64); !ok || workers <= 0 {
        return NewConfigurationError(configFile, "workers", "max_count", "positive integer", maxWorkers)
    }

    // Check timeout section
    if timeout, exists := configData["process_timeout"]; !exists {
        return NewConfigurationError(configFile, "timeouts", "process", "duration", nil)
    } else if timeoutStr, ok := timeout.(string); !ok {
        return NewConfigurationError(configFile, "timeouts", "process", "duration", timeout)
    } else if _, err := time.ParseDuration(timeoutStr); err != nil {
        return NewConfigurationError(configFile, "timeouts", "process", "valid duration", timeout)
    }

    return nil
}

// CallExternalAPI demonstrates network errors
func (as *AutomationService) CallExternalAPI(endpoint, operation string) error {
    start := time.Now()

    // Simulate different network scenarios
    switch endpoint {
    case "https://api.timeout.com":
        latency := time.Since(start) + 30*time.Second
        return NewNetworkError(operation, endpoint, 408, latency, fmt.Errorf("request timeout"))

    case "https://api.server-error.com":
        latency := time.Since(start) + 100*time.Millisecond
        return NewNetworkError(operation, endpoint, 500, latency, fmt.Errorf("internal server error"))

    case "https://api.rate-limited.com":
        latency := time.Since(start) + 50*time.Millisecond
        return NewNetworkError(operation, endpoint, 429, latency, fmt.Errorf("rate limit exceeded"))

    case "https://api.not-found.com":
        latency := time.Since(start) + 75*time.Millisecond
        return NewNetworkError(operation, endpoint, 404, latency, fmt.Errorf("endpoint not found"))

    default:
        latency := time.Since(start) + 25*time.Millisecond
        fmt.Printf("API call successful: %s %s (took %v)\n", operation, endpoint, latency)
        return nil
    }
}

// PART 4: Type as Context - Error Handling with Type Switch

func handleAutomationError(operation string, err error) {
    if err == nil {
        return
    }

    fmt.Printf("\nOperation '%s' failed: %v\n", operation, err)

    // Type as context - use type switch to handle different error types
    switch e := err.(type) {
    case *ValidationError:
        fmt.Printf("  →’ VALIDATION ERROR:\n")
        fmt.Printf("    Field: %s\n", e.Field)
        fmt.Printf("    Value: %v (type: %s)\n", e.Value, reflect.TypeOf(e.Value))
        fmt.Printf("    Rule: %s\n", e.Rule)
        fmt.Printf("    Recovery: Fix the %s field and retry\n", e.Field)

    case *NetworkError:
        fmt.Printf("  →’ NETWORK ERROR:\n")
        fmt.Printf("    Operation: %s\n", e.Operation)
        fmt.Printf("    Endpoint: %s\n", e.Endpoint)
        fmt.Printf("    Status Code: %d\n", e.StatusCode)
        fmt.Printf("    Latency: %v\n", e.Latency)
        fmt.Printf("    Retryable: %t\n", e.Retryable)

        if e.Retryable {
            fmt.Printf("    Recovery: Retry the operation after delay\n")
        } else {
            fmt.Printf("    Recovery: Check endpoint configuration\n")
        }

    case *ConfigurationError:
        fmt.Printf("  →’ CONFIGURATION ERROR:\n")
        fmt.Printf("    File: %s\n", e.ConfigFile)
        fmt.Printf("    Section: %s\n", e.Section)
        fmt.Printf("    Key: %s\n", e.Key)
        fmt.Printf("    Expected: %s\n", e.Expected)
        fmt.Printf("    Actual: %v\n", e.Actual)
        if len(e.Suggestions) > 0 {
            fmt.Printf("    Suggestions: %v\n", e.Suggestions)
        }
        fmt.Printf("    Recovery: Update configuration file and restart\n")

    default:
        fmt.Printf("  →’ UNKNOWN ERROR TYPE: %T\n", err)
        fmt.Printf("    Recovery: Log error and continue with defaults\n")
    }
}

func main() {
    fmt.Println("=== Type as Context with Custom Error Types ===")

    service := NewAutomationService("DemoService")

    fmt.Println("\n=== Part 1: Validation Errors ===")

    // Test validation with various invalid configurations
    invalidConfigs := []map[string]interface{}{
        {"service_name": "", "max_workers": 10.0, "process_timeout": "30s", "api_endpoint": "https://api.example.com"},
        {"service_name": "TestService", "max_workers": -5.0, "process_timeout": "30s", "api_endpoint": "https://api.example.com"},
        {"service_name": "TestService", "max_workers": 10.0, "process_timeout": "invalid", "api_endpoint": "https://api.example.com"},
        {"service_name": "TestService", "max_workers": 10.0, "process_timeout": "30s", "api_endpoint": ""},
    }

    for i, config := range invalidConfigs {
        fmt.Printf("\nValidating config %d:\n", i+1)
        err := service.ValidateConfig(config)
        handleAutomationError("validate_config", err)
    }

    fmt.Println("\n=== Part 2: Configuration Errors ===")

    // Test configuration loading with various issues
    configFiles := []struct {
        filename string
        data     map[string]interface{}
    }{
        {"missing_name.json", map[string]interface{}{"max_workers": 5.0, "process_timeout": "30s"}},
        {"invalid_workers.json", map[string]interface{}{"service_name": "Test", "max_workers": "not_a_number", "process_timeout": "30s"}},
        {"bad_timeout.json", map[string]interface{}{"service_name": "Test", "max_workers": 5.0, "process_timeout": 123}},
    }

    for _, configFile := range configFiles {
        fmt.Printf("\nLoading config from %s:\n", configFile.filename)
        err := service.LoadConfig(configFile.filename, configFile.data)
        handleAutomationError("load_config", err)
    }

    fmt.Println("\n=== Part 3: Network Errors ===")

    // Test network operations with various failure scenarios
    endpoints := []string{
        "https://api.timeout.com",
        "https://api.server-error.com",
        "https://api.rate-limited.com",
        "https://api.not-found.com",
        "https://api.success.com",
    }

    for _, endpoint := range endpoints {
        fmt.Printf("\nCalling API: %s\n", endpoint)
        err := service.CallExternalAPI(endpoint, "GET")
        handleAutomationError("api_call", err)
    }

    fmt.Println("\n=== Key Principles ===")
    fmt.Println("1. Custom error types provide rich context beyond simple strings")
    fmt.Println("2. Include ALL relevant fields in the Error() method for logging")
    fmt.Println("3. Use pointer semantics for error interface implementation")
    fmt.Println("4. Factory functions make error creation consistent and easy")
    fmt.Println("5. Type switch enables different handling based on error type")
    fmt.Println("6. Custom errors should help users make informed recovery decisions")
    fmt.Println("7. Be cautious: type switching reduces decoupling benefits")
}
```

**Hands-on Exercise 3: Behavior as Context with Error Interfaces**:

```go
// Demonstrating behavior-based error handling in automation systems
package main

import (
    "fmt"
    "net"
    "time"
)

// PART 1: Behavior Interfaces - Define error behaviors, not concrete types

// TemporaryError interface for errors that are temporary in nature
type TemporaryError interface {
    error
    Temporary() bool
}

// TimeoutError interface for timeout-related errors
type TimeoutError interface {
    error
    Timeout() bool
}

// RetryableError interface for errors that can be retried
type RetryableError interface {
    error
    Retryable() bool
    RetryAfter() time.Duration
}

// SeverityError interface for errors with severity levels
type SeverityError interface {
    error
    Severity() string
    IsCritical() bool
}

// RecoverableError interface for errors that suggest recovery actions
type RecoverableError interface {
    error
    CanRecover() bool
    RecoveryAction() string
}

// PART 2: Concrete Error Type that implements multiple behaviors

// AutomationError is a concrete error type that implements multiple behavior interfaces
type AutomationError struct {
    Code           string
    Message        string
    Cause          error
    Timestamp      time.Time
    isTemporary    bool
    isTimeout      bool
    retryable      bool
    retryDelay     time.Duration
    severity       string
    recoverable    bool
    recoveryAction string
}

// Error implements the basic error interface
func (ae *AutomationError) Error() string {
    if ae.Cause != nil {
        return fmt.Sprintf("[%s] %s: %v (at %s)", ae.Code, ae.Message, ae.Cause, ae.Timestamp.Format("15:04:05"))
    }
    return fmt.Sprintf("[%s] %s (at %s)", ae.Code, ae.Message, ae.Timestamp.Format("15:04:05"))
}

// Temporary implements TemporaryError interface
func (ae *AutomationError) Temporary() bool {
    return ae.isTemporary
}

// Timeout implements TimeoutError interface
func (ae *AutomationError) Timeout() bool {
    return ae.isTimeout
}

// Retryable implements RetryableError interface
func (ae *AutomationError) Retryable() bool {
    return ae.retryable
}

func (ae *AutomationError) RetryAfter() time.Duration {
    return ae.retryDelay
}

// Severity implements SeverityError interface
func (ae *AutomationError) Severity() string {
    return ae.severity
}

func (ae *AutomationError) IsCritical() bool {
    return ae.severity == "critical" || ae.severity == "fatal"
}

// CanRecover implements RecoverableError interface
func (ae *AutomationError) CanRecover() bool {
    return ae.recoverable
}

func (ae *AutomationError) RecoveryAction() string {
    return ae.recoveryAction
}

// Unwrap allows error unwrapping (Go 1.13+)
func (ae *AutomationError) Unwrap() error {
    return ae.Cause
}

// PART 3: Factory Functions for Different Error Scenarios

func NewTemporaryError(code, message string) *AutomationError {
    return &AutomationError{
        Code:           code,
        Message:        message,
        Timestamp:      time.Now(),
        isTemporary:    true,
        retryable:      true,
        retryDelay:     5 * time.Second,
        severity:       "warning",
        recoverable:    true,
        recoveryAction: "retry after delay",
    }
}

func NewTimeoutError(code, message string) *AutomationError {
    return &AutomationError{
        Code:           code,
        Message:        message,
        Timestamp:      time.Now(),
        isTimeout:      true,
        retryable:      true,
        retryDelay:     10 * time.Second,
        severity:       "error",
        recoverable:    true,
        recoveryAction: "increase timeout and retry",
    }
}

func NewCriticalError(code, message string, cause error) *AutomationError {
    return &AutomationError{
        Code:           code,
        Message:        message,
        Cause:          cause,
        Timestamp:      time.Now(),
        isTemporary:    false,
        retryable:      false,
        severity:       "critical",
        recoverable:    false,
        recoveryAction: "manual intervention required",
    }
}

func NewRecoverableError(code, message, recoveryAction string) *AutomationError {
    return &AutomationError{
        Code:           code,
        Message:        message,
        Timestamp:      time.Now(),
        retryable:      false,
        severity:       "error",
        recoverable:    true,
        recoveryAction: recoveryAction,
    }
}

// PART 4: Automation Services with Behavior-Based Error Handling

type DataProcessor struct {
    name           string
    maxRetries     int
    baseTimeout    time.Duration
    criticalErrors int
}

func NewDataProcessor(name string) *DataProcessor {
    return &DataProcessor{
        name:        name,
        maxRetries:  3,
        baseTimeout: 30 * time.Second,
    }
}

func (dp *DataProcessor) ProcessData(data string) error {
    fmt.Printf("DataProcessor %s: Processing data '%s'\n", dp.name, data)

    // Simulate different error scenarios
    switch data {
    case "timeout_data":
        return NewTimeoutError("PROC_TIMEOUT", "data processing timed out")

    case "temp_fail_data":
        return NewTemporaryError("TEMP_FAIL", "temporary processing failure")

    case "corrupt_data":
        return NewCriticalError("DATA_CORRUPT", "data corruption detected",
            fmt.Errorf("checksum mismatch"))

    case "config_missing":
        return NewRecoverableError("CONFIG_MISSING", "configuration file not found",
            "restore configuration from backup")

    case "disk_full":
        return NewRecoverableError("DISK_FULL", "insufficient disk space",
            "clean up temporary files or expand storage")

    default:
        fmt.Printf("DataProcessor %s: Successfully processed '%s'\n", dp.name, data)
        return nil
    }
}

func (dp *DataProcessor) ProcessWithRetry(data string) error {
    var lastErr error

    for attempt := 1; attempt <= dp.maxRetries; attempt++ {
        fmt.Printf("\nAttempt %d/%d for data: %s\n", attempt, dp.maxRetries, data)

        err := dp.ProcessData(data)
        if err == nil {
            return nil
        }

        lastErr = err

        // Use behavior interfaces to determine retry strategy
        if retryable, ok := err.(RetryableError); ok && retryable.Retryable() {
            if attempt < dp.maxRetries {
                delay := retryable.RetryAfter()
                fmt.Printf("  Retryable error, waiting %v before retry: %v\n", delay, err)
                time.Sleep(delay)
                continue
            }
        }

        // Non-retryable error
        fmt.Printf("  Non-retryable error: %v\n", err)
        break
    }

    return fmt.Errorf("processing failed after %d attempts: %w", dp.maxRetries, lastErr)
}

type SystemMonitor struct {
    name            string
    alertThreshold  int
    shutdownOnCritical bool
}

func NewSystemMonitor(name string) *SystemMonitor {
    return &SystemMonitor{
        name:               name,
        alertThreshold:     5,
        shutdownOnCritical: true,
    }
}

func (sm *SystemMonitor) HandleSystemError(component string, err error) {
    if err == nil {
        return
    }

    fmt.Printf("\nSystemMonitor %s: Error in component '%s'\n", sm.name, component)

    // Use behavior interfaces to determine response - this is the key insight!
    // We stay in the decoupled interface state and don't drop to concrete types

    // Check severity
    if severityErr, ok := err.(SeverityError); ok {
        severity := severityErr.Severity()
        fmt.Printf("  Severity: %s\n", severity)

        if severityErr.IsCritical() {
            fmt.Printf("  🔥š¨ CRITICAL ERROR DETECTED!\n")
            if sm.shutdownOnCritical {
                fmt.Printf("  🔥›‘ Initiating emergency shutdown...\n")
                // In real system: os.Exit(1) or panic()
            }
        }
    }

    // Check if temporary
    if tempErr, ok := err.(TemporaryError); ok && tempErr.Temporary() {
        fmt.Printf("  ³ Temporary error - monitoring for resolution\n")
    }

    // Check if timeout
    if timeoutErr, ok := err.(TimeoutError); ok && timeoutErr.Timeout() {
        fmt.Printf("  ⏰ Timeout error - adjusting timeout settings\n")
    }

    // Check recovery options
    if recoverableErr, ok := err.(RecoverableError); ok {
        if recoverableErr.CanRecover() {
            action := recoverableErr.RecoveryAction()
            fmt.Printf("  🔥”§ Recovery possible: %s\n", action)
            fmt.Printf("  🔥“‹ Scheduling recovery action...\n")
        } else {
            fmt.Printf("  ❌ No automatic recovery available\n")
        }
    }

    // Check retry options
    if retryableErr, ok := err.(RetryableError); ok && retryableErr.Retryable() {
        delay := retryableErr.RetryAfter()
        fmt.Printf("  🔥”„ Retryable error - will retry in %v\n", delay)
    }
}

// PART 5: Demonstrate Integration with Standard Library net.Error

func simulateNetworkOperation(endpoint string) error {
    fmt.Printf("Connecting to %s...\n", endpoint)

    switch endpoint {
    case "timeout.example.com":
        // Simulate a network timeout using a type that implements net.Error
        return &net.OpError{
            Op:   "dial",
            Net:  "tcp",
            Addr: &net.TCPAddr{IP: net.ParseIP("192.168.1.1"), Port: 80},
            Err:  fmt.Errorf("i/o timeout"),
        }

    case "temp-fail.example.com":
        return NewTemporaryError("NET_TEMP", "temporary network failure")

    default:
        fmt.Printf("Successfully connected to %s\n", endpoint)
        return nil
    }
}

func handleNetworkError(operation string, err error) {
    if err == nil {
        return
    }

    fmt.Printf("Network operation '%s' failed: %v\n", operation, err)

    // Check for standard library net.Error behavior
    if netErr, ok := err.(net.Error); ok {
        fmt.Printf("  Standard net.Error detected:\n")
        if netErr.Temporary() {
            fmt.Printf("    - Temporary: true\n")
        }
        if netErr.Timeout() {
            fmt.Printf("    - Timeout: true\n")
        }
    }

    // Check for our custom behavior interfaces
    if tempErr, ok := err.(TemporaryError); ok && tempErr.Temporary() {
        fmt.Printf("  Custom TemporaryError: true\n")
    }

    if timeoutErr, ok := err.(TimeoutError); ok && timeoutErr.Timeout() {
        fmt.Printf("  Custom TimeoutError: true\n")
    }
}

func main() {
    fmt.Println("=== Behavior as Context with Error Interfaces ===")

    fmt.Println("\n=== Part 1: Data Processing with Retry Logic ===")

    processor := NewDataProcessor("MainProcessor")

    testData := []string{
        "normal_data",
        "timeout_data",
        "temp_fail_data",
        "corrupt_data",
        "config_missing",
        "disk_full",
    }

    for _, data := range testData {
        fmt.Printf("\n" + "="*50 + "\n")
        err := processor.ProcessWithRetry(data)
        if err != nil {
            fmt.Printf("Final result: %v\n", err)
        }
    }

    fmt.Println("\n=== Part 2: System Monitoring with Behavior-Based Responses ===")

    monitor := NewSystemMonitor("CentralMonitor")

    // Simulate various system errors
    systemErrors := []struct {
        component string
        err       error
    }{
        {"database", NewTemporaryError("DB_CONN", "database connection lost")},
        {"storage", NewRecoverableError("DISK_FULL", "storage capacity exceeded", "expand storage or archive old data")},
        {"security", NewCriticalError("SEC_BREACH", "security breach detected", fmt.Errorf("unauthorized access attempt"))},
        {"network", NewTimeoutError("NET_TIMEOUT", "network request timed out")},
    }

    for _, sysErr := range systemErrors {
        monitor.HandleSystemError(sysErr.component, sysErr.err)
        fmt.Println()
    }

    fmt.Println("\n=== Part 3: Integration with Standard Library ===")

    endpoints := []string{
        "timeout.example.com",
        "temp-fail.example.com",
        "success.example.com",
    }

    for _, endpoint := range endpoints {
        fmt.Printf("\nTesting endpoint: %s\n", endpoint)
        err := simulateNetworkOperation(endpoint)
        handleNetworkError("connect", err)
    }

    fmt.Println("\n=== Key Principles ===")
    fmt.Println("1. Behavior interfaces keep error handling decoupled")
    fmt.Println("2. Multiple interfaces can be implemented by single error type")
    fmt.Println("3. Error handling logic based on behavior, not concrete types")
    fmt.Println("4. Integrates well with standard library error behaviors")
    fmt.Println("5. Enables sophisticated error handling without type switching")
    fmt.Println("6. Maintains flexibility for error handling improvements")
    fmt.Println("7. Behavior as context is safer than type as context")
}
```

**Prerequisites**: Module 23
