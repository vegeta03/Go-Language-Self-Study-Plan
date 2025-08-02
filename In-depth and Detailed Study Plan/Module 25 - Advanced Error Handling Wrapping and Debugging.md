# Module 25: Advanced Error Handling: Wrapping and Debugging

**Duration**: 1 hour (45 min theory + 15 min hands-on)

**Learning Objectives**:

- Master error wrapping and unwrapping with Go 1.13+ features
- Learn error chain analysis and debugging techniques
- Understand when and how to wrap errors effectively
- Apply advanced error handling patterns in automation systems
- Design error handling for complex automation workflows

**Videos Covered**:

- 6.6 Find the Bug (0:08:52)
- 6.7 Wrapping Errors (0:14:30)

**Key Concepts**:

- Error wrapping: fmt.Errorf with %w verb
- Error unwrapping: errors.Unwrap, errors.Is, errors.As
- Error chains: maintaining context through call stack
- Debugging with error chains
- When to wrap vs when to handle errors
- Error handling in complex automation pipelines

**Hands-on Exercise 1: Error Wrapping and Unwrapping with Go 1.13+ Features**:

```go
// Demonstrating error wrapping and unwrapping in automation systems
package main

import (
    "errors"
    "fmt"
    "time"
)

// PART 1: Basic Error Wrapping with fmt.Errorf and %w verb

// Base errors that can be wrapped
var (
    ErrDatabaseConnection = errors.New("database connection failed")
    ErrInvalidCredentials = errors.New("invalid credentials")
    ErrRateLimitExceeded  = errors.New("rate limit exceeded")
    ErrServiceUnavailable = errors.New("service unavailable")
    ErrConfigurationMissing = errors.New("configuration missing")
)

// Custom error type that supports unwrapping
type AutomationError struct {
    Operation string
    Component string
    Timestamp time.Time
    Cause     error
}

func (ae *AutomationError) Error() string {
    return fmt.Sprintf("automation error in %s during %s at %s: %v",
        ae.Component, ae.Operation, ae.Timestamp.Format("15:04:05"), ae.Cause)
}

func (ae *AutomationError) Unwrap() error {
    return ae.Cause
}

// PART 2: Low-level functions that return base errors

func connectDatabase(connectionString string) error {
    fmt.Printf("Connecting to database: %s\n", connectionString)

    switch connectionString {
    case "":
        return ErrDatabaseConnection
    case "invalid-host":
        return fmt.Errorf("invalid host in connection string: %w", ErrDatabaseConnection)
    case "timeout":
        return fmt.Errorf("connection timeout after 30s: %w", ErrDatabaseConnection)
    default:
        fmt.Println("  ✅… Database connected successfully")
        return nil
    }
}

func authenticateService(username, password string) error {
    fmt.Printf("Authenticating service: %s\n", username)

    if username == "" || password == "" {
        return ErrInvalidCredentials
    }

    if username == "expired" {
        return fmt.Errorf("credentials expired for user %s: %w", username, ErrInvalidCredentials)
    }

    if username == "locked" {
        return fmt.Errorf("account locked for user %s: %w", username, ErrInvalidCredentials)
    }

    fmt.Printf("  ✅… Service authenticated successfully: %s\n", username)
    return nil
}

func callExternalAPI(endpoint string, operation string) error {
    fmt.Printf("Calling external API: %s for %s\n", endpoint, operation)

    switch endpoint {
    case "rate-limited.api.com":
        return fmt.Errorf("API call to %s failed: %w", endpoint, ErrRateLimitExceeded)
    case "maintenance.api.com":
        return fmt.Errorf("API %s is under maintenance: %w", endpoint, ErrServiceUnavailable)
    case "slow.api.com":
        return fmt.Errorf("API call to %s timed out during %s: %w", endpoint, operation, ErrServiceUnavailable)
    default:
        fmt.Printf("  ✅… API call successful: %s\n", endpoint)
        return nil
    }
}

// PART 3: Mid-level functions that wrap errors with additional context

func initializeDatabase(config map[string]string) error {
    connectionString, exists := config["database_url"]
    if !exists {
        return fmt.Errorf("database initialization failed: %w", ErrConfigurationMissing)
    }

    if err := connectDatabase(connectionString); err != nil {
        // Wrap the error with additional context
        return fmt.Errorf("failed to initialize database with config %v: %w", config, err)
    }

    return nil
}

func setupAuthentication(config map[string]string) error {
    username := config["service_username"]
    password := config["service_password"]

    if err := authenticateService(username, password); err != nil {
        // Wrap with context about the setup operation
        return fmt.Errorf("authentication setup failed for service: %w", err)
    }

    return nil
}

func fetchDataFromAPI(endpoint string, dataType string) ([]byte, error) {
    if err := callExternalAPI(endpoint, "fetch_"+dataType); err != nil {
        // Wrap with context about what data we were trying to fetch
        return nil, fmt.Errorf("failed to fetch %s data from API: %w", dataType, err)
    }

    // Simulate successful data fetch
    data := []byte(fmt.Sprintf(`{"type":"%s","data":"sample_%s_data"}`, dataType, dataType))
    return data, nil
}

// PART 4: High-level automation service that creates error chains

type AutomationService struct {
    Name   string
    Config map[string]string
}

func NewAutomationService(name string, config map[string]string) *AutomationService {
    return &AutomationService{
        Name:   name,
        Config: config,
    }
}

func (as *AutomationService) Initialize() error {
    fmt.Printf("\n=== Initializing %s ===\n", as.Name)

    // Initialize database - can create error chain
    if err := initializeDatabase(as.Config); err != nil {
        return fmt.Errorf("service %s initialization failed: %w", as.Name, err)
    }

    // Setup authentication - can create error chain
    if err := setupAuthentication(as.Config); err != nil {
        return fmt.Errorf("service %s initialization failed: %w", as.Name, err)
    }

    fmt.Printf("✅… Service %s initialized successfully\n", as.Name)
    return nil
}

func (as *AutomationService) ProcessWorkflow(workflowID string, endpoints []string) error {
    fmt.Printf("\n=== Processing Workflow %s ===\n", workflowID)

    for i, endpoint := range endpoints {
        dataType := fmt.Sprintf("dataset_%d", i+1)

        data, err := fetchDataFromAPI(endpoint, dataType)
        if err != nil {
            // Create custom error with unwrap support
            automationErr := &AutomationError{
                Operation: "process_workflow",
                Component: "data_fetcher",
                Timestamp: time.Now(),
                Cause:     err,
            }

            // Wrap the custom error with workflow context
            return fmt.Errorf("workflow %s failed at step %d: %w", workflowID, i+1, automationErr)
        }

        fmt.Printf("  ✅… Processed %d bytes from %s\n", len(data), endpoint)
    }

    fmt.Printf("✅… Workflow %s completed successfully\n", workflowID)
    return nil
}

// PART 5: Error unwrapping and analysis functions

func demonstrateErrorUnwrapping(err error) {
    if err == nil {
        return
    }

    fmt.Printf("\n=== Error Unwrapping Demonstration ===\n")
    fmt.Printf("Original error: %v\n", err)

    // Manual unwrapping using errors.Unwrap
    fmt.Println("\nManual unwrapping chain:")
    current := err
    level := 0

    for current != nil {
        fmt.Printf("  Level %d: %v\n", level, current)
        current = errors.Unwrap(current)
        level++

        if level > 10 { // Prevent infinite loops
            fmt.Println("  ... (stopping to prevent infinite loop)")
            break
        }
    }
}

func demonstrateErrorIs(err error) {
    if err == nil {
        return
    }

    fmt.Printf("\n=== errors.Is() Demonstration ===\n")

    // Check for specific base errors in the chain
    baseErrors := []error{
        ErrDatabaseConnection,
        ErrInvalidCredentials,
        ErrRateLimitExceeded,
        ErrServiceUnavailable,
        ErrConfigurationMissing,
    }

    for _, baseErr := range baseErrors {
        if errors.Is(err, baseErr) {
            fmt.Printf("✅… Found %v in error chain\n", baseErr)
        }
    }
}

func demonstrateErrorAs(err error) {
    if err == nil {
        return
    }

    fmt.Printf("\n=== errors.As() Demonstration ===\n")

    // Try to extract custom error type from chain
    var automationErr *AutomationError
    if errors.As(err, &automationErr) {
        fmt.Printf("✅… Found AutomationError in chain:\n")
        fmt.Printf("  Operation: %s\n", automationErr.Operation)
        fmt.Printf("  Component: %s\n", automationErr.Component)
        fmt.Printf("  Timestamp: %s\n", automationErr.Timestamp.Format("15:04:05"))
        fmt.Printf("  Underlying cause: %v\n", automationErr.Cause)
    } else {
        fmt.Printf("❌ No AutomationError found in chain\n")
    }
}

func main() {
    fmt.Println("=== Error Wrapping and Unwrapping with Go 1.13+ Features ===")

    // Test scenarios that create different error chains
    testScenarios := []struct {
        name      string
        config    map[string]string
        endpoints []string
    }{
        {
            name: "Missing Database Configuration",
            config: map[string]string{
                "service_username": "admin",
                "service_password": "secret",
            },
            endpoints: []string{"api.example.com"},
        },
        {
            name: "Invalid Database Connection",
            config: map[string]string{
                "database_url":     "invalid-host",
                "service_username": "admin",
                "service_password": "secret",
            },
            endpoints: []string{"api.example.com"},
        },
        {
            name: "Authentication Failure",
            config: map[string]string{
                "database_url":     "postgres://localhost:5432/db",
                "service_username": "expired",
                "service_password": "secret",
            },
            endpoints: []string{"api.example.com"},
        },
        {
            name: "API Rate Limiting",
            config: map[string]string{
                "database_url":     "postgres://localhost:5432/db",
                "service_username": "admin",
                "service_password": "secret",
            },
            endpoints: []string{"rate-limited.api.com", "api.example.com"},
        },
        {
            name: "Multiple API Failures",
            config: map[string]string{
                "database_url":     "postgres://localhost:5432/db",
                "service_username": "admin",
                "service_password": "secret",
            },
            endpoints: []string{"api.example.com", "maintenance.api.com", "slow.api.com"},
        },
        {
            name: "Successful Execution",
            config: map[string]string{
                "database_url":     "postgres://localhost:5432/db",
                "service_username": "admin",
                "service_password": "secret",
            },
            endpoints: []string{"api.example.com", "data.example.com"},
        },
    }

    for i, scenario := range testScenarios {
        fmt.Printf("\n" + "="*60 + "\n")
        fmt.Printf("Scenario %d: %s\n", i+1, scenario.name)
        fmt.Printf("="*60 + "\n")

        service := NewAutomationService(fmt.Sprintf("Service-%d", i+1), scenario.config)

        // Try to initialize service
        err := service.Initialize()
        if err != nil {
            fmt.Printf("❌ Initialization failed: %v\n", err)
            demonstrateErrorUnwrapping(err)
            demonstrateErrorIs(err)
            demonstrateErrorAs(err)
            continue
        }

        // Try to process workflow
        workflowID := fmt.Sprintf("WF-%03d", i+1)
        err = service.ProcessWorkflow(workflowID, scenario.endpoints)
        if err != nil {
            fmt.Printf("❌ Workflow failed: %v\n", err)
            demonstrateErrorUnwrapping(err)
            demonstrateErrorIs(err)
            demonstrateErrorAs(err)
        }
    }

    fmt.Printf("\n" + "="*60 + "\n")
    fmt.Println("=== Key Concepts Summary ===")
    fmt.Println("1. Use fmt.Errorf with %w verb to wrap errors")
    fmt.Println("2. Implement Unwrap() method for custom error types")
    fmt.Println("3. errors.Unwrap() manually walks the error chain")
    fmt.Println("4. errors.Is() checks for specific errors in the chain")
    fmt.Println("5. errors.As() extracts specific error types from the chain")
    fmt.Println("6. Error wrapping preserves original error information")
    fmt.Println("7. Each wrap adds context without losing underlying cause")
}
```

**Hands-on Exercise 2: Error Chain Analysis and Debugging Techniques**:

```go
// Demonstrating advanced error chain analysis and debugging in automation systems
package main

import (
    "errors"
    "fmt"
    "runtime"
    "strings"
    "time"
)

// PART 1: Enhanced Error Types with Debug Information

// DebugError captures detailed debugging information
type DebugError struct {
    Message    string
    Code       string
    Timestamp  time.Time
    StackTrace []string
    Context    map[string]interface{}
    Cause      error
}

func (de *DebugError) Error() string {
    return fmt.Sprintf("[%s] %s at %s", de.Code, de.Message, de.Timestamp.Format("15:04:05.000"))
}

func (de *DebugError) Unwrap() error {
    return de.Cause
}

// GetStackTrace returns formatted stack trace
func (de *DebugError) GetStackTrace() []string {
    return de.StackTrace
}

// GetContext returns debugging context
func (de *DebugError) GetContext() map[string]interface{} {
    return de.Context
}

// NewDebugError creates a debug error with stack trace
func NewDebugError(code, message string, cause error) *DebugError {
    // Capture stack trace
    stackTrace := make([]string, 0, 10)
    for i := 1; i < 10; i++ {
        pc, file, line, ok := runtime.Caller(i)
        if !ok {
            break
        }

        fn := runtime.FuncForPC(pc)
        if fn == nil {
            continue
        }

        // Format: function_name (file:line)
        funcName := fn.Name()
        if lastSlash := strings.LastIndex(funcName, "/"); lastSlash >= 0 {
            funcName = funcName[lastSlash+1:]
        }

        stackTrace = append(stackTrace, fmt.Sprintf("%s (%s:%d)", funcName, file, line))
    }

    return &DebugError{
        Message:    message,
        Code:       code,
        Timestamp:  time.Now(),
        StackTrace: stackTrace,
        Context:    make(map[string]interface{}),
        Cause:      cause,
    }
}

// AddContext adds debugging context to the error
func (de *DebugError) AddContext(key string, value interface{}) *DebugError {
    de.Context[key] = value
    return de
}

// PART 2: Error Chain Walker and Analyzer

type ErrorAnalyzer struct {
    name string
}

func NewErrorAnalyzer(name string) *ErrorAnalyzer {
    return &ErrorAnalyzer{name: name}
}

// AnalyzeErrorChain provides comprehensive error chain analysis
func (ea *ErrorAnalyzer) AnalyzeErrorChain(err error) {
    if err == nil {
        fmt.Printf("\n=== %s: No Error to Analyze ===\n", ea.name)
        return
    }

    fmt.Printf("\n=== %s: Error Chain Analysis ===\n", ea.name)
    fmt.Printf("Root Error: %v\n", err)

    // Walk the entire error chain
    ea.walkErrorChain(err)

    // Analyze error types in chain
    ea.analyzeErrorTypes(err)

    // Check for known error patterns
    ea.checkErrorPatterns(err)

    // Extract debugging information
    ea.extractDebugInfo(err)
}

func (ea *ErrorAnalyzer) walkErrorChain(err error) {
    fmt.Printf("\n--- Error Chain Walk ---\n")

    current := err
    level := 0

    for current != nil {
        fmt.Printf("Level %d: %v\n", level, current)

        // Show error type information
        fmt.Printf("  Type: %T\n", current)

        // Check if this error has additional methods
        if unwrapper, ok := current.(interface{ Unwrap() error }); ok {
            fmt.Printf("  Supports Unwrap: Yes\n")
            current = unwrapper.Unwrap()
        } else {
            fmt.Printf("  Supports Unwrap: No\n")
            current = nil
        }

        level++
        if level > 15 { // Prevent infinite loops
            fmt.Printf("  ... (stopping at level %d to prevent infinite loop)\n", level)
            break
        }
    }
}

func (ea *ErrorAnalyzer) analyzeErrorTypes(err error) {
    fmt.Printf("\n--- Error Type Analysis ---\n")

    // Count different error types in the chain
    typeCount := make(map[string]int)
    current := err

    for current != nil {
        typeName := fmt.Sprintf("%T", current)
        typeCount[typeName]++
        current = errors.Unwrap(current)
    }

    fmt.Printf("Error types found in chain:\n")
    for typeName, count := range typeCount {
        fmt.Printf("  %s: %d occurrence(s)\n", typeName, count)
    }
}

func (ea *ErrorAnalyzer) checkErrorPatterns(err error) {
    fmt.Printf("\n--- Error Pattern Analysis ---\n")

    // Check for common error patterns
    patterns := []struct {
        name        string
        checkFunc   func(error) bool
        description string
    }{
        {
            "Wrapped Standard Error",
            func(e error) bool {
                return strings.Contains(e.Error(), ":")
            },
            "Error appears to be wrapped with additional context",
        },
        {
            "Timeout Pattern",
            func(e error) bool {
                errStr := strings.ToLower(e.Error())
                return strings.Contains(errStr, "timeout") || strings.Contains(errStr, "timed out")
            },
            "Error indicates a timeout condition",
        },
        {
            "Connection Pattern",
            func(e error) bool {
                errStr := strings.ToLower(e.Error())
                return strings.Contains(errStr, "connection") || strings.Contains(errStr, "connect")
            },
            "Error indicates a connection issue",
        },
        {
            "Authentication Pattern",
            func(e error) bool {
                errStr := strings.ToLower(e.Error())
                return strings.Contains(errStr, "auth") || strings.Contains(errStr, "credential")
            },
            "Error indicates an authentication issue",
        },
        {
            "Configuration Pattern",
            func(e error) bool {
                errStr := strings.ToLower(e.Error())
                return strings.Contains(errStr, "config") || strings.Contains(errStr, "setting")
            },
            "Error indicates a configuration issue",
        },
    }

    for _, pattern := range patterns {
        if pattern.checkFunc(err) {
            fmt.Printf("✅… %s: %s\n", pattern.name, pattern.description)
        }
    }
}

func (ea *ErrorAnalyzer) extractDebugInfo(err error) {
    fmt.Printf("\n--- Debug Information Extraction ---\n")

    // Try to extract DebugError information
    var debugErr *DebugError
    if errors.As(err, &debugErr) {
        fmt.Printf("✅… Found DebugError with detailed information:\n")
        fmt.Printf("  Code: %s\n", debugErr.Code)
        fmt.Printf("  Timestamp: %s\n", debugErr.Timestamp.Format("2006-01-02 15:04:05.000"))

        if len(debugErr.Context) > 0 {
            fmt.Printf("  Context:\n")
            for key, value := range debugErr.Context {
                fmt.Printf("    %s: %v\n", key, value)
            }
        }

        if len(debugErr.StackTrace) > 0 {
            fmt.Printf("  Stack Trace:\n")
            for i, frame := range debugErr.StackTrace {
                fmt.Printf("    %d: %s\n", i, frame)
            }
        }
    } else {
        fmt.Printf("❌ No DebugError found in chain\n")
    }
}

// PART 3: Automation Components that Generate Complex Error Chains

type DatabaseManager struct {
    connectionString string
    maxRetries      int
}

func NewDatabaseManager(connectionString string) *DatabaseManager {
    return &DatabaseManager{
        connectionString: connectionString,
        maxRetries:      3,
    }
}

func (dm *DatabaseManager) Connect() error {
    switch dm.connectionString {
    case "":
        return NewDebugError("DB_CONN_001", "empty connection string", nil).
            AddContext("component", "DatabaseManager").
            AddContext("operation", "Connect").
            AddContext("connection_string", dm.connectionString)

    case "invalid-host":
        baseErr := errors.New("no such host")
        return NewDebugError("DB_CONN_002", "invalid database host", baseErr).
            AddContext("component", "DatabaseManager").
            AddContext("operation", "Connect").
            AddContext("connection_string", dm.connectionString).
            AddContext("retry_count", 0)

    case "timeout":
        baseErr := errors.New("connection timeout")
        return NewDebugError("DB_CONN_003", "database connection timeout", baseErr).
            AddContext("component", "DatabaseManager").
            AddContext("operation", "Connect").
            AddContext("timeout_seconds", 30).
            AddContext("max_retries", dm.maxRetries)

    default:
        fmt.Printf("✅… Database connected: %s\n", dm.connectionString)
        return nil
    }
}

func (dm *DatabaseManager) ExecuteQuery(query string) ([]map[string]interface{}, error) {
    if query == "" {
        return nil, NewDebugError("DB_QUERY_001", "empty query string", nil).
            AddContext("component", "DatabaseManager").
            AddContext("operation", "ExecuteQuery").
            AddContext("query", query)
    }

    if strings.Contains(strings.ToLower(query), "drop") {
        baseErr := errors.New("dangerous operation detected")
        return nil, NewDebugError("DB_QUERY_002", "query contains dangerous operations", baseErr).
            AddContext("component", "DatabaseManager").
            AddContext("operation", "ExecuteQuery").
            AddContext("query", query).
            AddContext("danger_level", "high")
    }

    // Simulate successful query
    result := []map[string]interface{}{
        {"id": 1, "name": "test_record"},
    }
    return result, nil
}

type APIClient struct {
    baseURL    string
    timeout    time.Duration
    retryCount int
}

func NewAPIClient(baseURL string) *APIClient {
    return &APIClient{
        baseURL:    baseURL,
        timeout:    30 * time.Second,
        retryCount: 3,
    }
}

func (ac *APIClient) FetchData(endpoint string) ([]byte, error) {
    fullURL := ac.baseURL + "/" + endpoint

    switch endpoint {
    case "rate-limited":
        baseErr := errors.New("429 Too Many Requests")
        return nil, NewDebugError("API_RATE_001", "API rate limit exceeded", baseErr).
            AddContext("component", "APIClient").
            AddContext("operation", "FetchData").
            AddContext("endpoint", endpoint).
            AddContext("full_url", fullURL).
            AddContext("retry_after", "60s")

    case "server-error":
        baseErr := errors.New("500 Internal Server Error")
        return nil, NewDebugError("API_ERR_001", "API server error", baseErr).
            AddContext("component", "APIClient").
            AddContext("operation", "FetchData").
            AddContext("endpoint", endpoint).
            AddContext("full_url", fullURL).
            AddContext("server_status", "degraded")

    case "timeout":
        baseErr := errors.New("request timeout")
        return nil, NewDebugError("API_TIMEOUT_001", "API request timeout", baseErr).
            AddContext("component", "APIClient").
            AddContext("operation", "FetchData").
            AddContext("endpoint", endpoint).
            AddContext("timeout_duration", ac.timeout).
            AddContext("retry_count", ac.retryCount)

    default:
        data := []byte(fmt.Sprintf(`{"endpoint":"%s","data":"sample_data"}`, endpoint))
        return data, nil
    }
}

// PART 4: High-level Service that Creates Complex Error Chains

type DataProcessingService struct {
    name      string
    dbManager *DatabaseManager
    apiClient *APIClient
}

func NewDataProcessingService(name string, dbConnString, apiBaseURL string) *DataProcessingService {
    return &DataProcessingService{
        name:      name,
        dbManager: NewDatabaseManager(dbConnString),
        apiClient: NewAPIClient(apiBaseURL),
    }
}

func (dps *DataProcessingService) ProcessDataPipeline(pipelineID string, endpoints []string) error {
    fmt.Printf("\n=== Processing Pipeline %s ===\n", pipelineID)

    // Step 1: Connect to database
    if err := dps.dbManager.Connect(); err != nil {
        return fmt.Errorf("pipeline %s failed at database connection: %w", pipelineID, err)
    }

    // Step 2: Process each endpoint
    for i, endpoint := range endpoints {
        stepID := fmt.Sprintf("step_%d", i+1)

        // Fetch data from API
        data, err := dps.apiClient.FetchData(endpoint)
        if err != nil {
            return fmt.Errorf("pipeline %s failed at %s (endpoint %s): %w", pipelineID, stepID, endpoint, err)
        }

        // Store data in database
        query := fmt.Sprintf("INSERT INTO pipeline_data (pipeline_id, step_id, data) VALUES ('%s', '%s', '%s')",
            pipelineID, stepID, string(data))

        _, err = dps.dbManager.ExecuteQuery(query)
        if err != nil {
            return fmt.Errorf("pipeline %s failed to store data at %s: %w", pipelineID, stepID, err)
        }

        fmt.Printf("  ✅… Step %s completed successfully\n", stepID)
    }

    fmt.Printf("✅… Pipeline %s completed successfully\n", pipelineID)
    return nil
}

func main() {
    fmt.Println("=== Error Chain Analysis and Debugging Techniques ===")

    analyzer := NewErrorAnalyzer("AutomationDebugger")

    // Test scenarios that create complex error chains
    testScenarios := []struct {
        name        string
        dbConnStr   string
        apiBaseURL  string
        endpoints   []string
    }{
        {
            name:       "Database Connection Failure",
            dbConnStr:  "",
            apiBaseURL: "https://api.example.com",
            endpoints:  []string{"data"},
        },
        {
            name:       "Invalid Database Host",
            dbConnStr:  "invalid-host",
            apiBaseURL: "https://api.example.com",
            endpoints:  []string{"data"},
        },
        {
            name:       "Database Timeout",
            dbConnStr:  "timeout",
            apiBaseURL: "https://api.example.com",
            endpoints:  []string{"data"},
        },
        {
            name:       "API Rate Limiting",
            dbConnStr:  "postgres://localhost:5432/db",
            apiBaseURL: "https://api.example.com",
            endpoints:  []string{"rate-limited"},
        },
        {
            name:       "API Server Error",
            dbConnStr:  "postgres://localhost:5432/db",
            apiBaseURL: "https://api.example.com",
            endpoints:  []string{"server-error"},
        },
        {
            name:       "Multiple Failure Points",
            dbConnStr:  "postgres://localhost:5432/db",
            apiBaseURL: "https://api.example.com",
            endpoints:  []string{"data", "timeout", "server-error"},
        },
        {
            name:       "Successful Pipeline",
            dbConnStr:  "postgres://localhost:5432/db",
            apiBaseURL: "https://api.example.com",
            endpoints:  []string{"users", "orders", "products"},
        },
    }

    for i, scenario := range testScenarios {
        fmt.Printf("\n" + "="*70 + "\n")
        fmt.Printf("Scenario %d: %s\n", i+1, scenario.name)
        fmt.Printf("="*70 + "\n")

        service := NewDataProcessingService(
            fmt.Sprintf("Service-%d", i+1),
            scenario.dbConnStr,
            scenario.apiBaseURL,
        )

        pipelineID := fmt.Sprintf("PIPELINE-%03d", i+1)
        err := service.ProcessDataPipeline(pipelineID, scenario.endpoints)

        if err != nil {
            fmt.Printf("❌ Pipeline failed: %v\n", err)
            analyzer.AnalyzeErrorChain(err)
        }
    }

    fmt.Printf("\n" + "="*70 + "\n")
    fmt.Println("=== Key Debugging Techniques Summary ===")
    fmt.Println("1. Capture stack traces at error creation points")
    fmt.Println("2. Add contextual information to errors")
    fmt.Println("3. Walk error chains to understand failure progression")
    fmt.Println("4. Analyze error types and patterns")
    fmt.Println("5. Extract debugging information from custom error types")
    fmt.Println("6. Use error codes for programmatic error handling")
    fmt.Println("7. Include timestamps for temporal analysis")
}
```

**Hands-on Exercise 3: Advanced Error Handling Patterns and Bug Prevention**:

```go
// Demonstrating advanced error handling patterns and common bug prevention
package main

import (
    "errors"
    "fmt"
    "time"
)

// PART 1: Common Error Handling Bugs and How to Avoid Them

// BUG 1: Returning concrete nil instead of interface nil
type CustomError struct {
    Code    string
    Message string
}

func (ce *CustomError) Error() string {
    return fmt.Sprintf("[%s] %s", ce.Code, ce.Message)
}

// WRONG: This function has a subtle bug
func failWrong() error {
    var err *CustomError // This is nil, but it's a typed nil

    // Some condition that doesn't trigger
    if false {
        err = &CustomError{Code: "ERR001", Message: "something went wrong"}
    }

    // BUG: Returning typed nil instead of interface nil
    return err // This returns (*CustomError)(nil), not nil interface
}

// CORRECT: This function handles nil properly
func failCorrect() error {
    var err *CustomError

    if false {
        err = &CustomError{Code: "ERR001", Message: "something went wrong"}
    }

    // CORRECT: Check for nil and return interface nil
    if err == nil {
        return nil
    }
    return err
}

// BUG 2: Value semantics vs Pointer semantics for error interface
type ValueSemanticError struct {
    Code string
}

func (vse ValueSemanticError) Error() string {
    return fmt.Sprintf("value error: %s", vse.Code)
}

type PointerSemanticError struct {
    Code string
}

func (pse *PointerSemanticError) Error() string {
    return fmt.Sprintf("pointer error: %s", pse.Code)
}

func demonstrateSemanticsBug() {
    fmt.Println("=== Semantics Bug Demonstration ===")

    // Create error variables for comparison
    errValue1 := ValueSemanticError{Code: "VAL001"}
    errValue2 := ValueSemanticError{Code: "VAL001"}

    errPointer1 := &PointerSemanticError{Code: "PTR001"}
    errPointer2 := &PointerSemanticError{Code: "PTR001"}

    // Value semantics: comparison works as expected
    fmt.Printf("Value semantic errors equal: %t\n", errValue1 == errValue2) // true

    // Pointer semantics: comparison fails even with same content
    fmt.Printf("Pointer semantic errors equal: %t\n", errPointer1 == errPointer2) // false

    // This is why we use pointer semantics for error interface implementation
    // but be careful with error variable comparisons
}

// PART 2: Advanced Error Handling Patterns

// Pattern 1: Error Sentinel with Context
type ErrorSentinel struct {
    sentinel error
    context  map[string]interface{}
}

func NewErrorSentinel(sentinel error) *ErrorSentinel {
    return &ErrorSentinel{
        sentinel: sentinel,
        context:  make(map[string]interface{}),
    }
}

func (es *ErrorSentinel) Error() string {
    return es.sentinel.Error()
}

func (es *ErrorSentinel) Is(target error) bool {
    return errors.Is(es.sentinel, target)
}

func (es *ErrorSentinel) AddContext(key string, value interface{}) *ErrorSentinel {
    es.context[key] = value
    return es
}

func (es *ErrorSentinel) GetContext() map[string]interface{} {
    return es.context
}

// Pattern 2: Error Aggregation for Multiple Failures
type MultiError struct {
    errors []error
}

func NewMultiError() *MultiError {
    return &MultiError{
        errors: make([]error, 0),
    }
}

func (me *MultiError) Add(err error) {
    if err != nil {
        me.errors = append(me.errors, err)
    }
}

func (me *MultiError) Error() string {
    if len(me.errors) == 0 {
        return "no errors"
    }

    if len(me.errors) == 1 {
        return me.errors[0].Error()
    }

    return fmt.Sprintf("multiple errors occurred: %d total", len(me.errors))
}

func (me *MultiError) Errors() []error {
    return me.errors
}

func (me *MultiError) HasErrors() bool {
    return len(me.errors) > 0
}

func (me *MultiError) ErrorOrNil() error {
    if len(me.errors) == 0 {
        return nil
    }
    return me
}

// Pattern 3: Error Recovery with Circuit Breaker
type CircuitBreaker struct {
    name           string
    failureCount   int
    maxFailures    int
    resetTimeout   time.Duration
    lastFailure    time.Time
    state          string // "closed", "open", "half-open"
}

func NewCircuitBreaker(name string, maxFailures int, resetTimeout time.Duration) *CircuitBreaker {
    return &CircuitBreaker{
        name:         name,
        maxFailures:  maxFailures,
        resetTimeout: resetTimeout,
        state:        "closed",
    }
}

func (cb *CircuitBreaker) Call(operation func() error) error {
    // Check if circuit should be reset
    if cb.state == "open" && time.Since(cb.lastFailure) > cb.resetTimeout {
        cb.state = "half-open"
        cb.failureCount = 0
        fmt.Printf("Circuit breaker %s: transitioning to half-open\n", cb.name)
    }

    // Reject calls if circuit is open
    if cb.state == "open" {
        return fmt.Errorf("circuit breaker %s is open", cb.name)
    }

    // Execute operation
    err := operation()

    if err != nil {
        cb.failureCount++
        cb.lastFailure = time.Now()

        // Open circuit if failure threshold reached
        if cb.failureCount >= cb.maxFailures {
            cb.state = "open"
            fmt.Printf("Circuit breaker %s: opening due to %d failures\n", cb.name, cb.failureCount)
        }

        return fmt.Errorf("operation failed (circuit: %s, failures: %d): %w", cb.state, cb.failureCount, err)
    }

    // Success - reset failure count and close circuit
    if cb.state == "half-open" {
        cb.state = "closed"
        fmt.Printf("Circuit breaker %s: closing after successful call\n", cb.name)
    }
    cb.failureCount = 0

    return nil
}

// PART 3: Automation System with Advanced Error Handling

type AutomationService struct {
    name           string
    circuitBreaker *CircuitBreaker
}

func NewAutomationService(name string) *AutomationService {
    return &AutomationService{
        name:           name,
        circuitBreaker: NewCircuitBreaker(name+"_cb", 3, 10*time.Second),
    }
}

func (as *AutomationService) ProcessBatch(items []string) error {
    fmt.Printf("\n=== Processing batch of %d items ===\n", len(items))

    multiErr := NewMultiError()
    successCount := 0

    for i, item := range items {
        fmt.Printf("Processing item %d: %s\n", i+1, item)

        err := as.circuitBreaker.Call(func() error {
            return as.processItem(item)
        })

        if err != nil {
            fmt.Printf("  ❌ Failed: %v\n", err)
            multiErr.Add(fmt.Errorf("item %d (%s): %w", i+1, item, err))
        } else {
            fmt.Printf("  ✅… Success\n")
            successCount++
        }
    }

    fmt.Printf("\nBatch processing complete: %d/%d successful\n", successCount, len(items))

    return multiErr.ErrorOrNil()
}

func (as *AutomationService) processItem(item string) error {
    // Simulate different processing scenarios
    switch item {
    case "fail":
        return errors.New("processing failed")
    case "timeout":
        return errors.New("processing timeout")
    case "invalid":
        return errors.New("invalid item format")
    default:
        // Simulate processing time
        time.Sleep(10 * time.Millisecond)
        return nil
    }
}

// PART 4: Error Handling Best Practices Demonstration

func demonstrateBugPrevention() {
    fmt.Println("=== Bug Prevention Demonstration ===")

    // Bug 1: Typed nil vs interface nil
    fmt.Println("\n--- Typed Nil Bug ---")

    errWrong := failWrong()
    errCorrect := failCorrect()

    fmt.Printf("Wrong implementation (errWrong != nil): %t\n", errWrong != nil)   // true (BUG!)
    fmt.Printf("Correct implementation (errCorrect != nil): %t\n", errCorrect != nil) // false

    // Bug 2: Semantics comparison issues
    fmt.Println("\n--- Semantics Bug ---")
    demonstrateSemanticsBug()
}

func demonstrateAdvancedPatterns() {
    fmt.Println("\n=== Advanced Error Patterns ===")

    // Pattern 1: Error Sentinel with Context
    fmt.Println("\n--- Error Sentinel Pattern ---")

    baseErr := errors.New("database connection failed")
    sentinelErr := NewErrorSentinel(baseErr).
        AddContext("host", "db.example.com").
        AddContext("port", 5432).
        AddContext("retry_count", 3)

    fmt.Printf("Sentinel error: %v\n", sentinelErr)
    fmt.Printf("Is base error: %t\n", errors.Is(sentinelErr, baseErr))
    fmt.Printf("Context: %+v\n", sentinelErr.GetContext())

    // Pattern 2: Multi-Error Aggregation
    fmt.Println("\n--- Multi-Error Pattern ---")

    multiErr := NewMultiError()
    multiErr.Add(errors.New("validation failed"))
    multiErr.Add(errors.New("network timeout"))
    multiErr.Add(errors.New("permission denied"))

    if multiErr.HasErrors() {
        fmt.Printf("Aggregated error: %v\n", multiErr)
        fmt.Printf("Individual errors:\n")
        for i, err := range multiErr.Errors() {
            fmt.Printf("  %d: %v\n", i+1, err)
        }
    }
}

func demonstrateCircuitBreaker() {
    fmt.Println("\n=== Circuit Breaker Pattern ===")

    service := NewAutomationService("TestService")

    // Test batch with mixed success/failure items
    testBatches := [][]string{
        {"item1", "item2", "fail", "item4"},           // Some failures
        {"fail", "timeout", "invalid", "fail"},        // All failures (should open circuit)
        {"item1", "item2", "item3"},                   // Should be rejected (circuit open)
    }

    for i, batch := range testBatches {
        fmt.Printf("\n--- Batch %d ---\n", i+1)

        err := service.ProcessBatch(batch)
        if err != nil {
            if multiErr, ok := err.(*MultiError); ok {
                fmt.Printf("Batch failed with %d errors:\n", len(multiErr.Errors()))
                for j, e := range multiErr.Errors() {
                    fmt.Printf("  %d: %v\n", j+1, e)
                }
            } else {
                fmt.Printf("Batch failed: %v\n", err)
            }
        } else {
            fmt.Printf("Batch completed successfully\n")
        }

        // Wait a bit between batches
        time.Sleep(2 * time.Second)
    }

    // Wait for circuit breaker reset and try again
    fmt.Println("\n--- Waiting for circuit breaker reset ---")
    time.Sleep(11 * time.Second)

    fmt.Println("\n--- Retry after circuit reset ---")
    err := service.ProcessBatch([]string{"item1", "item2", "item3"})
    if err != nil {
        fmt.Printf("Retry failed: %v\n", err)
    } else {
        fmt.Printf("Retry successful\n")
    }
}

func main() {
    fmt.Println("=== Advanced Error Handling Patterns and Bug Prevention ===")

    demonstrateBugPrevention()
    demonstrateAdvancedPatterns()
    demonstrateCircuitBreaker()

    fmt.Println("\n=== Best Practices Summary ===")
    fmt.Println("1. Always return interface nil, not typed nil")
    fmt.Println("2. Use pointer semantics for error interface implementation")
    fmt.Println("3. Be careful with error variable comparisons")
    fmt.Println("4. Use error sentinels with context for rich error information")
    fmt.Println("5. Aggregate multiple errors when processing batches")
    fmt.Println("6. Implement circuit breakers for resilient error handling")
    fmt.Println("7. Design error handling as part of the happy path")
    fmt.Println("8. Provide enough context for informed error recovery decisions")
```

**Prerequisites**: Module 24

---

**Prerequisites**: Module 24
