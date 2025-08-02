# Module 26: Package Design and Language Mechanics

**Duration**: 1 hour (45 min theory + 15 min hands-on)

**Learning Objectives**:

- Understand Go's package system and language mechanics
- Master package declaration, imports, and visibility rules
- Learn package initialization and dependency management
- Apply package design principles to automation systems
- Design clean, maintainable package structures

**Videos Covered**:

- 7.1 Topics (0:00:52)
- 7.2 Language Mechanics (0:08:32)
- 7.3 Design Guidelines (0:05:49)

**Key Concepts**:

- Package declaration and naming conventions
- Import paths and package resolution
- Exported vs unexported identifiers
- Package initialization order and init functions
- Circular dependency prevention
- Package-level variables and constants

**Hands-on Exercise 1: Package Language Mechanics and Basic Structure**:

```go
// Demonstrating Go package language mechanics in automation systems
// This exercise shows the fundamental mechanics of Go packages

// File: automation/metrics/metrics.go
package metrics

import (
    "fmt"
    "sync"
    "time"
)

// PART 1: Package Declaration and Naming
// Package name should be short, clear, and describe what it provides
// This package provides metrics collection capabilities

// PART 2: Exported vs Unexported Identifiers
// Exported identifiers start with uppercase letter - visible outside package
// Unexported identifiers start with lowercase letter - internal to package

// Exported constants - available to package users
const (
    DefaultBufferSize = 1000
    MaxMetricAge     = 24 * time.Hour
)

// Exported types - part of the package API
type MetricType string

const (
    Counter   MetricType = "counter"
    Gauge     MetricType = "gauge"
    Histogram MetricType = "histogram"
)

type Metric struct {
    Name      string
    Type      MetricType
    Value     float64
    Tags      map[string]string
    Timestamp time.Time
}

// Exported interface - defines the contract for metric collection
type Collector interface {
    Record(name string, value float64, tags map[string]string) error
    Get(name string) (*Metric, error)
    List() []*Metric
    Reset() error
}

// PART 3: Package-level Variables and State
// unexported package-level variables - internal state
var (
    defaultCollector *memoryCollector
    initOnce        sync.Once
)

// PART 4: Package Initialization with init()
// init() functions run automatically when package is imported
// They run in the order they appear in the source
func init() {
    fmt.Println("metrics package: init() called")
    // Initialize default collector
    defaultCollector = newMemoryCollector()
}

// PART 5: Unexported Implementation Types
// memoryCollector is unexported - implementation detail
type memoryCollector struct {
    mu      sync.RWMutex
    metrics map[string]*Metric
}

// unexported constructor - creates new collector instance
func newMemoryCollector() *memoryCollector {
    return &memoryCollector{
        metrics: make(map[string]*Metric),
    }
}

// Methods on unexported type - implementation details
func (mc *memoryCollector) Record(name string, value float64, tags map[string]string) error {
    mc.mu.Lock()
    defer mc.mu.Unlock()

    metric := &Metric{
        Name:      name,
        Type:      Counter, // Default type
        Value:     value,
        Tags:      tags,
        Timestamp: time.Now(),
    }

    mc.metrics[name] = metric
    return nil
}

func (mc *memoryCollector) Get(name string) (*Metric, error) {
    mc.mu.RLock()
    defer mc.mu.RUnlock()

    metric, exists := mc.metrics[name]
    if !exists {
        return nil, fmt.Errorf("metric %s not found", name)
    }

    return metric, nil
}

func (mc *memoryCollector) List() []*Metric {
    mc.mu.RLock()
    defer mc.mu.RUnlock()

    metrics := make([]*Metric, 0, len(mc.metrics))
    for _, metric := range mc.metrics {
        metrics = append(metrics, metric)
    }

    return metrics
}

func (mc *memoryCollector) Reset() error {
    mc.mu.Lock()
    defer mc.mu.Unlock()

    mc.metrics = make(map[string]*Metric)
    return nil
}

// PART 6: Exported Factory Functions
// These are the public API for creating and accessing collectors

// New creates a new metrics collector
func New() Collector {
    return newMemoryCollector()
}

// Default returns the package-level default collector
func Default() Collector {
    // Use sync.Once to ensure thread-safe initialization
    initOnce.Do(func() {
        if defaultCollector == nil {
            defaultCollector = newMemoryCollector()
        }
    })
    return defaultCollector
}

// PART 7: Convenience Functions
// These provide easy access to common operations

// Record records a metric using the default collector
func Record(name string, value float64, tags map[string]string) error {
    return Default().Record(name, value, tags)
}

// Get retrieves a metric using the default collector
func Get(name string) (*Metric, error) {
    return Default().Get(name)
}

// List returns all metrics from the default collector
func List() []*Metric {
    return Default().List()
}

// File: automation/config/config.go
package config

import (
    "encoding/json"
    "fmt"
    "os"
    "path/filepath"
    "time"
)

// PART 8: Demonstrating Package Dependencies and Imports
// This package depends on standard library packages only
// No circular dependencies allowed in Go

// Exported configuration structure
type Config struct {
    ServiceName   string        `json:"service_name"`
    Port         int           `json:"port"`
    DatabaseURL  string        `json:"database_url"`
    Timeout      time.Duration `json:"timeout"`
    MetricsEnabled bool         `json:"metrics_enabled"`
    LogLevel     string        `json:"log_level"`
}

// Package-level default configuration
var defaultConfig = &Config{
    ServiceName:    "AutomationService",
    Port:          8080,
    Timeout:       30 * time.Second,
    MetricsEnabled: true,
    LogLevel:      "INFO",
}

// Load loads configuration from a file
func Load(filename string) (*Config, error) {
    // Validate file extension
    if filepath.Ext(filename) != ".json" {
        return nil, fmt.Errorf("unsupported config file format: %s", filename)
    }

    data, err := os.ReadFile(filename)
    if err != nil {
        return nil, fmt.Errorf("failed to read config file %s: %w", filename, err)
    }

    var config Config
    if err := json.Unmarshal(data, &config); err != nil {
        return nil, fmt.Errorf("failed to parse config file %s: %w", filename, err)
    }

    return &config, nil
}

// Default returns a copy of the default configuration
func Default() *Config {
    // Return a copy to prevent external modification
    return &Config{
        ServiceName:    defaultConfig.ServiceName,
        Port:          defaultConfig.Port,
        DatabaseURL:   defaultConfig.DatabaseURL,
        Timeout:       defaultConfig.Timeout,
        MetricsEnabled: defaultConfig.MetricsEnabled,
        LogLevel:      defaultConfig.LogLevel,
    }
}

// Validate validates the configuration
func (c *Config) Validate() error {
    if c.ServiceName == "" {
        return fmt.Errorf("service_name cannot be empty")
    }

    if c.Port <= 0 || c.Port > 65535 {
        return fmt.Errorf("port must be between 1 and 65535, got %d", c.Port)
    }

    if c.Timeout <= 0 {
        return fmt.Errorf("timeout must be positive, got %v", c.Timeout)
    }

    validLogLevels := []string{"DEBUG", "INFO", "WARN", "ERROR"}
    for _, level := range validLogLevels {
        if c.LogLevel == level {
            return nil
        }
    }

    return fmt.Errorf("invalid log level %s, must be one of %v", c.LogLevel, validLogLevels)
}

// File: main.go
package main

import (
    "automation/config"
    "automation/metrics"
    "fmt"
    "log"
)

// PART 9: Package Import and Usage
// Demonstrates how packages are imported and used

func main() {
    fmt.Println("=== Package Language Mechanics Demonstration ===")

    // PART 10: Using Imported Packages
    fmt.Println("\n--- Configuration Package ---")

    // Use default configuration
    cfg := config.Default()
    fmt.Printf("Default config: %+v\n", cfg)

    // Validate configuration
    if err := cfg.Validate(); err != nil {
        log.Printf("Configuration validation failed: %v", err)
    } else {
        fmt.Println("Configuration is valid")
    }

    fmt.Println("\n--- Metrics Package ---")

    // Use package-level functions (convenience API)
    err := metrics.Record("requests_total", 100, map[string]string{
        "method": "GET",
        "status": "200",
    })
    if err != nil {
        log.Printf("Failed to record metric: %v", err)
    }

    err = metrics.Record("response_time", 0.25, map[string]string{
        "endpoint": "/api/tasks",
    })
    if err != nil {
        log.Printf("Failed to record metric: %v", err)
    }

    // Retrieve specific metric
    metric, err := metrics.Get("requests_total")
    if err != nil {
        log.Printf("Failed to get metric: %v", err)
    } else {
        fmt.Printf("Retrieved metric: %+v\n", metric)
    }

    // List all metrics
    allMetrics := metrics.List()
    fmt.Printf("Total metrics recorded: %d\n", len(allMetrics))
    for _, m := range allMetrics {
        fmt.Printf("  %s: %.2f (tags: %v)\n", m.Name, m.Value, m.Tags)
    }

    // Create custom collector
    fmt.Println("\n--- Custom Collector ---")
    customCollector := metrics.New()

    err = customCollector.Record("custom_metric", 42, map[string]string{
        "source": "custom",
    })
    if err != nil {
        log.Printf("Failed to record custom metric: %v", err)
    }

    customMetrics := customCollector.List()
    fmt.Printf("Custom collector metrics: %d\n", len(customMetrics))

    fmt.Println("\n=== Key Language Mechanics Demonstrated ===")
    fmt.Println("1. Package declaration and naming conventions")
    fmt.Println("2. Exported vs unexported identifiers")
    fmt.Println("3. Package-level variables and initialization")
    fmt.Println("4. init() function automatic execution")
    fmt.Println("5. Package imports and dependency management")
    fmt.Println("6. Interface definitions and implementations")
    fmt.Println("7. Factory functions and constructors")
    fmt.Println("8. Package API design patterns")
}
```

**Hands-on Exercise 2: Package Design Guidelines and Best Practices**:

```go
// Demonstrating package design guidelines: Purpose, Usability, and Portability
// This exercise shows good and bad package design patterns

// GOOD EXAMPLE: Purpose-driven package design
// File: automation/scheduler/scheduler.go
package scheduler

import (
    "context"
    "fmt"
    "sync"
    "time"
)

// GUIDELINE 1: PURPOSE - Package must provide, not contain
// This package provides task scheduling capabilities
// Clear purpose: "I schedule and manage task execution"

// Job represents a schedulable unit of work
type Job struct {
    ID          string
    Name        string
    Schedule    string // cron-like schedule
    Handler     JobHandler
    NextRun     time.Time
    LastRun     *time.Time
    Enabled     bool
}

// JobHandler defines what a job does when executed
type JobHandler interface {
    Execute(ctx context.Context) error
    Description() string
}

// Scheduler manages job scheduling and execution
type Scheduler struct {
    mu       sync.RWMutex
    jobs     map[string]*Job
    running  bool
    stopCh   chan struct{}
    doneCh   chan struct{}
}

// New creates a new scheduler instance
func New() *Scheduler {
    return &Scheduler{
        jobs:   make(map[string]*Job),
        stopCh: make(chan struct{}),
        doneCh: make(chan struct{}),
    }
}

// AddJob adds a job to the scheduler
func (s *Scheduler) AddJob(job *Job) error {
    if job.ID == "" {
        return fmt.Errorf("job ID cannot be empty")
    }

    s.mu.Lock()
    defer s.mu.Unlock()

    if _, exists := s.jobs[job.ID]; exists {
        return fmt.Errorf("job with ID %s already exists", job.ID)
    }

    s.jobs[job.ID] = job
    return nil
}

// Start begins job scheduling
func (s *Scheduler) Start(ctx context.Context) error {
    s.mu.Lock()
    if s.running {
        s.mu.Unlock()
        return fmt.Errorf("scheduler is already running")
    }
    s.running = true
    s.mu.Unlock()

    go s.run(ctx)
    return nil
}

// Stop stops the scheduler
func (s *Scheduler) Stop() error {
    s.mu.Lock()
    if !s.running {
        s.mu.Unlock()
        return fmt.Errorf("scheduler is not running")
    }
    s.mu.Unlock()

    close(s.stopCh)
    <-s.doneCh
    return nil
}

func (s *Scheduler) run(ctx context.Context) {
    defer close(s.doneCh)

    ticker := time.NewTicker(1 * time.Minute)
    defer ticker.Stop()

    for {
        select {
        case <-ctx.Done():
            return
        case <-s.stopCh:
            return
        case <-ticker.C:
            s.executeReadyJobs(ctx)
        }
    }
}

func (s *Scheduler) executeReadyJobs(ctx context.Context) {
    s.mu.RLock()
    jobs := make([]*Job, 0, len(s.jobs))
    for _, job := range s.jobs {
        if job.Enabled && time.Now().After(job.NextRun) {
            jobs = append(jobs, job)
        }
    }
    s.mu.RUnlock()

    for _, job := range jobs {
        go func(j *Job) {
            if err := j.Handler.Execute(ctx); err != nil {
                fmt.Printf("Job %s failed: %v\n", j.ID, err)
            } else {
                fmt.Printf("Job %s completed successfully\n", j.ID)
            }

            // Update last run time
            s.mu.Lock()
            now := time.Now()
            j.LastRun = &now
            j.NextRun = now.Add(1 * time.Hour) // Simple: run every hour
            s.mu.Unlock()
        }(job)
    }
}

// GOOD EXAMPLE: Notification package with clear purpose
// File: automation/notification/notification.go
package notification

import (
    "context"
    "fmt"
)

// GUIDELINE 2: USABILITY - Intuitive, simple, hard to misuse
// This package provides notification sending capabilities

// Message represents a notification message
type Message struct {
    To      string
    Subject string
    Body    string
    Type    MessageType
}

type MessageType string

const (
    Email MessageType = "email"
    SMS   MessageType = "sms"
    Slack MessageType = "slack"
)

// Sender defines how to send notifications
type Sender interface {
    Send(ctx context.Context, message *Message) error
    SupportsType(msgType MessageType) bool
}

// Service manages notification sending
type Service struct {
    senders map[MessageType]Sender
}

// NewService creates a new notification service
func NewService() *Service {
    return &Service{
        senders: make(map[MessageType]Sender),
    }
}

// RegisterSender registers a sender for a message type
func (s *Service) RegisterSender(msgType MessageType, sender Sender) {
    s.senders[msgType] = sender
}

// Send sends a notification message
func (s *Service) Send(ctx context.Context, message *Message) error {
    sender, exists := s.senders[message.Type]
    if !exists {
        return fmt.Errorf("no sender registered for message type %s", message.Type)
    }

    if !sender.SupportsType(message.Type) {
        return fmt.Errorf("sender does not support message type %s", message.Type)
    }

    return sender.Send(ctx, message)
}

// Example concrete sender implementation
type EmailSender struct {
    smtpHost string
    port     int
}

func NewEmailSender(smtpHost string, port int) *EmailSender {
    return &EmailSender{
        smtpHost: smtpHost,
        port:     port,
    }
}

func (es *EmailSender) Send(ctx context.Context, message *Message) error {
    if message.Type != Email {
        return fmt.Errorf("email sender cannot send %s messages", message.Type)
    }

    fmt.Printf("EMAIL: Sending to %s via %s:%d\n", message.To, es.smtpHost, es.port)
    fmt.Printf("  Subject: %s\n", message.Subject)
    fmt.Printf("  Body: %s\n", message.Body)

    return nil
}

func (es *EmailSender) SupportsType(msgType MessageType) bool {
    return msgType == Email
}

// BAD EXAMPLE: Anti-patterns to avoid
// File: automation/models/models.go (DON'T DO THIS!)
package models

import "time"

// ANTI-PATTERN 1: "models" package - contains instead of provides
// This is a dumping ground for types - violates purpose guideline
// Problem: Single point of dependency, cascading changes

// User represents a user in the system
type User struct {
    ID       string    `json:"id"`
    Name     string    `json:"name"`
    Email    string    `json:"email"`
    Created  time.Time `json:"created"`
}

// Task represents a task in the system
type Task struct {
    ID          string    `json:"id"`
    Name        string    `json:"name"`
    Description string    `json:"description"`
    Status      string    `json:"status"`
    AssignedTo  string    `json:"assigned_to"`
    Created     time.Time `json:"created"`
}

// Config represents system configuration
type Config struct {
    DatabaseURL string `json:"database_url"`
    Port        int    `json:"port"`
}

// PROBLEM: Any change to these types affects ALL packages that import models
// This creates tight coupling and makes the system fragile

// File: automation/utils/utils.go (DON'T DO THIS!)
package utils

import (
    "fmt"
    "strings"
    "time"
)

// ANTI-PATTERN 2: "utils" package - generic dumping ground
// Problem: No clear purpose, becomes a junk drawer

// StringHelper contains string utility functions
type StringHelper struct{}

// ToUpper converts string to uppercase
func (sh *StringHelper) ToUpper(s string) string {
    return strings.ToUpper(s)
}

// TimeHelper contains time utility functions
type TimeHelper struct{}

// FormatTime formats time to string
func (th *TimeHelper) FormatTime(t time.Time) string {
    return t.Format("2006-01-02 15:04:05")
}

// MathHelper contains math utility functions
type MathHelper struct{}

// Max returns the maximum of two integers
func (mh *MathHelper) Max(a, b int) int {
    if a > b {
        return a
    }
    return b
}

// PROBLEM: Unrelated functionality grouped together
// No clear API contract, hard to understand purpose

// GOOD EXAMPLE: Refactored with clear purpose
// File: automation/textprocessor/processor.go
package textprocessor

import "strings"

// GUIDELINE 3: PORTABILITY - Minimal coupling, reusable
// This package provides text processing capabilities

// Processor handles text transformations
type Processor struct {
    options Options
}

// Options configures text processing behavior
type Options struct {
    CaseSensitive bool
    TrimSpaces   bool
}

// New creates a new text processor
func New(opts Options) *Processor {
    return &Processor{options: opts}
}

// Transform applies transformations to text
func (p *Processor) Transform(text string) string {
    result := text

    if p.options.TrimSpaces {
        result = strings.TrimSpace(result)
    }

    if !p.options.CaseSensitive {
        result = strings.ToLower(result)
    }

    return result
}

// Normalize normalizes text according to processor options
func (p *Processor) Normalize(text string) string {
    return p.Transform(text)
}

// File: automation/timeutil/timeutil.go
package timeutil

import (
    "fmt"
    "time"
)

// GOOD: Focused package with clear purpose - time utilities
// Provides time-related functionality, doesn't contain random utilities

// Formatter handles time formatting operations
type Formatter struct {
    layout string
}

// NewFormatter creates a new time formatter
func NewFormatter(layout string) *Formatter {
    return &Formatter{layout: layout}
}

// Format formats a time according to the formatter's layout
func (f *Formatter) Format(t time.Time) string {
    return t.Format(f.layout)
}

// Common formatters as package-level convenience
var (
    ISO8601    = NewFormatter(time.RFC3339)
    DateOnly   = NewFormatter("2006-01-02")
    TimeOnly   = NewFormatter("15:04:05")
    DateTime   = NewFormatter("2006-01-02 15:04:05")
)

// IsBusinessDay checks if a given time falls on a business day
func IsBusinessDay(t time.Time) bool {
    weekday := t.Weekday()
    return weekday != time.Saturday && weekday != time.Sunday
}

// BusinessDaysUntil calculates business days between two dates
func BusinessDaysUntil(from, to time.Time) int {
    if to.Before(from) {
        return 0
    }

    days := 0
    current := from

    for current.Before(to) {
        if IsBusinessDay(current) {
            days++
        }
        current = current.AddDate(0, 0, 1)
    }

    return days
}

// File: main.go
package main

import (
    "automation/notification"
    "automation/scheduler"
    "automation/textprocessor"
    "automation/timeutil"
    "context"
    "fmt"
    "time"
)

// Example job handler implementation
type DataCleanupJob struct {
    name string
}

func (dcj *DataCleanupJob) Execute(ctx context.Context) error {
    fmt.Printf("Executing data cleanup job: %s\n", dcj.name)
    // Simulate work
    time.Sleep(100 * time.Millisecond)
    return nil
}

func (dcj *DataCleanupJob) Description() string {
    return fmt.Sprintf("Data cleanup job: %s", dcj.name)
}

func main() {
    fmt.Println("=== Package Design Guidelines Demonstration ===")

    // GOOD: Purpose-driven packages
    fmt.Println("\n--- Scheduler Package (Good Purpose) ---")

    sched := scheduler.New()

    job := &scheduler.Job{
        ID:      "cleanup-001",
        Name:    "Daily Cleanup",
        Schedule: "0 2 * * *", // 2 AM daily
        Handler: &DataCleanupJob{name: "temp files"},
        NextRun: time.Now().Add(1 * time.Minute),
        Enabled: true,
    }

    err := sched.AddJob(job)
    if err != nil {
        fmt.Printf("Failed to add job: %v\n", err)
    } else {
        fmt.Printf("Added job: %s\n", job.Name)
    }

    fmt.Println("\n--- Notification Package (Good Usability) ---")

    notifService := notification.NewService()
    emailSender := notification.NewEmailSender("smtp.company.com", 587)
    notifService.RegisterSender(notification.Email, emailSender)

    message := &notification.Message{
        To:      "admin@company.com",
        Subject: "System Alert",
        Body:    "Scheduled maintenance completed successfully",
        Type:    notification.Email,
    }

    ctx := context.Background()
    err = notifService.Send(ctx, message)
    if err != nil {
        fmt.Printf("Failed to send notification: %v\n", err)
    }

    fmt.Println("\n--- Text Processor Package (Good Portability) ---")

    processor := textprocessor.New(textprocessor.Options{
        CaseSensitive: false,
        TrimSpaces:   true,
    })

    text := "  Hello World  "
    transformed := processor.Transform(text)
    fmt.Printf("Original: '%s' -> Transformed: '%s'\n", text, transformed)

    fmt.Println("\n--- Time Utilities Package (Good Focus) ---")

    now := time.Now()

    // Use different formatters
    fmt.Printf("ISO8601: %s\n", timeutil.ISO8601.Format(now))
    fmt.Printf("Date only: %s\n", timeutil.DateOnly.Format(now))
    fmt.Printf("Time only: %s\n", timeutil.TimeOnly.Format(now))

    // Business day calculations
    fmt.Printf("Is today a business day? %t\n", timeutil.IsBusinessDay(now))

    nextWeek := now.AddDate(0, 0, 7)
    businessDays := timeutil.BusinessDaysUntil(now, nextWeek)
    fmt.Printf("Business days until next week: %d\n", businessDays)

    fmt.Println("\n=== Design Guidelines Summary ===")
    fmt.Println("✅ PURPOSE: Packages provide specific capabilities, not generic containers")
    fmt.Println("✅ USABILITY: APIs are intuitive, simple, and hard to misuse")
    fmt.Println("✅ PORTABILITY: Minimal coupling, focused responsibility, reusable")
    fmt.Println("❌ AVOID: 'models', 'utils', 'helpers', 'common' packages")
    fmt.Println("❌ AVOID: Packages that contain instead of provide")
    fmt.Println("❌ AVOID: Single points of dependency")
}
```

**Hands-on Exercise 3: Package-Oriented Design and Project Architecture**:

```go
// Demonstrating package-oriented design for a complete automation system
// This exercise shows how to structure a real-world Go project

// PROJECT STRUCTURE:
// automation-system/
// ├── cmd/
// │   ├── server/
// │   │   └── main.go          // Server entry point
// │   └── cli/
// │       └── main.go          // CLI entry point
// ├── internal/
// │   ├── core/
// │   │   ├── task/            // Task domain logic
// │   │   ├── user/            // User domain logic
// │   │   └── workflow/        // Workflow domain logic
// │   ├── platform/
// │   │   ├── database/        // Database implementations
// │   │   ├── queue/           // Queue implementations
// │   │   └── cache/           // Cache implementations
// │   └── app/
// │       ├── taskservice/     // Task application service
// │       └── userservice/     // User application service
// └── pkg/
//     ├── logger/              // Shared logging
//     ├── config/              // Shared configuration
//     └── metrics/             // Shared metrics

// File: internal/core/task/task.go
package task

import (
    "context"
    "fmt"
    "time"
)

// CORE DOMAIN: Task represents the business entity
// This package contains the core business logic for tasks
// No external dependencies - pure business rules

// Status represents task execution status
type Status string

const (
    StatusPending   Status = "pending"
    StatusRunning   Status = "running"
    StatusCompleted Status = "completed"
    StatusFailed    Status = "failed"
    StatusCancelled Status = "cancelled"
)

// Task represents a unit of work in the automation system
type Task struct {
    ID          string
    Name        string
    Description string
    Status      Status
    CreatedAt   time.Time
    UpdatedAt   time.Time
    StartedAt   *time.Time
    CompletedAt *time.Time
    Config      map[string]interface{}
    Result      *Result
}

// Result represents the outcome of task execution
type Result struct {
    Success bool
    Message string
    Data    map[string]interface{}
    Error   error
}

// Repository defines how tasks are persisted
// Interface defined in domain, implemented in infrastructure
type Repository interface {
    Save(ctx context.Context, task *Task) error
    FindByID(ctx context.Context, id string) (*Task, error)
    FindByStatus(ctx context.Context, status Status) ([]*Task, error)
    Delete(ctx context.Context, id string) error
}

// Service defines task business operations
type Service interface {
    CreateTask(ctx context.Context, name, description string, config map[string]interface{}) (*Task, error)
    StartTask(ctx context.Context, id string) error
    CompleteTask(ctx context.Context, id string, result *Result) error
    CancelTask(ctx context.Context, id string) error
    GetTask(ctx context.Context, id string) (*Task, error)
    ListTasksByStatus(ctx context.Context, status Status) ([]*Task, error)
}

// Business logic methods on Task entity
func (t *Task) CanStart() error {
    if t.Status != StatusPending {
        return fmt.Errorf("task %s cannot be started: current status is %s", t.ID, t.Status)
    }
    return nil
}

func (t *Task) Start() error {
    if err := t.CanStart(); err != nil {
        return err
    }

    now := time.Now()
    t.Status = StatusRunning
    t.StartedAt = &now
    t.UpdatedAt = now

    return nil
}

func (t *Task) Complete(result *Result) error {
    if t.Status != StatusRunning {
        return fmt.Errorf("task %s cannot be completed: current status is %s", t.ID, t.Status)
    }

    now := time.Now()
    t.Status = StatusCompleted
    t.CompletedAt = &now
    t.UpdatedAt = now
    t.Result = result

    return nil
}

func (t *Task) Cancel() error {
    if t.Status == StatusCompleted {
        return fmt.Errorf("task %s cannot be cancelled: already completed", t.ID)
    }

    t.Status = StatusCancelled
    t.UpdatedAt = time.Now()

    return nil
}

// File: internal/platform/database/taskrepository.go
package database

import (
    "automation-system/internal/core/task"
    "context"
    "fmt"
    "sync"
)

// INFRASTRUCTURE LAYER: Implements domain interfaces
// This package provides concrete implementations for data persistence

// TaskRepository implements task.Repository interface
type TaskRepository struct {
    mu    sync.RWMutex
    tasks map[string]*task.Task // In-memory storage for demo
}

// NewTaskRepository creates a new task repository
func NewTaskRepository() *TaskRepository {
    return &TaskRepository{
        tasks: make(map[string]*task.Task),
    }
}

func (tr *TaskRepository) Save(ctx context.Context, t *task.Task) error {
    tr.mu.Lock()
    defer tr.mu.Unlock()

    tr.tasks[t.ID] = t
    return nil
}

func (tr *TaskRepository) FindByID(ctx context.Context, id string) (*task.Task, error) {
    tr.mu.RLock()
    defer tr.mu.RUnlock()

    t, exists := tr.tasks[id]
    if !exists {
        return nil, fmt.Errorf("task with ID %s not found", id)
    }

    return t, nil
}

func (tr *TaskRepository) FindByStatus(ctx context.Context, status task.Status) ([]*task.Task, error) {
    tr.mu.RLock()
    defer tr.mu.RUnlock()

    var tasks []*task.Task
    for _, t := range tr.tasks {
        if t.Status == status {
            tasks = append(tasks, t)
        }
    }

    return tasks, nil
}

func (tr *TaskRepository) Delete(ctx context.Context, id string) error {
    tr.mu.Lock()
    defer tr.mu.Unlock()

    if _, exists := tr.tasks[id]; !exists {
        return fmt.Errorf("task with ID %s not found", id)
    }

    delete(tr.tasks, id)
    return nil
}

// File: internal/app/taskservice/service.go
package taskservice

import (
    "automation-system/internal/core/task"
    "context"
    "fmt"
    "time"
)

// APPLICATION LAYER: Orchestrates domain operations
// This package implements application use cases

// Service implements task.Service interface
type Service struct {
    repo task.Repository
}

// New creates a new task service
func New(repo task.Repository) *Service {
    return &Service{
        repo: repo,
    }
}

func (s *Service) CreateTask(ctx context.Context, name, description string, config map[string]interface{}) (*task.Task, error) {
    if name == "" {
        return nil, fmt.Errorf("task name cannot be empty")
    }

    t := &task.Task{
        ID:          generateID(), // Simplified ID generation
        Name:        name,
        Description: description,
        Status:      task.StatusPending,
        CreatedAt:   time.Now(),
        UpdatedAt:   time.Now(),
        Config:      config,
    }

    if err := s.repo.Save(ctx, t); err != nil {
        return nil, fmt.Errorf("failed to save task: %w", err)
    }

    return t, nil
}

func (s *Service) StartTask(ctx context.Context, id string) error {
    t, err := s.repo.FindByID(ctx, id)
    if err != nil {
        return fmt.Errorf("failed to find task: %w", err)
    }

    if err := t.Start(); err != nil {
        return err
    }

    if err := s.repo.Save(ctx, t); err != nil {
        return fmt.Errorf("failed to save task: %w", err)
    }

    return nil
}

func (s *Service) CompleteTask(ctx context.Context, id string, result *task.Result) error {
    t, err := s.repo.FindByID(ctx, id)
    if err != nil {
        return fmt.Errorf("failed to find task: %w", err)
    }

    if err := t.Complete(result); err != nil {
        return err
    }

    if err := s.repo.Save(ctx, t); err != nil {
        return fmt.Errorf("failed to save task: %w", err)
    }

    return nil
}

func (s *Service) CancelTask(ctx context.Context, id string) error {
    t, err := s.repo.FindByID(ctx, id)
    if err != nil {
        return fmt.Errorf("failed to find task: %w", err)
    }

    if err := t.Cancel(); err != nil {
        return err
    }

    if err := s.repo.Save(ctx, t); err != nil {
        return fmt.Errorf("failed to save task: %w", err)
    }

    return nil
}

func (s *Service) GetTask(ctx context.Context, id string) (*task.Task, error) {
    return s.repo.FindByID(ctx, id)
}

func (s *Service) ListTasksByStatus(ctx context.Context, status task.Status) ([]*task.Task, error) {
    return s.repo.FindByStatus(ctx, status)
}

// Simplified ID generation for demo
func generateID() string {
    return fmt.Sprintf("task_%d", time.Now().UnixNano())
}

// File: pkg/logger/logger.go
package logger

import (
    "fmt"
    "log"
    "os"
)

// SHARED PACKAGE: Reusable across multiple applications
// This package provides logging capabilities for the entire system

// Level represents logging level
type Level string

const (
    DEBUG Level = "DEBUG"
    INFO  Level = "INFO"
    WARN  Level = "WARN"
    ERROR Level = "ERROR"
)

// Logger provides structured logging
type Logger struct {
    level  Level
    prefix string
    logger *log.Logger
}

// New creates a new logger instance
func New(level Level, prefix string) *Logger {
    return &Logger{
        level:  level,
        prefix: prefix,
        logger: log.New(os.Stdout, fmt.Sprintf("[%s] ", prefix), log.LstdFlags),
    }
}

func (l *Logger) Debug(msg string) {
    if l.shouldLog(DEBUG) {
        l.logger.Printf("[%s] %s", DEBUG, msg)
    }
}

func (l *Logger) Info(msg string) {
    if l.shouldLog(INFO) {
        l.logger.Printf("[%s] %s", INFO, msg)
    }
}

func (l *Logger) Warn(msg string) {
    if l.shouldLog(WARN) {
        l.logger.Printf("[%s] %s", WARN, msg)
    }
}

func (l *Logger) Error(msg string) {
    if l.shouldLog(ERROR) {
        l.logger.Printf("[%s] %s", ERROR, msg)
    }
}

func (l *Logger) shouldLog(level Level) bool {
    levels := map[Level]int{
        DEBUG: 0,
        INFO:  1,
        WARN:  2,
        ERROR: 3,
    }

    currentLevel, exists := levels[l.level]
    if !exists {
        return true
    }

    messageLevel, exists := levels[level]
    if !exists {
        return true
    }

    return messageLevel >= currentLevel
}

// File: cmd/server/main.go
package main

import (
    "automation-system/internal/app/taskservice"
    "automation-system/internal/core/task"
    "automation-system/internal/platform/database"
    "automation-system/pkg/logger"
    "context"
    "fmt"
)

// PRESENTATION LAYER: Entry point for server application
// This demonstrates how all layers work together

func main() {
    fmt.Println("=== Package-Oriented Design Demonstration ===")

    // Initialize shared services
    log := logger.New(logger.INFO, "SERVER")
    log.Info("Starting automation server")

    // Initialize infrastructure layer
    taskRepo := database.NewTaskRepository()
    log.Info("Task repository initialized")

    // Initialize application layer
    taskSvc := taskservice.New(taskRepo)
    log.Info("Task service initialized")

    // Demonstrate the complete flow
    ctx := context.Background()

    fmt.Println("\n--- Creating Tasks ---")

    // Create tasks
    task1, err := taskSvc.CreateTask(ctx, "Data Backup", "Backup user data to S3", map[string]interface{}{
        "bucket": "backup-bucket",
        "region": "us-west-2",
    })
    if err != nil {
        log.Error(fmt.Sprintf("Failed to create task: %v", err))
        return
    }
    log.Info(fmt.Sprintf("Created task: %s", task1.ID))

    task2, err := taskSvc.CreateTask(ctx, "Send Notifications", "Send daily summary emails", map[string]interface{}{
        "template": "daily-summary",
        "recipients": []string{"admin@company.com"},
    })
    if err != nil {
        log.Error(fmt.Sprintf("Failed to create task: %v", err))
        return
    }
    log.Info(fmt.Sprintf("Created task: %s", task2.ID))

    fmt.Println("\n--- Managing Task Lifecycle ---")

    // Start first task
    err = taskSvc.StartTask(ctx, task1.ID)
    if err != nil {
        log.Error(fmt.Sprintf("Failed to start task: %v", err))
    } else {
        log.Info(fmt.Sprintf("Started task: %s", task1.ID))
    }

    // Complete first task
    result := &task.Result{
        Success: true,
        Message: "Backup completed successfully",
        Data: map[string]interface{}{
            "files_backed_up": 1250,
            "total_size":     "2.5GB",
        },
    }

    err = taskSvc.CompleteTask(ctx, task1.ID, result)
    if err != nil {
        log.Error(fmt.Sprintf("Failed to complete task: %v", err))
    } else {
        log.Info(fmt.Sprintf("Completed task: %s", task1.ID))
    }

    // Cancel second task
    err = taskSvc.CancelTask(ctx, task2.ID)
    if err != nil {
        log.Error(fmt.Sprintf("Failed to cancel task: %v", err))
    } else {
        log.Info(fmt.Sprintf("Cancelled task: %s", task2.ID))
    }

    fmt.Println("\n--- Querying Tasks ---")

    // Get specific task
    retrievedTask, err := taskSvc.GetTask(ctx, task1.ID)
    if err != nil {
        log.Error(fmt.Sprintf("Failed to get task: %v", err))
    } else {
        fmt.Printf("Retrieved task: %s (Status: %s)\n", retrievedTask.Name, retrievedTask.Status)
        if retrievedTask.Result != nil {
            fmt.Printf("  Result: %s\n", retrievedTask.Result.Message)
        }
    }

    // List tasks by status
    completedTasks, err := taskSvc.ListTasksByStatus(ctx, task.StatusCompleted)
    if err != nil {
        log.Error(fmt.Sprintf("Failed to list completed tasks: %v", err))
    } else {
        fmt.Printf("Completed tasks: %d\n", len(completedTasks))
        for _, t := range completedTasks {
            fmt.Printf("  - %s: %s\n", t.ID, t.Name)
        }
    }

    cancelledTasks, err := taskSvc.ListTasksByStatus(ctx, task.StatusCancelled)
    if err != nil {
        log.Error(fmt.Sprintf("Failed to list cancelled tasks: %v", err))
    } else {
        fmt.Printf("Cancelled tasks: %d\n", len(cancelledTasks))
        for _, t := range cancelledTasks {
            fmt.Printf("  - %s: %s\n", t.ID, t.Name)
        }
    }

    log.Info("Server demonstration completed")

    fmt.Println("\n=== Package-Oriented Design Principles ===")
    fmt.Println("✅ LAYERED ARCHITECTURE:")
    fmt.Println("  - cmd/: Application entry points")
    fmt.Println("  - internal/core/: Domain logic (business rules)")
    fmt.Println("  - internal/platform/: Infrastructure implementations")
    fmt.Println("  - internal/app/: Application services (use cases)")
    fmt.Println("  - pkg/: Shared, reusable packages")
    fmt.Println()
    fmt.Println("✅ DEPENDENCY DIRECTION:")
    fmt.Println("  - Core domain has no external dependencies")
    fmt.Println("  - Infrastructure depends on domain interfaces")
    fmt.Println("  - Application orchestrates domain operations")
    fmt.Println("  - Presentation layer wires everything together")
    fmt.Println()
    fmt.Println("✅ PACKAGE ORGANIZATION:")
    fmt.Println("  - Each package has a clear, single purpose")
    fmt.Println("  - Interfaces defined where they're needed")
    fmt.Println("  - Implementation details hidden")
    fmt.Println("  - Easy to test, maintain, and extend")
}
```

**Prerequisites**: Module 25
