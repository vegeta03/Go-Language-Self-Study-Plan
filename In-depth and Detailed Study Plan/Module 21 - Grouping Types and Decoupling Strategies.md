# Module 21: Grouping Types and Decoupling Strategies

**Duration**: 1 hour (45 min theory + 15 min hands-on)

**Learning Objectives**:

- Understand Go's approach to grouping: behavior over configuration
- Learn why embedding doesn't create subtyping relationships
- Master interface-based grouping and composition patterns
- Apply decoupling strategies in automation system design
- Design flexible, extensible automation architectures

**Videos Covered**:

- 5.1 Topics (0:00:59)
- 5.2 Grouping Types (0:12:38)
- 5.3 Decoupling Part 1 (0:06:58)
- 5.4 Decoupling Part 2 (0:18:25)
- 5.5 Decoupling Part 3 (0:14:36)

**Key Concepts**:

- Convention over configuration: group by behavior, not identity
- No subtyping in Go: embedding ≠  subtyping
- Interface-based grouping enables diversity and flexibility
- Decoupling through behavior contracts
- Composition patterns for extensible systems
- Design from concrete to abstract

**Hands-on Exercise 1: Convention Over Configuration - Grouping by Behavior**:

```go
// Demonstrating the difference between grouping by identity vs behavior
package main

import (
    "fmt"
)

// WRONG APPROACH: Grouping by what things ARE (configuration/identity)
// This is the common mistake from class-based programming backgrounds

// AutomationComponent - a "base type" for reusable state
type AutomationComponent struct {
    Name      string
    Version   string
    IsActive  bool
}

// Generic behavior that provides no real value
func (ac AutomationComponent) GetInfo() string {
    return fmt.Sprintf("%s v%s (Active: %t)", ac.Name, ac.Version, ac.IsActive)
}

// DataProcessor embeds AutomationComponent
type DataProcessor struct {
    AutomationComponent // Embedding
    ProcessingRate      int
}

func (dp DataProcessor) ProcessData(data string) string {
    return fmt.Sprintf("Processed by %s: %s", dp.Name, data)
}

// AlertManager embeds AutomationComponent
type AlertManager struct {
    AutomationComponent // Embedding
    AlertThreshold      float64
}

func (am AlertManager) SendAlert(message string) {
    fmt.Printf("ALERT from %s: %s\n", am.Name, message)
}

// This WILL NOT WORK - cannot group by embedded type
// Embedding does NOT create subtyping relationships!
func demonstrateGroupingProblem() {
    fmt.Println("=== Attempting to Group by Identity (FAILS) ===")

    // This would cause compile errors:
    // var components []AutomationComponent = []AutomationComponent{
    //     DataProcessor{...}, // Compile error! DataProcessor is not AutomationComponent
    //     AlertManager{...},  // Compile error! AlertManager is not AutomationComponent
    // }

    fmt.Println("Cannot group DataProcessor and AlertManager by AutomationComponent")
    fmt.Println("Embedding does NOT create subtyping relationship!")
}

// CORRECT APPROACH: Grouping by what things DO (behavior/convention)

// Define behavior contracts - what things DO, not what they ARE
type Speaker interface {
    Speak() string
}

// Concrete types focused on their specific purpose
type Dog struct {
    Name     string
    IsMammal bool
    Pack     string
}

func (d Dog) Speak() string {
    return "Woof!"
}

type Cat struct {
    Name     string
    IsMammal bool
    Lives    int
}

func (c Cat) Speak() string {
    return "Meow!"
}

type Robot struct {
    Model      string
    BatteryLevel int
}

func (r Robot) Speak() string {
    return "Beep boop!"
}

// Now we can group by behavior - what they DO
func demonstrateBehaviorGrouping() {
    fmt.Println("\n=== Grouping by Behavior (WORKS) ===")

    // Create diverse types
    dog := Dog{Name: "Buddy", IsMammal: true, Pack: "Family"}
    cat := Cat{Name: "Whiskers", IsMammal: true, Lives: 9}
    robot := Robot{Model: "R2D2", BatteryLevel: 85}

    // Group by what they DO, not what they ARE
    speakers := []Speaker{dog, cat, robot}

    fmt.Println("All speakers can speak:")
    for i, speaker := range speakers {
        fmt.Printf("Speaker %d: %s\n", i+1, speaker.Speak())
    }

    // This demonstrates diversity - completely different types
    // can be grouped together by common behavior
}

// Automation system example with behavior-based grouping
type Processor interface {
    Process(data string) string
}

type Logger interface {
    Log(message string)
}

// FileHandler processes files and logs
type FileHandler struct {
    name      string
    directory string
}

func (fh FileHandler) Process(data string) string {
    return fmt.Sprintf("File processed: %s in %s", data, fh.directory)
}

func (fh FileHandler) Log(message string) {
    fmt.Printf("[FILE-LOG] %s: %s\n", fh.name, message)
}

// APIHandler processes API calls and logs
type APIHandler struct {
    name     string
    endpoint string
}

func (ah APIHandler) Process(data string) string {
    return fmt.Sprintf("API processed: %s via %s", data, ah.endpoint)
}

func (ah APIHandler) Log(message string) {
    fmt.Printf("[API-LOG] %s: %s\n", ah.name, message)
}

func demonstrateAutomationGrouping() {
    fmt.Println("\n=== Automation System Behavior Grouping ===")

    fileHandler := FileHandler{name: "CSVProcessor", directory: "/data"}
    apiHandler := APIHandler{name: "RestProcessor", endpoint: "api.example.com"}

    // Group by processing behavior
    processors := []Processor{fileHandler, apiHandler}

    // Group by logging behavior
    loggers := []Logger{fileHandler, apiHandler}

    // Process data with all processors
    testData := "sample_data"
    fmt.Println("Processing with all processors:")
    for _, processor := range processors {
        result := processor.Process(testData)
        fmt.Printf("  %s\n", result)
    }

    // Log with all loggers
    fmt.Println("\nLogging with all loggers:")
    for _, logger := range loggers {
        logger.Log("Operation completed successfully")
    }
}

func main() {
    fmt.Println("=== Convention Over Configuration in Go ===")

    // Show the problem with identity-based grouping
    demonstrateGroupingProblem()

    // Show the solution with behavior-based grouping
    demonstrateBehaviorGrouping()

    // Show practical automation example
    demonstrateAutomationGrouping()

    fmt.Println("\n=== Key Principles ===")
    fmt.Println("1. Go uses convention over configuration")
    fmt.Println("2. Group by what things DO, not what they ARE")
    fmt.Println("3. Embedding does NOT create subtyping relationships")
    fmt.Println("4. Interfaces enable diversity and flexibility")
    fmt.Println("5. Focus on behavior contracts, not identity")
}
```

**Hands-on Exercise 2: Decoupling Strategies and Layered API Design**:

```go
// Demonstrating concrete-first development and layered API design
package main

import (
    "fmt"
    "strings"
    "time"
)

// STEP 1: Start with concrete implementation - solve the problem first
// Problem: Process automation logs with different formats and outputs

// Concrete log processor - no interfaces yet!
type LogProcessor struct {
    name        string
    inputDir    string
    outputDir   string
    processed   int
    errors      int
}

// PRIMITIVE LAYER: Core functionality - does one thing well
// This layer focuses on the fundamental data transformation

func (lp *LogProcessor) parseLogLine(line string) (timestamp time.Time, level string, message string, err error) {
    // Simple log format: "2023-01-01 10:00:00 INFO Application started"
    parts := strings.SplitN(line, " ", 4)
    if len(parts) < 4 {
        return time.Time{}, "", "", fmt.Errorf("invalid log format")
    }

    timeStr := parts[0] + " " + parts[1]
    timestamp, err = time.Parse("2006-01-02 15:04:05", timeStr)
    if err != nil {
        return time.Time{}, "", "", fmt.Errorf("invalid timestamp: %v", err)
    }

    level = parts[2]
    message = parts[3]
    return timestamp, level, message, nil
}

func (lp *LogProcessor) formatOutput(timestamp time.Time, level string, message string, format string) string {
    switch format {
    case "json":
        return fmt.Sprintf(`{"timestamp":"%s","level":"%s","message":"%s"}`,
            timestamp.Format(time.RFC3339), level, message)
    case "csv":
        return fmt.Sprintf("%s,%s,%s",
            timestamp.Format(time.RFC3339), level, message)
    default: // plain
        return fmt.Sprintf("[%s] %s: %s",
            timestamp.Format("15:04:05"), level, message)
    }
}

// LOWER LEVEL LAYER: Builds on primitive layer, handles batches

func (lp *LogProcessor) processLogBatch(lines []string, outputFormat string) ([]string, []error) {
    var results []string
    var errors []error

    for _, line := range lines {
        if strings.TrimSpace(line) == "" {
            continue // Skip empty lines
        }

        timestamp, level, message, err := lp.parseLogLine(line)
        if err != nil {
            errors = append(errors, fmt.Errorf("line '%s': %v", line, err))
            lp.errors++
            continue
        }

        formatted := lp.formatOutput(timestamp, level, message, outputFormat)
        results = append(results, formatted)
        lp.processed++
    }

    return results, errors
}

func (lp *LogProcessor) validateInput(lines []string) error {
    if len(lines) == 0 {
        return fmt.Errorf("no input lines provided")
    }

    validLines := 0
    for _, line := range lines {
        if strings.TrimSpace(line) != "" {
            validLines++
        }
    }

    if validLines == 0 {
        return fmt.Errorf("no valid input lines found")
    }

    return nil
}

// HIGH LEVEL LAYER: Easy-to-use API for end users

func (lp *LogProcessor) ProcessLogs(input []string, outputFormat string) (*ProcessingResult, error) {
    // Validate input
    if err := lp.validateInput(input); err != nil {
        return nil, fmt.Errorf("input validation failed: %v", err)
    }

    // Reset counters
    lp.processed = 0
    lp.errors = 0

    // Process the batch
    results, errors := lp.processLogBatch(input, outputFormat)

    // Create result summary
    result := &ProcessingResult{
        ProcessedLines: results,
        ProcessedCount: lp.processed,
        ErrorCount:     lp.errors,
        Errors:         errors,
        Format:         outputFormat,
        ProcessedAt:    time.Now(),
    }

    return result, nil
}

// Result type for high-level API
type ProcessingResult struct {
    ProcessedLines []string
    ProcessedCount int
    ErrorCount     int
    Errors         []error
    Format         string
    ProcessedAt    time.Time
}

func (pr *ProcessingResult) Summary() string {
    return fmt.Sprintf("Processed %d lines, %d errors, format: %s",
        pr.ProcessedCount, pr.ErrorCount, pr.Format)
}

// STEP 2: Now that we have a working solution, identify what needs decoupling
// After building concrete implementation, we can see what varies:
// 1. Input sources (files, network, database)
// 2. Output formats (json, csv, plain, xml)
// 3. Processing rules (filtering, transformation)

// STEP 3: Refactor for decoupling - introduce interfaces for variation points

// Parser interface for different log formats
type LogParser interface {
    Parse(line string) (LogEntry, error)
}

// Formatter interface for different output formats
type LogFormatter interface {
    Format(entry LogEntry) string
}

// LogEntry represents parsed log data
type LogEntry struct {
    Timestamp time.Time
    Level     string
    Message   string
    Source    string
}

// Concrete implementations of interfaces

type StandardLogParser struct{}

func (slp StandardLogParser) Parse(line string) (LogEntry, error) {
    parts := strings.SplitN(line, " ", 4)
    if len(parts) < 4 {
        return LogEntry{}, fmt.Errorf("invalid log format")
    }

    timeStr := parts[0] + " " + parts[1]
    timestamp, err := time.Parse("2006-01-02 15:04:05", timeStr)
    if err != nil {
        return LogEntry{}, fmt.Errorf("invalid timestamp: %v", err)
    }

    return LogEntry{
        Timestamp: timestamp,
        Level:     parts[2],
        Message:   parts[3],
        Source:    "standard",
    }, nil
}

type JSONFormatter struct{}

func (jf JSONFormatter) Format(entry LogEntry) string {
    return fmt.Sprintf(`{"timestamp":"%s","level":"%s","message":"%s","source":"%s"}`,
        entry.Timestamp.Format(time.RFC3339), entry.Level, entry.Message, entry.Source)
}

type CSVFormatter struct{}

func (cf CSVFormatter) Format(entry LogEntry) string {
    return fmt.Sprintf("%s,%s,%s,%s",
        entry.Timestamp.Format(time.RFC3339), entry.Level, entry.Message, entry.Source)
}

// Decoupled log processor using interfaces
type DecoupledLogProcessor struct {
    name      string
    parser    LogParser
    formatter LogFormatter
    processed int
    errors    int
}

func NewDecoupledLogProcessor(name string, parser LogParser, formatter LogFormatter) *DecoupledLogProcessor {
    return &DecoupledLogProcessor{
        name:      name,
        parser:    parser,
        formatter: formatter,
    }
}

func (dlp *DecoupledLogProcessor) ProcessLogs(lines []string) (*ProcessingResult, error) {
    dlp.processed = 0
    dlp.errors = 0

    var results []string
    var errors []error

    for _, line := range lines {
        if strings.TrimSpace(line) == "" {
            continue
        }

        entry, err := dlp.parser.Parse(line)
        if err != nil {
            errors = append(errors, fmt.Errorf("parse error: %v", err))
            dlp.errors++
            continue
        }

        formatted := dlp.formatter.Format(entry)
        results = append(results, formatted)
        dlp.processed++
    }

    return &ProcessingResult{
        ProcessedLines: results,
        ProcessedCount: dlp.processed,
        ErrorCount:     dlp.errors,
        Errors:         errors,
        Format:         "decoupled",
        ProcessedAt:    time.Now(),
    }, nil
}

func main() {
    fmt.Println("=== Layered API Design and Decoupling ===")

    // Sample log data
    logLines := []string{
        "2023-01-01 10:00:00 INFO Application started",
        "2023-01-01 10:01:00 WARN Low memory detected",
        "2023-01-01 10:02:00 ERROR Database connection failed",
        "", // empty line
        "invalid log line",
        "2023-01-01 10:03:00 INFO Processing completed",
    }

    fmt.Println("=== STEP 1: Concrete Implementation First ===")

    // Start with concrete implementation
    processor := &LogProcessor{
        name:      "ConcreteProcessor",
        inputDir:  "/logs",
        outputDir: "/processed",
    }

    // Test different output formats
    formats := []string{"plain", "json", "csv"}

    for _, format := range formats {
        fmt.Printf("\nProcessing with %s format:\n", format)
        result, err := processor.ProcessLogs(logLines, format)
        if err != nil {
            fmt.Printf("Error: %v\n", err)
            continue
        }

        fmt.Printf("Summary: %s\n", result.Summary())
        fmt.Println("Sample output:")
        for i, line := range result.ProcessedLines {
            if i < 2 { // Show first 2 lines
                fmt.Printf("  %s\n", line)
            }
        }

        if len(result.Errors) > 0 {
            fmt.Printf("Errors encountered: %d\n", len(result.Errors))
        }
    }

    fmt.Println("\n=== STEP 2: Decoupled Implementation ===")

    // Now use decoupled version with interfaces
    parser := StandardLogParser{}
    jsonFormatter := JSONFormatter{}
    csvFormatter := CSVFormatter{}

    // Create processors with different formatters
    jsonProcessor := NewDecoupledLogProcessor("JSONProcessor", parser, jsonFormatter)
    csvProcessor := NewDecoupledLogProcessor("CSVProcessor", parser, csvFormatter)

    processors := []*DecoupledLogProcessor{jsonProcessor, csvProcessor}

    for _, proc := range processors {
        fmt.Printf("\nProcessing with %s:\n", proc.name)
        result, err := proc.ProcessLogs(logLines)
        if err != nil {
            fmt.Printf("Error: %v\n", err)
            continue
        }

        fmt.Printf("Summary: %s\n", result.Summary())
        fmt.Println("Sample output:")
        for i, line := range result.ProcessedLines {
            if i < 2 {
                fmt.Printf("  %s\n", line)
            }
        }
    }

    fmt.Println("\n=== Design Principles Summary ===")
    fmt.Println("1. Start concrete - solve the problem first")
    fmt.Println("2. Build layered APIs: Primitive →’ Lower Level →’ High Level")
    fmt.Println("3. Each layer should be testable independently")
    fmt.Println("4. Identify variation points after concrete solution works")
    fmt.Println("5. Refactor to interfaces only when you need flexibility")
    fmt.Println("6. Decoupling is a refactoring step, not a starting point")
}
```

**Hands-on Exercise 3: Advanced Composition Patterns and Interface-Based Grouping**:

```go
// Advanced composition patterns for flexible automation systems
package main

import (
    "fmt"
    "log"
    "time"
)

// Define behavior contracts for grouping
type Processor interface {
    Process(data string) (string, error)
}

type Alerter interface {
    Alert(message string) error
}

type StatusReporter interface {
    Status() string
}

type Configurable interface {
    Configure(config map[string]interface{}) error
}

// Concrete implementations focused on behavior
type FileProcessor struct {
    name      string
    directory string
    isActive  bool
}

func (fp *FileProcessor) Process(data string) (string, error) {
    if !fp.isActive {
        return "", fmt.Errorf("processor %s is not active", fp.name)
    }
    result := fmt.Sprintf("File processed: %s -> %s/%s.processed", data, fp.directory, data)
    return result, nil
}

func (fp *FileProcessor) Status() string {
    status := "inactive"
    if fp.isActive {
        status = "active"
    }
    return fmt.Sprintf("FileProcessor %s: %s", fp.name, status)
}

func (fp *FileProcessor) Configure(config map[string]interface{}) error {
    if dir, ok := config["directory"].(string); ok {
        fp.directory = dir
    }
    if active, ok := config["active"].(bool); ok {
        fp.isActive = active
    }
    return nil
}

type NetworkProcessor struct {
    name     string
    endpoint string
    timeout  time.Duration
    isActive bool
}

func (np *NetworkProcessor) Process(data string) (string, error) {
    if !np.isActive {
        return "", fmt.Errorf("processor %s is not active", np.name)
    }
    // Simulate network processing
    time.Sleep(np.timeout)
    result := fmt.Sprintf("Network processed: %s via %s", data, np.endpoint)
    return result, nil
}

func (np *NetworkProcessor) Status() string {
    status := "inactive"
    if np.isActive {
        status = "active"
    }
    return fmt.Sprintf("NetworkProcessor %s: %s (endpoint: %s)", np.name, status, np.endpoint)
}

func (np *NetworkProcessor) Configure(config map[string]interface{}) error {
    if endpoint, ok := config["endpoint"].(string); ok {
        np.endpoint = endpoint
    }
    if timeout, ok := config["timeout"].(time.Duration); ok {
        np.timeout = timeout
    }
    if active, ok := config["active"].(bool); ok {
        np.isActive = active
    }
    return nil
}

type EmailAlerter struct {
    name       string
    smtpHost   string
    recipients []string
    isActive   bool
}

func (ea *EmailAlerter) Alert(message string) error {
    if !ea.isActive {
        return fmt.Errorf("alerter %s is not active", ea.name)
    }
    fmt.Printf("EMAIL ALERT from %s via %s: %s\n", ea.name, ea.smtpHost, message)
    fmt.Printf("Recipients: %v\n", ea.recipients)
    return nil
}

func (ea *EmailAlerter) Status() string {
    status := "inactive"
    if ea.isActive {
        status = "active"
    }
    return fmt.Sprintf("EmailAlerter %s: %s", ea.name, status)
}

func (ea *EmailAlerter) Configure(config map[string]interface{}) error {
    if host, ok := config["smtp_host"].(string); ok {
        ea.smtpHost = host
    }
    if recipients, ok := config["recipients"].([]string); ok {
        ea.recipients = recipients
    }
    if active, ok := config["active"].(bool); ok {
        ea.isActive = active
    }
    return nil
}

// HybridProcessor demonstrates multiple interface implementation
type HybridProcessor struct {
    name     string
    isActive bool
}

func (hp *HybridProcessor) Process(data string) (string, error) {
    if !hp.isActive {
        return "", fmt.Errorf("hybrid processor is not active")
    }
    return fmt.Sprintf("Hybrid processed: %s", data), nil
}

func (hp *HybridProcessor) Alert(message string) error {
    if !hp.isActive {
        return fmt.Errorf("hybrid alerter is not active")
    }
    fmt.Printf("HYBRID ALERT from %s: %s\n", hp.name, message)
    return nil
}

func (hp *HybridProcessor) Status() string {
    status := "inactive"
    if hp.isActive {
        status = "active"
    }
    return fmt.Sprintf("HybridProcessor %s: %s", hp.name, status)
}

func (hp *HybridProcessor) Configure(config map[string]interface{}) error {
    if active, ok := config["active"].(bool); ok {
        hp.isActive = active
    }
    return nil
}

// AutomationSystem groups components by behavior, not identity
type AutomationSystem struct {
    name          string
    processors    []Processor      // Group by behavior
    alerters      []Alerter        // Group by behavior
    reporters     []StatusReporter // Group by behavior
    configurables []Configurable   // Group by behavior
}

func NewAutomationSystem(name string) *AutomationSystem {
    return &AutomationSystem{
        name:          name,
        processors:    make([]Processor, 0),
        alerters:      make([]Alerter, 0),
        reporters:     make([]StatusReporter, 0),
        configurables: make([]Configurable, 0),
    }
}

// Smart component registration - automatically detects supported behaviors
func (as *AutomationSystem) AddProcessor(p Processor) {
    as.processors = append(as.processors, p)

    // Check if processor also implements other interfaces
    if reporter, ok := p.(StatusReporter); ok {
        as.reporters = append(as.reporters, reporter)
    }
    if configurable, ok := p.(Configurable); ok {
        as.configurables = append(as.configurables, configurable)
    }
}

func (as *AutomationSystem) AddAlerter(a Alerter) {
    as.alerters = append(as.alerters, a)

    // Check if alerter also implements other interfaces
    if reporter, ok := a.(StatusReporter); ok {
        as.reporters = append(as.reporters, reporter)
    }
    if configurable, ok := a.(Configurable); ok {
        as.configurables = append(as.configurables, configurable)
    }
}

// Orchestration methods that work with grouped behaviors
func (as *AutomationSystem) ProcessData(data []string) {
    fmt.Printf("\n=== Processing Data with %s ===\n", as.name)

    for _, item := range data {
        for i, processor := range as.processors {
            result, err := processor.Process(item)
            if err != nil {
                // Alert on error using all available alerters
                errorMsg := fmt.Sprintf("Processor %d failed: %v", i, err)
                for _, alerter := range as.alerters {
                    alerter.Alert(errorMsg)
                }
            } else {
                fmt.Printf("Result: %s\n", result)
            }
        }
    }
}

func (as *AutomationSystem) SystemStatus() {
    fmt.Printf("\n=== System Status: %s ===\n", as.name)
    for _, reporter := range as.reporters {
        fmt.Printf("- %s\n", reporter.Status())
    }
}

func (as *AutomationSystem) ConfigureAll(config map[string]interface{}) {
    fmt.Printf("\n=== Configuring %s ===\n", as.name)
    for _, configurable := range as.configurables {
        if err := configurable.Configure(config); err != nil {
            log.Printf("Configuration error: %v", err)
        }
    }
}

// Advanced composition: Pipeline pattern
type ProcessingPipeline struct {
    name       string
    stages     []Processor
    alerters   []Alerter
}

func NewProcessingPipeline(name string) *ProcessingPipeline {
    return &ProcessingPipeline{
        name:     name,
        stages:   make([]Processor, 0),
        alerters: make([]Alerter, 0),
    }
}

func (pp *ProcessingPipeline) AddStage(processor Processor) {
    pp.stages = append(pp.stages, processor)
}

func (pp *ProcessingPipeline) AddAlerter(alerter Alerter) {
    pp.alerters = append(pp.alerters, alerter)
}

func (pp *ProcessingPipeline) Execute(data string) (string, error) {
    fmt.Printf("\n=== Executing Pipeline: %s ===\n", pp.name)

    current := data
    for i, stage := range pp.stages {
        fmt.Printf("Stage %d: Processing '%s'\n", i+1, current)

        result, err := stage.Process(current)
        if err != nil {
            errorMsg := fmt.Sprintf("Pipeline %s failed at stage %d: %v", pp.name, i+1, err)

            // Alert all alerters about the failure
            for _, alerter := range pp.alerters {
                alerter.Alert(errorMsg)
            }

            return "", fmt.Errorf("pipeline failed at stage %d: %v", i+1, err)
        }

        current = result
        fmt.Printf("Stage %d result: '%s'\n", i+1, current)
    }

    fmt.Printf("Pipeline completed successfully\n")
    return current, nil
}

func main() {
    fmt.Println("=== Advanced Composition Patterns ===")

    // Create diverse components
    fileProc := &FileProcessor{name: "CSVProcessor", directory: "/data", isActive: true}
    netProc := &NetworkProcessor{name: "APIProcessor", endpoint: "api.example.com", timeout: 50 * time.Millisecond, isActive: true}
    emailAlert := &EmailAlerter{name: "CriticalAlerts", smtpHost: "smtp.company.com", recipients: []string{"admin@company.com"}, isActive: true}
    hybridProc := &HybridProcessor{name: "HybridProcessor", isActive: true}

    fmt.Println("=== Behavior-Based Grouping System ===")

    // Create automation system
    system := NewAutomationSystem("MainAutomationSystem")

    // Add components - they're grouped by what they DO, not what they ARE
    system.AddProcessor(fileProc)
    system.AddProcessor(netProc)
    system.AddProcessor(hybridProc) // Also implements Alerter
    system.AddAlerter(emailAlert)
    system.AddAlerter(hybridProc) // Same component, different behavior grouping

    // System status - all components that can report status
    system.SystemStatus()

    // Process data - all processors work together
    testData := []string{"data1.csv", "data2.json", "data3.xml"}
    system.ProcessData(testData)

    // Configure all configurable components
    config := map[string]interface{}{
        "active":     true,
        "directory":  "/new-data",
        "endpoint":   "new-api.example.com",
        "smtp_host":  "new-smtp.company.com",
    }
    system.ConfigureAll(config)

    fmt.Println("=== Pipeline Composition Pattern ===")

    // Create processing pipeline
    pipeline := NewProcessingPipeline("DataTransformationPipeline")

    // Add stages to pipeline
    pipeline.AddStage(fileProc)
    pipeline.AddStage(netProc)
    pipeline.AddAlerter(emailAlert)

    // Execute pipeline
    finalResult, err := pipeline.Execute("raw_data.txt")
    if err != nil {
        fmt.Printf("Pipeline execution failed: %v\n", err)
    } else {
        fmt.Printf("Final pipeline result: %s\n", finalResult)
    }

    fmt.Println("\n=== Composition Principles Summary ===")
    fmt.Println("1. Group by behavior (what things DO), not identity (what things ARE)")
    fmt.Println("2. Use interfaces to define behavior contracts")
    fmt.Println("3. Components can implement multiple interfaces")
    fmt.Println("4. Automatic behavior detection through type assertions")
    fmt.Println("5. Composition enables flexible system architectures")
    fmt.Println("6. Same component can participate in multiple behavior groups")
    fmt.Println("7. Pipeline pattern demonstrates sequential composition")
}
```

**Prerequisites**: Module 20
