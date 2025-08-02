# Module 38: Modern Go Development Practices and Tooling

**Duration**: 1 hour (45 min theory + 15 min hands-on)

**Learning Objectives**:

- Master modern Go development tooling and practices (Go 1.24+)
- Understand Go workspaces and multi-module development
- Learn advanced testing patterns with `testing/synctest` and `testing.B.Loop`
- Apply FIPS 140-3 compliance and security best practices
- Implement modern CI/CD pipelines with Go tooling
- Design maintainable automation systems with contemporary Go patterns

**Videos Covered**:

- Modern Go tooling and development practices (Go 1.24+)
- Contemporary development patterns and best practices
- Advanced security and compliance features

**Key Concepts**:

- Go workspaces: multi-module development with `go.work` files
- Tool directives: managing executable dependencies in `go.mod`
- Advanced testing: `testing/synctest` for concurrent code testing
- Benchmark improvements: `testing.B.Loop` for more accurate benchmarks
- FIPS 140-3 compliance: cryptographic module usage and validation
- Security enhancements: `crypto/subtle.WithDataIndependentTiming`
- Modern package patterns: directory-limited filesystem access with `os.Root`
- Performance optimizations: Swiss Tables maps and improved runtime

**Hands-on Exercise 1: Modern Go Workspace and Tooling**:

Setting up modern Go development environment with workspaces and tool management:

```go
// Modern Go workspace setup and tool management
package workspace

import (
    "context"
    "fmt"
    "os"
    "os/exec"
    "path/filepath"
    "strings"
)

// WorkspaceManager manages Go workspaces and tools
type WorkspaceManager struct {
    rootDir     string
    workspaceFile string
    modules     []string
    tools       []string
}

// NewWorkspaceManager creates a new workspace manager
func NewWorkspaceManager(rootDir string) *WorkspaceManager {
    return &WorkspaceManager{
        rootDir:       rootDir,
        workspaceFile: filepath.Join(rootDir, "go.work"),
    }
}

// InitializeWorkspace creates a new Go workspace
func (wm *WorkspaceManager) InitializeWorkspace(ctx context.Context) error {
    // Create workspace directory
    if err := os.MkdirAll(wm.rootDir, 0755); err != nil {
        return fmt.Errorf("failed to create workspace directory: %w", err)
    }
    
    // Initialize go.work file
    cmd := exec.CommandContext(ctx, "go", "work", "init")
    cmd.Dir = wm.rootDir
    
    if output, err := cmd.CombinedOutput(); err != nil {
        return fmt.Errorf("failed to initialize workspace: %w\nOutput: %s", err, output)
    }
    
    fmt.Printf("Initialized Go workspace at %s\n", wm.rootDir)
    return nil
}

// AddModule adds a module to the workspace
func (wm *WorkspaceManager) AddModule(ctx context.Context, modulePath string) error {
    // Create module directory
    moduleDir := filepath.Join(wm.rootDir, modulePath)
    if err := os.MkdirAll(moduleDir, 0755); err != nil {
        return fmt.Errorf("failed to create module directory: %w", err)
    }
    
    // Initialize module
    cmd := exec.CommandContext(ctx, "go", "mod", "init", modulePath)
    cmd.Dir = moduleDir
    
    if output, err := cmd.CombinedOutput(); err != nil {
        return fmt.Errorf("failed to initialize module: %w\nOutput: %s", err, output)
    }
    
    // Add module to workspace
    cmd = exec.CommandContext(ctx, "go", "work", "use", modulePath)
    cmd.Dir = wm.rootDir
    
    if output, err := cmd.CombinedOutput(); err != nil {
        return fmt.Errorf("failed to add module to workspace: %w\nOutput: %s", err, output)
    }
    
    wm.modules = append(wm.modules, modulePath)
    fmt.Printf("Added module %s to workspace\n", modulePath)
    return nil
}

// AddTool adds a tool dependency to a module
func (wm *WorkspaceManager) AddTool(ctx context.Context, modulePath, toolPackage string) error {
    moduleDir := filepath.Join(wm.rootDir, modulePath)
    
    // Add tool directive to go.mod
    cmd := exec.CommandContext(ctx, "go", "get", "-tool", toolPackage)
    cmd.Dir = moduleDir
    
    if output, err := cmd.CombinedOutput(); err != nil {
        return fmt.Errorf("failed to add tool: %w\nOutput: %s", err, output)
    }
    
    wm.tools = append(wm.tools, toolPackage)
    fmt.Printf("Added tool %s to module %s\n", toolPackage, modulePath)
    return nil
}

// RunTool executes a tool from the workspace
func (wm *WorkspaceManager) RunTool(ctx context.Context, toolName string, args ...string) error {
    cmd := exec.CommandContext(ctx, "go", "tool", toolName)
    cmd.Args = append(cmd.Args, args...)
    cmd.Dir = wm.rootDir
    cmd.Stdout = os.Stdout
    cmd.Stderr = os.Stderr
    
    if err := cmd.Run(); err != nil {
        return fmt.Errorf("failed to run tool %s: %w", toolName, err)
    }
    
    return nil
}

// BuildWithJSON builds modules with JSON output
func (wm *WorkspaceManager) BuildWithJSON(ctx context.Context, modulePath string) error {
    moduleDir := filepath.Join(wm.rootDir, modulePath)
    
    cmd := exec.CommandContext(ctx, "go", "build", "-json", "./...")
    cmd.Dir = moduleDir
    cmd.Stdout = os.Stdout
    cmd.Stderr = os.Stderr
    
    if err := cmd.Run(); err != nil {
        return fmt.Errorf("failed to build module with JSON output: %w", err)
    }
    
    return nil
}

// UpdateTools updates all tools in the workspace
func (wm *WorkspaceManager) UpdateTools(ctx context.Context) error {
    for _, module := range wm.modules {
        moduleDir := filepath.Join(wm.rootDir, module)
        
        // Update all tools
        cmd := exec.CommandContext(ctx, "go", "get", "tool")
        cmd.Dir = moduleDir
        
        if output, err := cmd.CombinedOutput(); err != nil {
            return fmt.Errorf("failed to update tools in %s: %w\nOutput: %s", 
                             module, err, output)
        }
        
        fmt.Printf("Updated tools in module %s\n", module)
    }
    
    return nil
}

// GenerateWorkspaceInfo generates information about the workspace
func (wm *WorkspaceManager) GenerateWorkspaceInfo() WorkspaceInfo {
    return WorkspaceInfo{
        RootDir:   wm.rootDir,
        Modules:   append([]string(nil), wm.modules...),
        Tools:     append([]string(nil), wm.tools...),
        GoVersion: wm.getGoVersion(),
    }
}

// WorkspaceInfo contains information about the workspace
type WorkspaceInfo struct {
    RootDir   string   `json:"root_dir"`
    Modules   []string `json:"modules"`
    Tools     []string `json:"tools"`
    GoVersion string   `json:"go_version"`
}

// getGoVersion returns the current Go version
func (wm *WorkspaceManager) getGoVersion() string {
    cmd := exec.Command("go", "version")
    output, err := cmd.Output()
    if err != nil {
        return "unknown"
    }
    
    return strings.TrimSpace(string(output))
}
```

**Hands-on Exercise 2: Advanced Testing with Modern Patterns**:

Implementing advanced testing patterns using Go 1.24+ features:

```go
// Advanced testing patterns with modern Go features
package testing_modern

import (
    "context"
    "sync"
    "testing"
    "testing/synctest"
    "time"
)

// AutomationService represents a service that needs concurrent testing
type AutomationService struct {
    name     string
    workers  int
    jobs     chan Job
    results  chan Result
    done     chan struct{}
    mu       sync.RWMutex
    running  bool
}

// Job represents work to be processed
type Job struct {
    ID   string
    Data interface{}
}

// Result represents the result of processing a job
type Result struct {
    JobID string
    Data  interface{}
    Error error
}

// NewAutomationService creates a new automation service
func NewAutomationService(name string, workers int) *AutomationService {
    return &AutomationService{
        name:    name,
        workers: workers,
        jobs:    make(chan Job, workers*2),
        results: make(chan Result, workers*2),
        done:    make(chan struct{}),
    }
}

// Start starts the automation service
func (as *AutomationService) Start(ctx context.Context) error {
    as.mu.Lock()
    defer as.mu.Unlock()
    
    if as.running {
        return fmt.Errorf("service already running")
    }
    
    as.running = true
    
    // Start workers
    for i := 0; i < as.workers; i++ {
        go as.worker(ctx, i)
    }
    
    return nil
}

// Stop stops the automation service
func (as *AutomationService) Stop() {
    as.mu.Lock()
    defer as.mu.Unlock()
    
    if !as.running {
        return
    }
    
    as.running = false
    close(as.done)
    close(as.jobs)
}

// SubmitJob submits a job for processing
func (as *AutomationService) SubmitJob(job Job) bool {
    as.mu.RLock()
    defer as.mu.RUnlock()
    
    if !as.running {
        return false
    }
    
    select {
    case as.jobs <- job:
        return true
    default:
        return false
    }
}

// GetResult gets a result from processing
func (as *AutomationService) GetResult(ctx context.Context) (Result, error) {
    select {
    case result := <-as.results:
        return result, nil
    case <-ctx.Done():
        return Result{}, ctx.Err()
    case <-as.done:
        return Result{}, fmt.Errorf("service stopped")
    }
}

// worker processes jobs
func (as *AutomationService) worker(ctx context.Context, id int) {
    for {
        select {
        case job, ok := <-as.jobs:
            if !ok {
                return
            }
            
            // Simulate processing time
            time.Sleep(10 * time.Millisecond)
            
            result := Result{
                JobID: job.ID,
                Data:  fmt.Sprintf("processed_%s_by_worker_%d", job.ID, id),
            }
            
            select {
            case as.results <- result:
            case <-ctx.Done():
                return
            case <-as.done:
                return
            }
            
        case <-ctx.Done():
            return
        case <-as.done:
            return
        }
    }
}

// TestAutomationServiceConcurrency tests concurrent behavior using synctest
func TestAutomationServiceConcurrency(t *testing.T) {
    // Use synctest for deterministic concurrent testing
    synctest.Run(func() {
        ctx := context.Background()
        service := NewAutomationService("test-service", 3)
        
        // Start the service
        if err := service.Start(ctx); err != nil {
            t.Fatalf("Failed to start service: %v", err)
        }
        defer service.Stop()
        
        // Submit multiple jobs concurrently
        jobs := []Job{
            {ID: "job1", Data: "data1"},
            {ID: "job2", Data: "data2"},
            {ID: "job3", Data: "data3"},
        }
        
        var wg sync.WaitGroup
        
        // Submit jobs
        for _, job := range jobs {
            wg.Add(1)
            go func(j Job) {
                defer wg.Done()
                if !service.SubmitJob(j) {
                    t.Errorf("Failed to submit job %s", j.ID)
                }
            }(job)
        }
        
        // Wait for all jobs to be submitted
        synctest.Wait()
        wg.Wait()
        
        // Collect results
        results := make(map[string]Result)
        for i := 0; i < len(jobs); i++ {
            result, err := service.GetResult(ctx)
            if err != nil {
                t.Fatalf("Failed to get result: %v", err)
            }
            results[result.JobID] = result
        }
        
        // Verify all jobs were processed
        for _, job := range jobs {
            if _, exists := results[job.ID]; !exists {
                t.Errorf("Job %s was not processed", job.ID)
            }
        }
    })
}

// BenchmarkAutomationServiceThroughput benchmarks service throughput using B.Loop
func BenchmarkAutomationServiceThroughput(b *testing.B) {
    ctx := context.Background()
    service := NewAutomationService("bench-service", 4)
    
    if err := service.Start(ctx); err != nil {
        b.Fatalf("Failed to start service: %v", err)
    }
    defer service.Stop()
    
    // Use the new B.Loop method for more accurate benchmarking
    for b.Loop() {
        job := Job{
            ID:   fmt.Sprintf("job_%d", b.N),
            Data: "benchmark_data",
        }
        
        if !service.SubmitJob(job) {
            b.Fatal("Failed to submit job")
        }
        
        _, err := service.GetResult(ctx)
        if err != nil {
            b.Fatalf("Failed to get result: %v", err)
        }
    }
}

// TestWithContext demonstrates testing with context and cleanup
func TestWithContext(t *testing.T) {
    // Use the new T.Context method
    ctx := t.Context()
    
    service := NewAutomationService("context-test", 2)
    
    if err := service.Start(ctx); err != nil {
        t.Fatalf("Failed to start service: %v", err)
    }
    
    // The service will be automatically stopped when the test context is canceled
    t.Cleanup(func() {
        service.Stop()
    })
    
    // Test operations with context
    job := Job{ID: "context_job", Data: "test_data"}
    
    if !service.SubmitJob(job) {
        t.Fatal("Failed to submit job")
    }
    
    result, err := service.GetResult(ctx)
    if err != nil {
        t.Fatalf("Failed to get result: %v", err)
    }
    
    if result.JobID != job.ID {
        t.Errorf("Expected job ID %s, got %s", job.ID, result.JobID)
    }
}

// TestWithChdir demonstrates testing with directory changes
func TestWithChdir(t *testing.T) {
    // Use the new T.Chdir method to change working directory for the test
    t.Chdir("/tmp")
    
    // Test operations that depend on working directory
    cwd, err := os.Getwd()
    if err != nil {
        t.Fatalf("Failed to get working directory: %v", err)
    }
    
    if cwd != "/tmp" {
        t.Errorf("Expected working directory /tmp, got %s", cwd)
    }
    
    // Working directory will be restored automatically after the test
}
```

**Hands-on Exercise 3: Security and Performance with Modern Go**:

Implementing security best practices and performance optimizations with Go 1.24+ features:

```go
// Security and performance with modern Go features
package security_performance

import (
    "crypto/rand"
    "crypto/subtle"
    "fmt"
    "os"
    "runtime"
    "sync"
    "time"
    "unsafe"
)

// SecureAutomationProcessor demonstrates FIPS 140-3 compliance and security features
type SecureAutomationProcessor struct {
    name           string
    fipsMode       bool
    dataIndepTiming bool
    root           *os.Root
    mu             sync.RWMutex
    processedCount uint64
}

// NewSecureProcessor creates a new secure automation processor
func NewSecureProcessor(name, rootDir string, fipsMode bool) (*SecureAutomationProcessor, error) {
    // Use directory-limited filesystem access
    root, err := os.OpenRoot(rootDir)
    if err != nil {
        return nil, fmt.Errorf("failed to open root directory: %w", err)
    }
    
    processor := &SecureAutomationProcessor{
        name:            name,
        fipsMode:        fipsMode,
        dataIndepTiming: true, // Enable data-independent timing by default
        root:            root,
    }
    
    // Configure FIPS mode if requested
    if fipsMode {
        os.Setenv("GOFIPS140", "v1.0.0")
        // Note: In real applications, you'd also set the runtime GODEBUG setting
    }
    
    return processor, nil
}

// ProcessSecureData processes data with security considerations
func (sap *SecureAutomationProcessor) ProcessSecureData(data []byte, key []byte) ([]byte, error) {
    sap.mu.Lock()
    defer sap.mu.Unlock()
    
    // Use data-independent timing for cryptographic operations
    return subtle.WithDataIndependentTiming(func() ([]byte, error) {
        // Generate secure random nonce
        nonce, err := sap.generateSecureRandom(12)
        if err != nil {
            return nil, fmt.Errorf("failed to generate nonce: %w", err)
        }
        
        // Perform secure XOR operation (simplified example)
        result := make([]byte, len(data))
        keyLen := len(key)
        
        for i, b := range data {
            result[i] = b ^ key[i%keyLen] ^ nonce[i%len(nonce)]
        }
        
        // Use constant-time comparison for validation
        if subtle.ConstantTimeCompare(data[:min(len(data), 4)], []byte("test")) == 1 {
            // Special handling for test data
            copy(result[:4], []byte("proc"))
        }
        
        sap.processedCount++
        return result, nil
    })
}

// generateSecureRandom generates cryptographically secure random bytes
func (sap *SecureAutomationProcessor) generateSecureRandom(size int) ([]byte, error) {
    // Use crypto/rand.Text for secure random text generation (Go 1.24+)
    return rand.Text("abcdefghijklmnopqrstuvwxyz0123456789", size)
}

// SecureFileOperation performs file operations within the restricted root
func (sap *SecureAutomationProcessor) SecureFileOperation(filename string, data []byte) error {
    // All file operations are restricted to the root directory
    file, err := sap.root.Create(filename)
    if err != nil {
        return fmt.Errorf("failed to create file: %w", err)
    }
    defer file.Close()
    
    _, err = file.Write(data)
    if err != nil {
        return fmt.Errorf("failed to write data: %w", err)
    }
    
    return nil
}

// ReadSecureFile reads a file securely within the root directory
func (sap *SecureAutomationProcessor) ReadSecureFile(filename string) ([]byte, error) {
    file, err := sap.root.Open(filename)
    if err != nil {
        return nil, fmt.Errorf("failed to open file: %w", err)
    }
    defer file.Close()
    
    stat, err := file.Stat()
    if err != nil {
        return nil, fmt.Errorf("failed to stat file: %w", err)
    }
    
    data := make([]byte, stat.Size())
    _, err = file.Read(data)
    if err != nil {
        return nil, fmt.Errorf("failed to read file: %w", err)
    }
    
    return data, nil
}

// GetProcessingStats returns processing statistics
func (sap *SecureAutomationProcessor) GetProcessingStats() ProcessingStats {
    sap.mu.RLock()
    defer sap.mu.RUnlock()
    
    var memStats runtime.MemStats
    runtime.ReadMemStats(&memStats)
    
    return ProcessingStats{
        ProcessedCount:   sap.processedCount,
        MemoryAllocated:  memStats.TotalAlloc,
        GoroutineCount:   runtime.NumGoroutine(),
        FIPSMode:         sap.fipsMode,
        DataIndepTiming:  sap.dataIndepTiming,
    }
}

// ProcessingStats contains processing statistics
type ProcessingStats struct {
    ProcessedCount  uint64 `json:"processed_count"`
    MemoryAllocated uint64 `json:"memory_allocated"`
    GoroutineCount  int    `json:"goroutine_count"`
    FIPSMode        bool   `json:"fips_mode"`
    DataIndepTiming bool   `json:"data_independent_timing"`
}

// HighPerformanceProcessor demonstrates performance optimizations
type HighPerformanceProcessor struct {
    name        string
    swissMap    map[string]interface{} // Uses Swiss Tables implementation
    cleanupFuncs []func()
    mu          sync.RWMutex
}

// NewHighPerformanceProcessor creates a new high-performance processor
func NewHighPerformanceProcessor(name string) *HighPerformanceProcessor {
    return &HighPerformanceProcessor{
        name:     name,
        swissMap: make(map[string]interface{}), // Automatically uses Swiss Tables
    }
}

// ProcessWithCleanup processes data with automatic cleanup
func (hpp *HighPerformanceProcessor) ProcessWithCleanup(key string, data interface{}) error {
    hpp.mu.Lock()
    defer hpp.mu.Unlock()
    
    // Store data in high-performance map
    hpp.swissMap[key] = data
    
    // Add cleanup function using the new runtime.AddCleanup
    cleanup := runtime.AddCleanup(data, func() {
        hpp.mu.Lock()
        defer hpp.mu.Unlock()
        delete(hpp.swissMap, key)
        fmt.Printf("Cleaned up data for key: %s\n", key)
    })
    
    hpp.cleanupFuncs = append(hpp.cleanupFuncs, cleanup)
    
    return nil
}

// GetData retrieves data from the high-performance map
func (hpp *HighPerformanceProcessor) GetData(key string) (interface{}, bool) {
    hpp.mu.RLock()
    defer hpp.mu.RUnlock()
    
    value, exists := hpp.swissMap[key]
    return value, exists
}

// BenchmarkMapPerformance benchmarks map performance
func (hpp *HighPerformanceProcessor) BenchmarkMapPerformance(iterations int) time.Duration {
    start := time.Now()
    
    for i := 0; i < iterations; i++ {
        key := fmt.Sprintf("key_%d", i)
        hpp.swissMap[key] = i
        
        if value, exists := hpp.swissMap[key]; !exists || value != i {
            panic("map operation failed")
        }
        
        delete(hpp.swissMap, key)
    }
    
    return time.Since(start)
}

// Close cleans up resources
func (hpp *HighPerformanceProcessor) Close() {
    hpp.mu.Lock()
    defer hpp.mu.Unlock()
    
    // Cleanup functions will be called automatically when objects are GC'd
    // This is just for demonstration
    for _, cleanup := range hpp.cleanupFuncs {
        if cleanup != nil {
            cleanup()
        }
    }
    hpp.cleanupFuncs = nil
}

// min returns the minimum of two integers
func min(a, b int) int {
    if a < b {
        return a
    }
    return b
}

// UnsafeOptimization demonstrates unsafe optimizations (use with caution)
func UnsafeOptimization(data []byte) string {
    // Convert []byte to string without allocation (unsafe but efficient)
    // This is safe only if the string is not modified and the byte slice is not reused
    return *(*string)(unsafe.Pointer(&data))
}

// SafeStringConversion provides a safe alternative
func SafeStringConversion(data []byte) string {
    // Safe conversion that creates a copy
    return string(data)
}
```

**Prerequisites**: Sessions 1-37 (Complete Go training program)

**Course Summary**: This session completes the comprehensive Go training program, covering modern development practices, advanced tooling, security considerations, and performance optimizations. Participants now have the complete knowledge and skills needed to build professional-grade automation systems using Go.
