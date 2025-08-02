# Module 27: Package-Oriented Design and Best Practices

**Duration**: 1 hour (45 min theory + 15 min hands-on)

**Learning Objectives**:

- Master package-oriented design principles
- Learn to organize code around business capabilities
- Understand package boundaries and dependencies
- Apply package design patterns to automation systems
- Design scalable, maintainable package architectures

**Videos Covered**:

- 7.4 Package-Oriented Design (0:18:26)

**Key Concepts**:

- Package-oriented design philosophy
- Organizing packages around business capabilities
- Package boundaries and cohesion
- Dependency management and inversion
- Package design patterns and anti-patterns
- Scalable package architectures

**Hands-on Exercise 1: Kit Project and Foundational Packages**:

```go
// Demonstrating kit project structure - foundational packages for automation systems
// Kit project: github.com/company/automation-kit

// File: automation-kit/logger/logger.go
package logger

import (
    "fmt"
    "io"
    "log"
    "os"
)

// FOUNDATIONAL PACKAGE: Provides logging capabilities
// Key principles:
// 1. No dependencies on other kit packages
// 2. No policy decisions (like where to log)
// 3. Simple, focused API
// 4. Highly reusable across applications

// Level represents logging levels
type Level int

const (
    DEBUG Level = iota
    INFO
    WARN
    ERROR
)

// Logger provides structured logging capabilities
type Logger struct {
    level  Level
    output io.Writer
    logger *log.Logger
}

// New creates a new logger instance
func New(level Level, output io.Writer, prefix string) *Logger {
    if output == nil {
        output = os.Stdout
    }

    return &Logger{
        level:  level,
        output: output,
        logger: log.New(output, prefix, log.LstdFlags|log.Lshortfile),
    }
}

// Debug logs a debug message
func (l *Logger) Debug(msg string) {
    if l.level <= DEBUG {
        l.logger.Printf("[DEBUG] %s", msg)
    }
}

// Info logs an info message
func (l *Logger) Info(msg string) {
    if l.level <= INFO {
        l.logger.Printf("[INFO] %s", msg)
    }
}

// Warn logs a warning message
func (l *Logger) Warn(msg string) {
    if l.level <= WARN {
        l.logger.Printf("[WARN] %s", msg)
    }
}

// Error logs an error message
func (l *Logger) Error(msg string) {
    if l.level <= ERROR {
        l.logger.Printf("[ERROR] %s", msg)
    }
}

// File: automation-kit/config/config.go
package config

import (
    "encoding/json"
    "fmt"
    "os"
    "strconv"
    "time"
)

// FOUNDATIONAL PACKAGE: Provides configuration management
// Key principles:
// 1. No logging (that would be policy)
// 2. No dependencies on other kit packages
// 3. Simple environment variable and file support
// 4. Type-safe value retrieval

// Config provides configuration management
type Config struct {
    values map[string]string
}

// New creates a new configuration instance
func New() *Config {
    return &Config{
        values: make(map[string]string),
    }
}

// LoadFromEnv loads configuration from environment variables
func (c *Config) LoadFromEnv(prefix string) {
    for _, env := range os.Environ() {
        // Simple parsing - in real implementation would be more robust
        if len(env) > len(prefix) && env[:len(prefix)] == prefix {
            parts := splitEnv(env)
            if len(parts) == 2 {
                key := parts[0][len(prefix):]
                c.values[key] = parts[1]
            }
        }
    }
}

// LoadFromFile loads configuration from JSON file
func (c *Config) LoadFromFile(filename string) error {
    data, err := os.ReadFile(filename)
    if err != nil {
        return fmt.Errorf("failed to read config file: %w", err)
    }

    var jsonData map[string]interface{}
    if err := json.Unmarshal(data, &jsonData); err != nil {
        return fmt.Errorf("failed to parse config file: %w", err)
    }

    // Convert all values to strings for simplicity
    for key, value := range jsonData {
        c.values[key] = fmt.Sprintf("%v", value)
    }

    return nil
}

// String retrieves a string value
func (c *Config) String(key, defaultValue string) string {
    if value, exists := c.values[key]; exists {
        return value
    }
    return defaultValue
}

// Int retrieves an integer value
func (c *Config) Int(key string, defaultValue int) int {
    if value, exists := c.values[key]; exists {
        if intValue, err := strconv.Atoi(value); err == nil {
            return intValue
        }
    }
    return defaultValue
}

// Bool retrieves a boolean value
func (c *Config) Bool(key string, defaultValue bool) bool {
    if value, exists := c.values[key]; exists {
        if boolValue, err := strconv.ParseBool(value); err == nil {
            return boolValue
        }
    }
    return defaultValue
}

// Duration retrieves a duration value
func (c *Config) Duration(key string, defaultValue time.Duration) time.Duration {
    if value, exists := c.values[key]; exists {
        if duration, err := time.ParseDuration(value); err == nil {
            return duration
        }
    }
    return defaultValue
}

// Helper function to split environment variable
func splitEnv(env string) []string {
    for i, char := range env {
        if char == '=' {
            return []string{env[:i], env[i+1:]}
        }
    }
    return []string{env}
}

// File: automation-kit/metrics/metrics.go
package metrics

import (
    "sync"
    "time"
)

// FOUNDATIONAL PACKAGE: Provides metrics collection
// Key principles:
// 1. No external dependencies
// 2. Thread-safe operations
// 3. Simple, focused API
// 4. No policy about where metrics go

// Metric represents a single metric
type Metric struct {
    Name      string
    Value     float64
    Tags      map[string]string
    Timestamp time.Time
}

// Collector collects and stores metrics
type Collector struct {
    mu      sync.RWMutex
    metrics map[string]*Metric
}

// New creates a new metrics collector
func New() *Collector {
    return &Collector{
        metrics: make(map[string]*Metric),
    }
}

// Record records a metric value
func (c *Collector) Record(name string, value float64, tags map[string]string) {
    c.mu.Lock()
    defer c.mu.Unlock()

    c.metrics[name] = &Metric{
        Name:      name,
        Value:     value,
        Tags:      copyTags(tags),
        Timestamp: time.Now(),
    }
}

// Get retrieves a metric by name
func (c *Collector) Get(name string) (*Metric, bool) {
    c.mu.RLock()
    defer c.mu.RUnlock()

    metric, exists := c.metrics[name]
    if !exists {
        return nil, false
    }

    // Return a copy to prevent external modification
    return &Metric{
        Name:      metric.Name,
        Value:     metric.Value,
        Tags:      copyTags(metric.Tags),
        Timestamp: metric.Timestamp,
    }, true
}

// All returns all metrics
func (c *Collector) All() []*Metric {
    c.mu.RLock()
    defer c.mu.RUnlock()

    metrics := make([]*Metric, 0, len(c.metrics))
    for _, metric := range c.metrics {
        metrics = append(metrics, &Metric{
            Name:      metric.Name,
            Value:     metric.Value,
            Tags:      copyTags(metric.Tags),
            Timestamp: metric.Timestamp,
        })
    }

    return metrics
}

// Reset clears all metrics
func (c *Collector) Reset() {
    c.mu.Lock()
    defer c.mu.Unlock()

    c.metrics = make(map[string]*Metric)
}

// Helper function to copy tags map
func copyTags(tags map[string]string) map[string]string {
    if tags == nil {
        return nil
    }

    copy := make(map[string]string, len(tags))
    for k, v := range tags {
        copy[k] = v
    }
    return copy
}

// File: automation-kit/validation/validation.go
package validation

import (
    "fmt"
    "regexp"
    "strings"
)

// FOUNDATIONAL PACKAGE: Provides validation utilities
// Key principles:
// 1. Pure functions, no state
// 2. Composable validation rules
// 3. Clear error messages
// 4. No dependencies on other packages

// Rule represents a validation rule
type Rule func(value interface{}) error

// Required validates that a value is not empty
func Required(value interface{}) error {
    if value == nil {
        return fmt.Errorf("value is required")
    }

    switch v := value.(type) {
    case string:
        if strings.TrimSpace(v) == "" {
            return fmt.Errorf("value is required")
        }
    case []interface{}:
        if len(v) == 0 {
            return fmt.Errorf("value is required")
        }
    }

    return nil
}

// MinLength validates minimum string length
func MinLength(min int) Rule {
    return func(value interface{}) error {
        str, ok := value.(string)
        if !ok {
            return fmt.Errorf("value must be a string")
        }

        if len(str) < min {
            return fmt.Errorf("value must be at least %d characters", min)
        }

        return nil
    }
}

// MaxLength validates maximum string length
func MaxLength(max int) Rule {
    return func(value interface{}) error {
        str, ok := value.(string)
        if !ok {
            return fmt.Errorf("value must be a string")
        }

        if len(str) > max {
            return fmt.Errorf("value must be at most %d characters", max)
        }

        return nil
    }
}

// Email validates email format
func Email(value interface{}) error {
    str, ok := value.(string)
    if !ok {
        return fmt.Errorf("value must be a string")
    }

    emailRegex := regexp.MustCompile(`^[a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\.[a-zA-Z]{2,}$`)
    if !emailRegex.MatchString(str) {
        return fmt.Errorf("value must be a valid email address")
    }

    return nil
}

// Validate applies multiple rules to a value
func Validate(value interface{}, rules ...Rule) error {
    for _, rule := range rules {
        if err := rule(value); err != nil {
            return err
        }
    }
    return nil
}

// File: automation-kit/example/main.go
package main

import (
    "automation-kit/config"
    "automation-kit/logger"
    "automation-kit/metrics"
    "automation-kit/validation"
    "fmt"
    "os"
    "time"
)

// DEMONSTRATION: Using kit packages together
// Shows how foundational packages work together without coupling

func main() {
    fmt.Println("=== Kit Project Demonstration ===")

    // Initialize foundational components
    cfg := config.New()
    cfg.LoadFromEnv("AUTOMATION_")

    log := logger.New(logger.INFO, os.Stdout, "[KIT-DEMO] ")
    metricsCollector := metrics.New()

    log.Info("Kit demonstration starting")

    // Configuration usage
    fmt.Println("\n--- Configuration ---")
    serviceName := cfg.String("SERVICE_NAME", "automation-service")
    port := cfg.Int("PORT", 8080)
    timeout := cfg.Duration("TIMEOUT", 30*time.Second)
    debug := cfg.Bool("DEBUG", false)

    fmt.Printf("Service: %s\n", serviceName)
    fmt.Printf("Port: %d\n", port)
    fmt.Printf("Timeout: %v\n", timeout)
    fmt.Printf("Debug: %t\n", debug)

    // Metrics usage
    fmt.Println("\n--- Metrics ---")
    metricsCollector.Record("startup_time", 1.5, map[string]string{
        "service": serviceName,
        "version": "1.0.0",
    })

    metricsCollector.Record("config_loaded", 1, map[string]string{
        "source": "environment",
    })

    allMetrics := metricsCollector.All()
    fmt.Printf("Recorded %d metrics:\n", len(allMetrics))
    for _, metric := range allMetrics {
        fmt.Printf("  %s: %.2f %v\n", metric.Name, metric.Value, metric.Tags)
    }

    // Validation usage
    fmt.Println("\n--- Validation ---")

    testData := []struct {
        name  string
        value interface{}
        rules []validation.Rule
    }{
        {
            name:  "Service Name",
            value: serviceName,
            rules: []validation.Rule{
                validation.Required,
                validation.MinLength(3),
                validation.MaxLength(50),
            },
        },
        {
            name:  "Email",
            value: "admin@company.com",
            rules: []validation.Rule{
                validation.Required,
                validation.Email,
            },
        },
        {
            name:  "Invalid Email",
            value: "not-an-email",
            rules: []validation.Rule{
                validation.Required,
                validation.Email,
            },
        },
    }

    for _, test := range testData {
        fmt.Printf("Validating %s: ", test.name)
        if err := validation.Validate(test.value, test.rules...); err != nil {
            fmt.Printf("❌ %v\n", err)
        } else {
            fmt.Printf("✅… Valid\n")
        }
    }

    log.Info("Kit demonstration completed")

    fmt.Println("\n=== Kit Project Principles ===")
    fmt.Println("✅… FOUNDATIONAL: Packages provide core capabilities")
    fmt.Println("✅… DECOUPLED: No dependencies between kit packages")
    fmt.Println("✅… REUSABLE: Can be used across multiple applications")
    fmt.Println("✅… FOCUSED: Each package has a single, clear purpose")
    fmt.Println("✅… NO POLICY: Packages don't make decisions about usage")
    fmt.Println("✅… PRIMITIVE: Low-level building blocks for applications")
}
```

**Hands-on Exercise 2: Application Project Structure (cmd, internal, platform)**:

```go
// Demonstrating application project structure for automation system
// Project structure:
// automation-app/
// ├── cmd/
// │   ├── server/          # Web server binary
// │   ├── worker/          # Background worker binary
// │   └── cli/             # CLI tool binary
// ├── internal/
// │   ├── task/            # Business logic - task domain
// │   ├── user/            # Business logic - user domain
// │   └── workflow/        # Business logic - workflow domain
// ├── internal/platform/
// │   ├── database/        # Database support
// │   ├── queue/           # Queue support
// │   └── web/             # Web framework support
// └── vendor/              # Dependencies (managed by go mod)

// File: cmd/server/main.go
package main

import (
    "automation-app/internal/platform/database"
    "automation-app/internal/platform/web"
    "automation-app/internal/task"
    "automation-app/internal/user"
    "context"
    "fmt"
    "log"
    "net/http"
    "os"
    "os/signal"
    "syscall"
    "time"
)

// APPLICATION LAYER: Server binary entry point
// Responsibilities:
// 1. Startup and shutdown logic
// 2. Dependency wiring
// 3. HTTP routing and middleware
// 4. Signal handling
// 5. Configuration loading

func main() {
    fmt.Println("=== Automation Server Starting ===")

    // Initialize platform dependencies
    db, err := database.New("memory://")
    if err != nil {
        log.Fatalf("Failed to initialize database: %v", err)
    }
    defer db.Close()

    // Initialize business services
    taskService := task.NewService(db)
    userService := user.NewService(db)

    // Initialize web framework
    webFramework := web.New()

    // Setup routes - this is application-specific logic
    setupRoutes(webFramework, taskService, userService)

    // Create HTTP server
    server := &http.Server{
        Addr:         ":8080",
        Handler:      webFramework,
        ReadTimeout:  15 * time.Second,
        WriteTimeout: 15 * time.Second,
    }

    // Start server in goroutine
    go func() {
        fmt.Println("Server listening on :8080")
        if err := server.ListenAndServe(); err != nil && err != http.ErrServerClosed {
            log.Fatalf("Server failed: %v", err)
        }
    }()

    // Wait for interrupt signal
    quit := make(chan os.Signal, 1)
    signal.Notify(quit, syscall.SIGINT, syscall.SIGTERM)
    <-quit

    fmt.Println("Server shutting down...")

    // Graceful shutdown
    ctx, cancel := context.WithTimeout(context.Background(), 30*time.Second)
    defer cancel()

    if err := server.Shutdown(ctx); err != nil {
        log.Fatalf("Server forced to shutdown: %v", err)
    }

    fmt.Println("Server stopped")
}

// setupRoutes configures HTTP routes - application-specific logic
func setupRoutes(w *web.Framework, taskSvc *task.Service, userSvc *user.Service) {
    // Task routes
    w.GET("/tasks", handleListTasks(taskSvc))
    w.POST("/tasks", handleCreateTask(taskSvc))
    w.GET("/tasks/:id", handleGetTask(taskSvc))
    w.PUT("/tasks/:id", handleUpdateTask(taskSvc))
    w.DELETE("/tasks/:id", handleDeleteTask(taskSvc))

    // User routes
    w.GET("/users", handleListUsers(userSvc))
    w.POST("/users", handleCreateUser(userSvc))
    w.GET("/users/:id", handleGetUser(userSvc))
}

// HTTP handlers - application layer, presentational logic
func handleListTasks(svc *task.Service) web.HandlerFunc {
    return func(c *web.Context) error {
        tasks, err := svc.List(c.Request.Context())
        if err != nil {
            return c.JSON(500, map[string]string{"error": err.Error()})
        }
        return c.JSON(200, tasks)
    }
}

func handleCreateTask(svc *task.Service) web.HandlerFunc {
    return func(c *web.Context) error {
        var req struct {
            Name        string `json:"name"`
            Description string `json:"description"`
        }

        if err := c.Bind(&req); err != nil {
            return c.JSON(400, map[string]string{"error": "invalid request"})
        }

        task, err := svc.Create(c.Request.Context(), req.Name, req.Description)
        if err != nil {
            return c.JSON(500, map[string]string{"error": err.Error()})
        }

        return c.JSON(201, task)
    }
}

func handleGetTask(svc *task.Service) web.HandlerFunc {
    return func(c *web.Context) error {
        id := c.Param("id")
        task, err := svc.GetByID(c.Request.Context(), id)
        if err != nil {
            return c.JSON(404, map[string]string{"error": "task not found"})
        }
        return c.JSON(200, task)
    }
}

func handleUpdateTask(svc *task.Service) web.HandlerFunc {
    return func(c *web.Context) error {
        // Implementation details...
        return c.JSON(200, map[string]string{"status": "updated"})
    }
}

func handleDeleteTask(svc *task.Service) web.HandlerFunc {
    return func(c *web.Context) error {
        // Implementation details...
        return c.JSON(200, map[string]string{"status": "deleted"})
    }
}

func handleListUsers(svc *user.Service) web.HandlerFunc {
    return func(c *web.Context) error {
        users, err := svc.List(c.Request.Context())
        if err != nil {
            return c.JSON(500, map[string]string{"error": err.Error()})
        }
        return c.JSON(200, users)
    }
}

func handleCreateUser(svc *user.Service) web.HandlerFunc {
    return func(c *web.Context) error {
        var req struct {
            Name  string `json:"name"`
            Email string `json:"email"`
        }

        if err := c.Bind(&req); err != nil {
            return c.JSON(400, map[string]string{"error": "invalid request"})
        }

        user, err := svc.Create(c.Request.Context(), req.Name, req.Email)
        if err != nil {
            return c.JSON(500, map[string]string{"error": err.Error()})
        }

        return c.JSON(201, user)
    }
}

func handleGetUser(svc *user.Service) web.HandlerFunc {
    return func(c *web.Context) error {
        id := c.Param("id")
        user, err := svc.GetByID(c.Request.Context(), id)
        if err != nil {
            return c.JSON(404, map[string]string{"error": "user not found"})
        }
        return c.JSON(200, user)
    }
}

// File: cmd/worker/main.go
package main

import (
    "automation-app/internal/platform/database"
    "automation-app/internal/platform/queue"
    "automation-app/internal/task"
    "context"
    "fmt"
    "log"
    "os"
    "os/signal"
    "syscall"
    "time"
)

// APPLICATION LAYER: Worker binary entry point
// Responsibilities:
// 1. Background job processing
// 2. Queue message handling
// 3. Task execution coordination
// 4. Error handling and retries

func main() {
    fmt.Println("=== Automation Worker Starting ===")

    // Initialize platform dependencies
    db, err := database.New("memory://")
    if err != nil {
        log.Fatalf("Failed to initialize database: %v", err)
    }
    defer db.Close()

    q, err := queue.New("memory://")
    if err != nil {
        log.Fatalf("Failed to initialize queue: %v", err)
    }
    defer q.Close()

    // Initialize business services
    taskService := task.NewService(db)

    // Create worker
    worker := &Worker{
        queue:       q,
        taskService: taskService,
        stopCh:      make(chan struct{}),
    }

    // Start worker
    go worker.Start()

    // Wait for interrupt signal
    quit := make(chan os.Signal, 1)
    signal.Notify(quit, syscall.SIGINT, syscall.SIGTERM)
    <-quit

    fmt.Println("Worker shutting down...")
    worker.Stop()
    fmt.Println("Worker stopped")
}

// Worker handles background task processing
type Worker struct {
    queue       *queue.Queue
    taskService *task.Service
    stopCh      chan struct{}
}

func (w *Worker) Start() {
    fmt.Println("Worker started, waiting for tasks...")

    for {
        select {
        case <-w.stopCh:
            return
        default:
            // Poll for messages
            msg, err := w.queue.Receive(context.Background(), 5*time.Second)
            if err != nil {
                continue // Timeout or error, continue polling
            }

            // Process message
            if err := w.processMessage(msg); err != nil {
                log.Printf("Failed to process message: %v", err)
            }
        }
    }
}

func (w *Worker) Stop() {
    close(w.stopCh)
}

func (w *Worker) processMessage(msg *queue.Message) error {
    fmt.Printf("Processing message: %s\n", msg.Body)

    // Parse message and execute task
    // This is application-specific logic
    ctx := context.Background()

    // Simulate task processing
    time.Sleep(100 * time.Millisecond)

    return nil
}

// File: internal/task/task.go
package task

import (
    "context"
    "fmt"
    "time"
)

// BUSINESS LAYER: Task domain logic
// Responsibilities:
// 1. Task business rules and validation
// 2. Task lifecycle management
// 3. Domain-specific operations
// 4. Business invariants enforcement

// Task represents a unit of work in the automation system
type Task struct {
    ID          string
    Name        string
    Description string
    Status      Status
    CreatedAt   time.Time
    UpdatedAt   time.Time
    CompletedAt *time.Time
}

// Status represents task execution status
type Status string

const (
    StatusPending   Status = "pending"
    StatusRunning   Status = "running"
    StatusCompleted Status = "completed"
    StatusFailed    Status = "failed"
    StatusCancelled Status = "cancelled"
)

// Repository defines data persistence interface
// Interface defined in business layer, implemented in platform layer
type Repository interface {
    Create(ctx context.Context, task *Task) error
    GetByID(ctx context.Context, id string) (*Task, error)
    Update(ctx context.Context, task *Task) error
    List(ctx context.Context) ([]*Task, error)
    Delete(ctx context.Context, id string) error
}

// Service provides task business operations
type Service struct {
    repo Repository
}

// NewService creates a new task service
func NewService(repo Repository) *Service {
    return &Service{
        repo: repo,
    }
}

// Create creates a new task with business validation
func (s *Service) Create(ctx context.Context, name, description string) (*Task, error) {
    // Business validation
    if name == "" {
        return nil, fmt.Errorf("task name is required")
    }

    if len(name) > 100 {
        return nil, fmt.Errorf("task name too long (max 100 characters)")
    }

    task := &Task{
        ID:          generateID(),
        Name:        name,
        Description: description,
        Status:      StatusPending,
        CreatedAt:   time.Now(),
        UpdatedAt:   time.Now(),
    }

    if err := s.repo.Create(ctx, task); err != nil {
        return nil, fmt.Errorf("failed to create task: %w", err)
    }

    return task, nil
}

// GetByID retrieves a task by ID
func (s *Service) GetByID(ctx context.Context, id string) (*Task, error) {
    if id == "" {
        return nil, fmt.Errorf("task ID is required")
    }

    return s.repo.GetByID(ctx, id)
}

// List retrieves all tasks
func (s *Service) List(ctx context.Context) ([]*Task, error) {
    return s.repo.List(ctx)
}

// Start starts task execution
func (s *Service) Start(ctx context.Context, id string) error {
    task, err := s.repo.GetByID(ctx, id)
    if err != nil {
        return err
    }

    // Business rule: can only start pending tasks
    if task.Status != StatusPending {
        return fmt.Errorf("cannot start task in status %s", task.Status)
    }

    task.Status = StatusRunning
    task.UpdatedAt = time.Now()

    return s.repo.Update(ctx, task)
}

// Complete marks task as completed
func (s *Service) Complete(ctx context.Context, id string) error {
    task, err := s.repo.GetByID(ctx, id)
    if err != nil {
        return err
    }

    // Business rule: can only complete running tasks
    if task.Status != StatusRunning {
        return fmt.Errorf("cannot complete task in status %s", task.Status)
    }

    now := time.Now()
    task.Status = StatusCompleted
    task.UpdatedAt = now
    task.CompletedAt = &now

    return s.repo.Update(ctx, task)
}

// generateID generates a unique task ID
func generateID() string {
    return fmt.Sprintf("task_%d", time.Now().UnixNano())
}

// File: internal/user/user.go
package user

import (
    "context"
    "fmt"
    "regexp"
    "time"
)

// BUSINESS LAYER: User domain logic
// Responsibilities:
// 1. User business rules and validation
// 2. User lifecycle management
// 3. Authentication and authorization logic
// 4. User-specific business operations

// User represents a system user
type User struct {
    ID        string
    Name      string
    Email     string
    Active    bool
    CreatedAt time.Time
    UpdatedAt time.Time
}

// Repository defines user data persistence interface
type Repository interface {
    Create(ctx context.Context, user *User) error
    GetByID(ctx context.Context, id string) (*User, error)
    GetByEmail(ctx context.Context, email string) (*User, error)
    Update(ctx context.Context, user *User) error
    List(ctx context.Context) ([]*User, error)
    Delete(ctx context.Context, id string) error
}

// Service provides user business operations
type Service struct {
    repo Repository
}

// NewService creates a new user service
func NewService(repo Repository) *Service {
    return &Service{
        repo: repo,
    }
}

// Create creates a new user with business validation
func (s *Service) Create(ctx context.Context, name, email string) (*User, error) {
    // Business validation
    if name == "" {
        return nil, fmt.Errorf("user name is required")
    }

    if email == "" {
        return nil, fmt.Errorf("user email is required")
    }

    if !isValidEmail(email) {
        return nil, fmt.Errorf("invalid email format")
    }

    // Business rule: email must be unique
    existing, err := s.repo.GetByEmail(ctx, email)
    if err == nil && existing != nil {
        return nil, fmt.Errorf("user with email %s already exists", email)
    }

    user := &User{
        ID:        generateUserID(),
        Name:      name,
        Email:     email,
        Active:    true,
        CreatedAt: time.Now(),
        UpdatedAt: time.Now(),
    }

    if err := s.repo.Create(ctx, user); err != nil {
        return nil, fmt.Errorf("failed to create user: %w", err)
    }

    return user, nil
}

// GetByID retrieves a user by ID
func (s *Service) GetByID(ctx context.Context, id string) (*User, error) {
    if id == "" {
        return nil, fmt.Errorf("user ID is required")
    }

    return s.repo.GetByID(ctx, id)
}

// List retrieves all users
func (s *Service) List(ctx context.Context) ([]*User, error) {
    return s.repo.List(ctx)
}

// Deactivate deactivates a user
func (s *Service) Deactivate(ctx context.Context, id string) error {
    user, err := s.repo.GetByID(ctx, id)
    if err != nil {
        return err
    }

    user.Active = false
    user.UpdatedAt = time.Now()

    return s.repo.Update(ctx, user)
}

// isValidEmail validates email format
func isValidEmail(email string) bool {
    emailRegex := regexp.MustCompile(`^[a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\.[a-zA-Z]{2,}$`)
    return emailRegex.MatchString(email)
}

// generateUserID generates a unique user ID
func generateUserID() string {
    return fmt.Sprintf("user_%d", time.Now().UnixNano())
}

// File: internal/platform/database/database.go
package database

import (
    "automation-app/internal/task"
    "automation-app/internal/user"
    "context"
    "fmt"
    "sync"
)

// PLATFORM LAYER: Database support
// Responsibilities:
// 1. Implement business layer repository interfaces
// 2. Handle data persistence details
// 3. Provide database abstraction
// 4. Manage connections and transactions

// Database provides database operations
type Database struct {
    taskRepo *TaskRepository
    userRepo *UserRepository
}

// New creates a new database instance
func New(connectionString string) (*Database, error) {
    // In real implementation, would parse connection string
    // and create appropriate database connections

    return &Database{
        taskRepo: NewTaskRepository(),
        userRepo: NewUserRepository(),
    }, nil
}

// Close closes database connections
func (db *Database) Close() error {
    // In real implementation, would close database connections
    return nil
}

// TaskRepository returns the task repository
func (db *Database) TaskRepository() task.Repository {
    return db.taskRepo
}

// UserRepository returns the user repository
func (db *Database) UserRepository() user.Repository {
    return db.userRepo
}

// TaskRepository implements task.Repository interface
type TaskRepository struct {
    mu    sync.RWMutex
    tasks map[string]*task.Task
}

// NewTaskRepository creates a new task repository
func NewTaskRepository() *TaskRepository {
    return &TaskRepository{
        tasks: make(map[string]*task.Task),
    }
}

func (r *TaskRepository) Create(ctx context.Context, t *task.Task) error {
    r.mu.Lock()
    defer r.mu.Unlock()

    if _, exists := r.tasks[t.ID]; exists {
        return fmt.Errorf("task with ID %s already exists", t.ID)
    }

    // In real implementation, would execute SQL INSERT
    r.tasks[t.ID] = t
    return nil
}

func (r *TaskRepository) GetByID(ctx context.Context, id string) (*task.Task, error) {
    r.mu.RLock()
    defer r.mu.RUnlock()

    t, exists := r.tasks[id]
    if !exists {
        return nil, fmt.Errorf("task with ID %s not found", id)
    }

    // In real implementation, would execute SQL SELECT
    return t, nil
}

func (r *TaskRepository) Update(ctx context.Context, t *task.Task) error {
    r.mu.Lock()
    defer r.mu.Unlock()

    if _, exists := r.tasks[t.ID]; !exists {
        return fmt.Errorf("task with ID %s not found", t.ID)
    }

    // In real implementation, would execute SQL UPDATE
    r.tasks[t.ID] = t
    return nil
}

func (r *TaskRepository) List(ctx context.Context) ([]*task.Task, error) {
    r.mu.RLock()
    defer r.mu.RUnlock()

    tasks := make([]*task.Task, 0, len(r.tasks))
    for _, t := range r.tasks {
        tasks = append(tasks, t)
    }

    // In real implementation, would execute SQL SELECT with pagination
    return tasks, nil
}

func (r *TaskRepository) Delete(ctx context.Context, id string) error {
    r.mu.Lock()
    defer r.mu.Unlock()

    if _, exists := r.tasks[id]; !exists {
        return fmt.Errorf("task with ID %s not found", id)
    }

    // In real implementation, would execute SQL DELETE
    delete(r.tasks, id)
    return nil
}

// File: internal/platform/web/web.go
package web

import (
    "encoding/json"
    "net/http"
)

// PLATFORM LAYER: Web framework support
// Responsibilities:
// 1. HTTP request/response handling
// 2. Routing and middleware
// 3. JSON serialization/deserialization
// 4. HTTP-specific concerns

// Framework provides web framework capabilities
type Framework struct {
    mux *http.ServeMux
}

// New creates a new web framework instance
func New() *Framework {
    return &Framework{
        mux: http.NewServeMux(),
    }
}

// ServeHTTP implements http.Handler interface
func (f *Framework) ServeHTTP(w http.ResponseWriter, r *http.Request) {
    f.mux.ServeHTTP(w, r)
}

// GET registers a GET route
func (f *Framework) GET(pattern string, handler HandlerFunc) {
    f.mux.HandleFunc(pattern, func(w http.ResponseWriter, r *http.Request) {
        if r.Method != http.MethodGet {
            http.Error(w, "Method not allowed", http.StatusMethodNotAllowed)
            return
        }

        ctx := &Context{
            Request:  r,
            Response: w,
        }

        if err := handler(ctx); err != nil {
            http.Error(w, err.Error(), http.StatusInternalServerError)
        }
    })
}

// POST registers a POST route
func (f *Framework) POST(pattern string, handler HandlerFunc) {
    f.mux.HandleFunc(pattern, func(w http.ResponseWriter, r *http.Request) {
        if r.Method != http.MethodPost {
            http.Error(w, "Method not allowed", http.StatusMethodNotAllowed)
            return
        }

        ctx := &Context{
            Request:  r,
            Response: w,
        }

        if err := handler(ctx); err != nil {
            http.Error(w, err.Error(), http.StatusInternalServerError)
        }
    })
}

// PUT registers a PUT route
func (f *Framework) PUT(pattern string, handler HandlerFunc) {
    f.mux.HandleFunc(pattern, func(w http.ResponseWriter, r *http.Request) {
        if r.Method != http.MethodPut {
            http.Error(w, "Method not allowed", http.StatusMethodNotAllowed)
            return
        }

        ctx := &Context{
            Request:  r,
            Response: w,
        }

        if err := handler(ctx); err != nil {
            http.Error(w, err.Error(), http.StatusInternalServerError)
        }
    })
}

// DELETE registers a DELETE route
func (f *Framework) DELETE(pattern string, handler HandlerFunc) {
    f.mux.HandleFunc(pattern, func(w http.ResponseWriter, r *http.Request) {
        if r.Method != http.MethodDelete {
            http.Error(w, "Method not allowed", http.StatusMethodNotAllowed)
            return
        }

        ctx := &Context{
            Request:  r,
            Response: w,
        }

        if err := handler(ctx); err != nil {
            http.Error(w, err.Error(), http.StatusInternalServerError)
        }
    })
}

// HandlerFunc defines the signature for HTTP handlers
type HandlerFunc func(*Context) error

// Context provides request/response context
type Context struct {
    Request  *http.Request
    Response http.ResponseWriter
}

// JSON sends a JSON response
func (c *Context) JSON(status int, data interface{}) error {
    c.Response.Header().Set("Content-Type", "application/json")
    c.Response.WriteHeader(status)
    return json.NewEncoder(c.Response).Encode(data)
}

// Bind binds request body to a struct
func (c *Context) Bind(v interface{}) error {
    return json.NewDecoder(c.Request.Body).Decode(v)
}

// Param extracts URL parameter (simplified implementation)
func (c *Context) Param(key string) string {
    // In real implementation, would extract from URL path
    // This is a simplified version
    return c.Request.URL.Query().Get(key)
}

// File: internal/platform/queue/queue.go
package queue

import (
    "context"
    "fmt"
    "sync"
    "time"
)

// PLATFORM LAYER: Queue support
// Responsibilities:
// 1. Message queue operations
// 2. Asynchronous message handling
// 3. Queue abstraction
// 4. Message serialization/deserialization

// Queue provides message queue operations
type Queue struct {
    mu       sync.RWMutex
    messages chan *Message
}

// Message represents a queue message
type Message struct {
    ID   string
    Body string
    Metadata map[string]string
}

// New creates a new queue instance
func New(connectionString string) (*Queue, error) {
    // In real implementation, would connect to actual message queue
    return &Queue{
        messages: make(chan *Message, 100),
    }, nil
}

// Close closes the queue
func (q *Queue) Close() error {
    close(q.messages)
    return nil
}

// Send sends a message to the queue
func (q *Queue) Send(ctx context.Context, message *Message) error {
    select {
    case q.messages <- message:
        return nil
    case <-ctx.Done():
        return ctx.Err()
    default:
        return fmt.Errorf("queue is full")
    }
}

// Receive receives a message from the queue
func (q *Queue) Receive(ctx context.Context, timeout time.Duration) (*Message, error) {
    select {
    case message := <-q.messages:
        return message, nil
    case <-time.After(timeout):
        return nil, fmt.Errorf("receive timeout")
    case <-ctx.Done():
        return nil, ctx.Err()
    }
}
```

**Hands-on Exercise 3: Package Validation and Dependency Management**:

```go
// Demonstrating package validation and dependency management principles
// This exercise shows how to validate package design and manage dependencies

// File: tools/pkgvalidator/main.go
package main

import (
    "fmt"
    "go/ast"
    "go/parser"
    "go/token"
    "os"
    "path/filepath"
    "strings"
)

// PACKAGE VALIDATION TOOL
// Validates package design according to package-oriented design principles

// PackageInfo holds information about a package
type PackageInfo struct {
    Name         string
    Path         string
    Files        []string
    Imports      []string
    Exports      []string
    Dependencies map[string]bool
}

// ValidationRule represents a package validation rule
type ValidationRule struct {
    Name        string
    Description string
    Check       func(*PackageInfo) []string
}

// PackageValidator validates package design
type PackageValidator struct {
    rules []ValidationRule
}

// NewPackageValidator creates a new package validator
func NewPackageValidator() *PackageValidator {
    validator := &PackageValidator{}
    validator.addDefaultRules()
    return validator
}

// addDefaultRules adds default validation rules
func (pv *PackageValidator) addDefaultRules() {
    pv.rules = []ValidationRule{
        {
            Name:        "NoUtilsPackage",
            Description: "Packages should not be named 'utils', 'helpers', or 'common'",
            Check:       pv.checkNoUtilsPackage,
        },
        {
            Name:        "NoModelsPackage",
            Description: "Packages should not be named 'models' or 'types'",
            Check:       pv.checkNoModelsPackage,
        },
        {
            Name:        "SinglePurpose",
            Description: "Package should have a single, clear purpose",
            Check:       pv.checkSinglePurpose,
        },
        {
            Name:        "NoCircularDependencies",
            Description: "Packages should not have circular dependencies",
            Check:       pv.checkNoCircularDependencies,
        },
        {
            Name:        "InterfaceSegregation",
            Description: "Interfaces should be small and focused",
            Check:       pv.checkInterfaceSegregation,
        },
    }
}

// Validate validates a package
func (pv *PackageValidator) Validate(packagePath string) ([]string, error) {
    pkgInfo, err := pv.analyzePackage(packagePath)
    if err != nil {
        return nil, fmt.Errorf("failed to analyze package: %w", err)
    }

    var violations []string

    for _, rule := range pv.rules {
        ruleViolations := rule.Check(pkgInfo)
        for _, violation := range ruleViolations {
            violations = append(violations, fmt.Sprintf("[%s] %s", rule.Name, violation))
        }
    }

    return violations, nil
}

// analyzePackage analyzes a Go package
func (pv *PackageValidator) analyzePackage(packagePath string) (*PackageInfo, error) {
    fset := token.NewFileSet()

    pkgs, err := parser.ParseDir(fset, packagePath, nil, parser.ParseComments)
    if err != nil {
        return nil, err
    }

    // For simplicity, take the first package
    var pkg *ast.Package
    for _, p := range pkgs {
        pkg = p
        break
    }

    if pkg == nil {
        return nil, fmt.Errorf("no package found in %s", packagePath)
    }

    pkgInfo := &PackageInfo{
        Name:         pkg.Name,
        Path:         packagePath,
        Files:        make([]string, 0),
        Imports:      make([]string, 0),
        Exports:      make([]string, 0),
        Dependencies: make(map[string]bool),
    }

    // Analyze files
    for filename, file := range pkg.Files {
        pkgInfo.Files = append(pkgInfo.Files, filename)

        // Collect imports
        for _, imp := range file.Imports {
            importPath := strings.Trim(imp.Path.Value, `"`)
            pkgInfo.Imports = append(pkgInfo.Imports, importPath)
            pkgInfo.Dependencies[importPath] = true
        }

        // Collect exports (simplified)
        for _, decl := range file.Decls {
            switch d := decl.(type) {
            case *ast.FuncDecl:
                if d.Name.IsExported() {
                    pkgInfo.Exports = append(pkgInfo.Exports, "func:"+d.Name.Name)
                }
            case *ast.GenDecl:
                for _, spec := range d.Specs {
                    switch s := spec.(type) {
                    case *ast.TypeSpec:
                        if s.Name.IsExported() {
                            pkgInfo.Exports = append(pkgInfo.Exports, "type:"+s.Name.Name)
                        }
                    case *ast.ValueSpec:
                        for _, name := range s.Names {
                            if name.IsExported() {
                                pkgInfo.Exports = append(pkgInfo.Exports, "var:"+name.Name)
                            }
                        }
                    }
                }
            }
        }
    }

    return pkgInfo, nil
}

// Validation rule implementations
func (pv *PackageValidator) checkNoUtilsPackage(pkg *PackageInfo) []string {
    badNames := []string{"utils", "util", "helpers", "helper", "common"}

    for _, badName := range badNames {
        if strings.ToLower(pkg.Name) == badName {
            return []string{fmt.Sprintf("Package name '%s' is too generic. Use a name that describes what the package provides.", pkg.Name)}
        }
    }

    return nil
}

func (pv *PackageValidator) checkNoModelsPackage(pkg *PackageInfo) []string {
    badNames := []string{"models", "model", "types", "entities", "entity"}

    for _, badName := range badNames {
        if strings.ToLower(pkg.Name) == badName {
            return []string{fmt.Sprintf("Package name '%s' suggests a data container. Consider organizing by business capability instead.", pkg.Name)}
        }
    }

    return nil
}

func (pv *PackageValidator) checkSinglePurpose(pkg *PackageInfo) []string {
    // Simplified check: if package has too many different types of exports
    funcCount := 0
    typeCount := 0
    varCount := 0

    for _, export := range pkg.Exports {
        switch {
        case strings.HasPrefix(export, "func:"):
            funcCount++
        case strings.HasPrefix(export, "type:"):
            typeCount++
        case strings.HasPrefix(export, "var:"):
            varCount++
        }
    }

    var violations []string

    // If package exports too many different things, it might lack focus
    if funcCount > 20 {
        violations = append(violations, fmt.Sprintf("Package exports %d functions, consider splitting into smaller packages", funcCount))
    }

    if typeCount > 10 {
        violations = append(violations, fmt.Sprintf("Package exports %d types, consider if they all belong together", typeCount))
    }

    return violations
}

func (pv *PackageValidator) checkNoCircularDependencies(pkg *PackageInfo) []string {
    // Simplified check - in real implementation would build dependency graph
    var violations []string

    for dep := range pkg.Dependencies {
        // Check if dependency might create circular reference
        if strings.Contains(dep, pkg.Name) {
            violations = append(violations, fmt.Sprintf("Potential circular dependency with %s", dep))
        }
    }

    return violations
}

func (pv *PackageValidator) checkInterfaceSegregation(pkg *PackageInfo) []string {
    // Simplified check - would need more sophisticated AST analysis
    var violations []string

    interfaceCount := 0
    for _, export := range pkg.Exports {
        if strings.HasPrefix(export, "type:") && strings.Contains(export, "Interface") {
            interfaceCount++
        }
    }

    if interfaceCount > 5 {
        violations = append(violations, fmt.Sprintf("Package defines %d interfaces, consider if they're all necessary", interfaceCount))
    }

    return violations
}

// File: tools/depmanager/main.go
package main

import (
    "fmt"
    "go/build"
    "os"
    "path/filepath"
    "strings"
)

// DEPENDENCY MANAGEMENT TOOL
// Analyzes and manages package dependencies

// DependencyManager manages package dependencies
type DependencyManager struct {
    projectRoot string
    packages    map[string]*PackageDependency
}

// PackageDependency represents a package and its dependencies
type PackageDependency struct {
    Name         string
    Path         string
    Dependencies []string
    Dependents   []string
    Layer        string // kit, platform, business, application
}

// NewDependencyManager creates a new dependency manager
func NewDependencyManager(projectRoot string) *DependencyManager {
    return &DependencyManager{
        projectRoot: projectRoot,
        packages:    make(map[string]*PackageDependency),
    }
}

// AnalyzeProject analyzes project dependencies
func (dm *DependencyManager) AnalyzeProject() error {
    return filepath.Walk(dm.projectRoot, func(path string, info os.FileInfo, err error) error {
        if err != nil {
            return err
        }

        if !info.IsDir() {
            return nil
        }

        // Skip vendor and hidden directories
        if strings.Contains(path, "vendor") || strings.HasPrefix(filepath.Base(path), ".") {
            return filepath.SkipDir
        }

        // Check if directory contains Go files
        if dm.hasGoFiles(path) {
            pkg, err := dm.analyzePackage(path)
            if err != nil {
                return err
            }
            dm.packages[pkg.Path] = pkg
        }

        return nil
    })
}

// hasGoFiles checks if directory contains Go files
func (dm *DependencyManager) hasGoFiles(dir string) bool {
    files, err := os.ReadDir(dir)
    if err != nil {
        return false
    }

    for _, file := range files {
        if strings.HasSuffix(file.Name(), ".go") && !strings.HasSuffix(file.Name(), "_test.go") {
            return true
        }
    }

    return false
}

// analyzePackage analyzes a single package
func (dm *DependencyManager) analyzePackage(packagePath string) (*PackageDependency, error) {
    pkg, err := build.ImportDir(packagePath, 0)
    if err != nil {
        return nil, err
    }

    dependency := &PackageDependency{
        Name:         pkg.Name,
        Path:         packagePath,
        Dependencies: make([]string, 0),
        Dependents:   make([]string, 0),
        Layer:        dm.determineLayer(packagePath),
    }

    // Collect internal dependencies only
    for _, imp := range pkg.Imports {
        if strings.HasPrefix(imp, dm.getModuleName()) {
            dependency.Dependencies = append(dependency.Dependencies, imp)
        }
    }

    return dependency, nil
}

// determineLayer determines which layer a package belongs to
func (dm *DependencyManager) determineLayer(packagePath string) string {
    relPath, _ := filepath.Rel(dm.projectRoot, packagePath)

    switch {
    case strings.Contains(relPath, "cmd"):
        return "application"
    case strings.Contains(relPath, "internal"):
        if strings.Contains(relPath, "platform") {
            return "platform"
        }
        return "business"
    case strings.Contains(relPath, "pkg"):
        return "kit"
    default:
        return "unknown"
    }
}

// getModuleName gets the module name (simplified)
func (dm *DependencyManager) getModuleName() string {
    return "automation-app" // In real implementation, would read from go.mod
}

// ValidateDependencies validates dependency rules
func (dm *DependencyManager) ValidateDependencies() []string {
    var violations []string

    for _, pkg := range dm.packages {
        violations = append(violations, dm.validateLayerDependencies(pkg)...)
        violations = append(violations, dm.validateCircularDependencies(pkg)...)
    }

    return violations
}

// validateLayerDependencies validates layer dependency rules
func (dm *DependencyManager) validateLayerDependencies(pkg *PackageDependency) []string {
    var violations []string

    for _, dep := range pkg.Dependencies {
        depPkg, exists := dm.packages[dep]
        if !exists {
            continue
        }

        // Check layer dependency rules
        switch pkg.Layer {
        case "kit":
            // Kit packages should not depend on anything internal
            if depPkg.Layer != "kit" {
                violations = append(violations, fmt.Sprintf("Kit package %s depends on %s package %s", pkg.Name, depPkg.Layer, depPkg.Name))
            }

        case "business":
            // Business packages should not depend on platform or application
            if depPkg.Layer == "platform" || depPkg.Layer == "application" {
                violations = append(violations, fmt.Sprintf("Business package %s depends on %s package %s", pkg.Name, depPkg.Layer, depPkg.Name))
            }

        case "platform":
            // Platform packages should not depend on business or application
            if depPkg.Layer == "business" || depPkg.Layer == "application" {
                violations = append(violations, fmt.Sprintf("Platform package %s depends on %s package %s", pkg.Name, depPkg.Layer, depPkg.Name))
            }
        }
    }

    return violations
}

// validateCircularDependencies validates circular dependencies
func (dm *DependencyManager) validateCircularDependencies(pkg *PackageDependency) []string {
    visited := make(map[string]bool)
    path := make([]string, 0)

    if cycle := dm.findCycle(pkg.Path, visited, path); cycle != nil {
        return []string{fmt.Sprintf("Circular dependency detected: %s", strings.Join(cycle, " -> "))}
    }

    return nil
}

// findCycle finds circular dependencies using DFS
func (dm *DependencyManager) findCycle(pkgPath string, visited map[string]bool, path []string) []string {
    if visited[pkgPath] {
        // Found cycle
        for i, p := range path {
            if p == pkgPath {
                return append(path[i:], pkgPath)
            }
        }
        return nil
    }

    visited[pkgPath] = true
    path = append(path, pkgPath)

    pkg, exists := dm.packages[pkgPath]
    if !exists {
        return nil
    }

    for _, dep := range pkg.Dependencies {
        if cycle := dm.findCycle(dep, visited, path); cycle != nil {
            return cycle
        }
    }

    visited[pkgPath] = false
    return nil
}

// File: cmd/validator/main.go
package main

import (
    "fmt"
    "log"
    "os"
)

// DEMONSTRATION: Package validation and dependency management
func main() {
    fmt.Println("=== Package Validation and Dependency Management ===")

    if len(os.Args) < 2 {
        fmt.Println("Usage: validator <project-path>")
        os.Exit(1)
    }

    projectPath := os.Args[1]

    // Package validation
    fmt.Println("\n--- Package Validation ---")
    validator := NewPackageValidator()

    // Validate some example packages
    testPackages := []string{
        "internal/task",
        "internal/user",
        "internal/platform/database",
        "pkg/logger",
    }

    for _, pkg := range testPackages {
        fmt.Printf("\nValidating package: %s\n", pkg)
        violations, err := validator.Validate(filepath.Join(projectPath, pkg))
        if err != nil {
            fmt.Printf("  Error: %v\n", err)
            continue
        }

        if len(violations) == 0 {
            fmt.Printf("  ✅… No violations found\n")
        } else {
            fmt.Printf("  ❌ Found %d violations:\n", len(violations))
            for _, violation := range violations {
                fmt.Printf("    - %s\n", violation)
            }
        }
    }

    // Dependency analysis
    fmt.Println("\n--- Dependency Analysis ---")
    depManager := NewDependencyManager(projectPath)

    if err := depManager.AnalyzeProject(); err != nil {
        log.Printf("Failed to analyze project: %v", err)
        return
    }

    violations := depManager.ValidateDependencies()
    if len(violations) == 0 {
        fmt.Println("✅… No dependency violations found")
    } else {
        fmt.Printf("❌ Found %d dependency violations:\n", len(violations))
        for _, violation := range violations {
            fmt.Printf("  - %s\n", violation)
        }
    }

    // Print dependency summary
    fmt.Println("\n--- Package Summary ---")
    for path, pkg := range depManager.packages {
        fmt.Printf("Package: %s (Layer: %s)\n", pkg.Name, pkg.Layer)
        fmt.Printf("  Path: %s\n", path)
        if len(pkg.Dependencies) > 0 {
            fmt.Printf("  Dependencies: %v\n", pkg.Dependencies)
        }
        fmt.Println()
    }

    fmt.Println("=== Package-Oriented Design Validation Complete ===")
    fmt.Println("\n=== Key Validation Principles ===")
    fmt.Println("✅… NAMING: Avoid 'utils', 'models', 'helpers' package names")
    fmt.Println("✅… PURPOSE: Each package should have a single, clear purpose")
    fmt.Println("✅… DEPENDENCIES: Follow layer dependency rules")
    fmt.Println("✅… CIRCULAR: No circular dependencies between packages")
    fmt.Println("✅… INTERFACES: Keep interfaces small and focused")
    fmt.Println("✅… LAYERS: Kit -> Business -> Platform -> Application")
    fmt.Println("✅… DIRECTION: Dependencies flow inward, not outward")
}
```

**Prerequisites**: Module 26
