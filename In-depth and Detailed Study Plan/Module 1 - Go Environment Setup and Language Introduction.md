# Module 1: Go Environment Setup and Language Introduction

**Duration**: 1 hour (45 min theory + 15 min hands-on)

**Learning Objectives**:

- Set up Go development environment with latest tooling (Go 1.24+)
- Understand Go's design philosophy and use cases in automation
- Learn basic Go toolchain commands and workspace management
- Create first Go program with modern best practices

**Videos Covered**:

- Course introduction and overview (conceptual)

**Key Concepts**:

- Go installation and GOPATH vs Go modules
- Go workspaces for multi-module projects
- Basic project structure and naming conventions
- Introduction to `go mod`, `go build`, `go run`
- VS Code setup with Go extension

**Hands-on Exercise 1: Go Environment and Toolchain Exploration**:

```go
// Create a simple automation utility
package main

import (
    "fmt"
    "os"
    "time"
)

func main() {
    fmt.Printf("System Monitor started at %v\n", time.Now())
    fmt.Printf("Current working directory: %s\n", getCurrentDir())
}

func getCurrentDir() string {
    dir, err := os.Getwd()
    if err != nil {
        return "unknown"
    }
    return dir
}
```

**Hands-on Exercise 2: Go Module and Workspace Setup**:

```go
// Multi-module automation project setup
// This exercise demonstrates Go modules and workspace management

// File: automation-platform/go.mod
module automation-platform

go 1.24

require (
    automation-platform/config v0.0.0
    automation-platform/logger v0.0.0
)

replace automation-platform/config => ./config
replace automation-platform/logger => ./logger

// File: automation-platform/main.go
package main

import (
    "fmt"
    "automation-platform/config"
    "automation-platform/logger"
)

func main() {
    cfg := config.Load()
    log := logger.New(cfg.LogLevel)

    log.Info("Automation platform starting...")
    fmt.Printf("Platform: %s v%s\n", cfg.Name, cfg.Version)
    fmt.Printf("Environment: %s\n", cfg.Environment)

    log.Info("Platform initialization complete")
}

// File: automation-platform/config/go.mod
module automation-platform/config

go 1.24

// File: automation-platform/config/config.go
package config

type Config struct {
    Name        string
    Version     string
    Environment string
    LogLevel    string
}

func Load() *Config {
    return &Config{
        Name:        "AutomationPlatform",
        Version:     "1.0.0",
        Environment: "development",
        LogLevel:    "info",
    }
}

// File: automation-platform/logger/go.mod
module automation-platform/logger

go 1.24

// File: automation-platform/logger/logger.go
package logger

import (
    "fmt"
    "time"
)

type Logger struct {
    level string
}

func New(level string) *Logger {
    return &Logger{level: level}
}

func (l *Logger) Info(message string) {
    l.log("INFO", message)
}

func (l *Logger) Error(message string) {
    l.log("ERROR", message)
}

func (l *Logger) log(level, message string) {
    timestamp := time.Now().Format("2006-01-02 15:04:05")
    fmt.Printf("[%s] %s: %s\n", timestamp, level, message)
}
```

**Hands-on Exercise 3: Go Build and Development Workflow**:

```go
// Automation task runner with build tags and conditional compilation
// This exercise demonstrates Go build process and development workflow

//go:build development
// +build development

package main

import (
    "fmt"
    "os"
    "runtime"
    "time"
)

// Development-specific configuration
const (
    DEBUG_MODE = true
    LOG_LEVEL  = "debug"
    TIMEOUT    = 5 * time.Second
)

func main() {
    fmt.Println("=== Automation Task Runner (Development Build) ===")

    // Display build information
    showBuildInfo()

    // Display environment information
    showEnvironmentInfo()

    // Run development-specific tasks
    runDevelopmentTasks()
}

func showBuildInfo() {
    fmt.Printf("Go Version: %s\n", runtime.Version())
    fmt.Printf("OS/Arch: %s/%s\n", runtime.GOOS, runtime.GOARCH)
    fmt.Printf("Build Mode: Development\n")
    fmt.Printf("Debug Mode: %t\n", DEBUG_MODE)
    fmt.Printf("Log Level: %s\n", LOG_LEVEL)
    fmt.Println()
}

func showEnvironmentInfo() {
    fmt.Println("Environment Variables:")
    fmt.Printf("GOPATH: %s\n", os.Getenv("GOPATH"))
    fmt.Printf("GOROOT: %s\n", os.Getenv("GOROOT"))
    fmt.Printf("PATH: %s\n", os.Getenv("PATH"))

    if workDir, err := os.Getwd(); err == nil {
        fmt.Printf("Working Directory: %s\n", workDir)
    }

    fmt.Printf("Number of CPUs: %d\n", runtime.NumCPU())
    fmt.Printf("Number of Goroutines: %d\n", runtime.NumGoroutine())
    fmt.Println()
}

func runDevelopmentTasks() {
    fmt.Println("Running Development Tasks:")

    tasks := []struct {
        name string
        fn   func() error
    }{
        {"Initialize Configuration", initializeConfig},
        {"Setup Development Database", setupDevDatabase},
        {"Start Development Server", startDevServer},
        {"Run Health Checks", runHealthChecks},
    }

    for i, task := range tasks {
        fmt.Printf("%d. %s... ", i+1, task.name)

        start := time.Now()
        if err := task.fn(); err != nil {
            fmt.Printf("FAILED (%v)\n", err)
            continue
        }

        duration := time.Since(start)
        fmt.Printf("OK (%v)\n", duration)

        // Development mode: add delay for observation
        if DEBUG_MODE {
            time.Sleep(500 * time.Millisecond)
        }
    }

    fmt.Println("\nDevelopment environment ready!")
}

func initializeConfig() error {
    // Simulate configuration initialization
    time.Sleep(100 * time.Millisecond)
    return nil
}

func setupDevDatabase() error {
    // Simulate database setup
    time.Sleep(200 * time.Millisecond)
    return nil
}

func startDevServer() error {
    // Simulate server startup
    time.Sleep(300 * time.Millisecond)
    return nil
}

func runHealthChecks() error {
    // Simulate health checks
    time.Sleep(150 * time.Millisecond)
    return nil
}

// Production build version (automation-runner-prod.go)
//go:build production
// +build production

/*
package main

import (
    "fmt"
    "runtime"
    "time"
)

const (
    DEBUG_MODE = false
    LOG_LEVEL  = "info"
    TIMEOUT    = 30 * time.Second
)

func main() {
    fmt.Println("=== Automation Task Runner (Production Build) ===")
    fmt.Printf("Go Version: %s\n", runtime.Version())
    fmt.Printf("Build Mode: Production\n")
    fmt.Printf("Debug Mode: %t\n", DEBUG_MODE)

    // Production-optimized execution
    runProductionTasks()
}

func runProductionTasks() {
    // Production task implementation
    fmt.Println("Production tasks executed successfully")
}
*/
```

**Prerequisites**: None
