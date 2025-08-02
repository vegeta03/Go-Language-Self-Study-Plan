# Module 17: Methods Part 3: Function Variables and Method Sets

**Duration**: 1 hour (45 min theory + 15 min hands-on)

**Learning Objectives**:

- Understand methods as syntactic sugar over functions
- Master function variables and method values
- Learn about method sets and type behavior promotion
- Understand the cost of decoupling through function variables
- Apply function variables in automation callback patterns

**Videos Covered**:

- 4.4 Methods Part 3 (Function Method Variables) (0:13:40)

**Key Concepts**:

- Methods are syntactic sugar: receiver is the first parameter
- Function variables: methods can be assigned to variables
- Method values vs method expressions
- Semantic consistency prevents mixing value/pointer semantics
- Function variables create indirection and potential allocations
- Named types don't inherit behavior from underlying types
- Decoupling code and data through function variables

**Hands-on Exercise 1: Methods as Syntactic Sugar and Semantic Consistency**:

```go
// Demonstrating methods as syntactic sugar and semantic consistency
package main

import (
    "fmt"
    "log"
)

// AutomationTask represents a task in an automation system
type AutomationTask struct {
    ID       string
    Name     string
    Priority int
    Status   string
}

// Value receiver - read-only operation (value semantics)
func (at AutomationTask) GetDescription() string {
    return fmt.Sprintf("Task %s: %s (Priority: %d, Status: %s)",
        at.ID, at.Name, at.Priority, at.Status)
}

// Pointer receiver - modifies state (pointer semantics)
func (at *AutomationTask) UpdateStatus(newStatus string) {
    at.Status = newStatus
    log.Printf("Task %s status updated to: %s", at.ID, newStatus)
}

// Value receiver - returns new task (immutable pattern)
func (at AutomationTask) WithHigherPriority() AutomationTask {
    newTask := at // Copy the task (value semantics)
    newTask.Priority = at.Priority + 1
    return newTask
}

// Demonstrate semantic consistency issues
func demonstrateSemanticConsistency() {
    fmt.Println("=== Semantic Consistency Demonstration ===")

    // Create a slice of tasks (value semantics for collection)
    tasks := []AutomationTask{
        {ID: "T001", Name: "Database Backup", Priority: 5, Status: "pending"},
        {ID: "T002", Name: "Log Rotation", Priority: 3, Status: "pending"},
    }

    fmt.Println("Original tasks:")
    for _, task := range tasks {
        fmt.Printf("  %s\n", task.GetDescription())
    }

    // WRONG: Mixing semantics - using pointer method on copy
    fmt.Println("\n=== Semantic Inconsistency (WRONG) ===")
    for _, task := range tasks { // task is a COPY
        // This won't work as expected - we're modifying the copy!
        task.UpdateStatus("processing") // Pointer method on copy
        fmt.Printf("  After update: %s\n", task.GetDescription())
    }

    fmt.Println("\nOriginal tasks after 'update' (unchanged!):")
    for _, task := range tasks {
        fmt.Printf("  %s\n", task.GetDescription())
    }

    // CORRECT: Consistent pointer semantics
    fmt.Println("\n=== Semantic Consistency (CORRECT) ===")
    for i := range tasks { // Use index to get pointer to original
        tasks[i].UpdateStatus("processing") // Pointer method on original
        fmt.Printf("  After update: %s\n", tasks[i].GetDescription())
    }

    fmt.Println("\nOriginal tasks after correct update:")
    for _, task := range tasks {
        fmt.Printf("  %s\n", task.GetDescription())
    }
}

// Demonstrate that methods are syntactic sugar
func demonstrateMethodsAsSyntacticSugar() {
    fmt.Println("\n=== Methods as Syntactic Sugar ===")

    task := AutomationTask{
        ID:       "T003",
        Name:     "System Health Check",
        Priority: 2,
        Status:   "ready",
    }

    // Normal method call (syntactic sugar)
    fmt.Printf("Method call: %s\n", task.GetDescription())

    // What actually happens (don't do this in real code!)
    // The receiver becomes the first parameter
    fmt.Printf("Function call: %s\n", AutomationTask.GetDescription(task))

    // For pointer methods
    task.UpdateStatus("running")

    // What actually happens for pointer methods (don't do this!)
    (*AutomationTask).UpdateStatus(&task, "completed")

    fmt.Printf("Final task: %s\n", task.GetDescription())
}

func main() {
    demonstrateSemanticConsistency()
    demonstrateMethodsAsSyntacticSugar()
}
```

**Hands-on Exercise 2: Function Variables and Method Values vs Expressions**:

```go
// Function variables, method values, and method expressions in automation
package main

import (
    "fmt"
    "log"
    "time"
)

// DataProcessor represents a data processing component
type DataProcessor struct {
    Name      string
    BatchSize int
    Active    bool
}

// Value receiver - read-only operation
func (dp DataProcessor) GetStatus() string {
    status := "inactive"
    if dp.Active {
        status = "active"
    }
    return fmt.Sprintf("Processor %s: %s (batch size: %d)",
        dp.Name, status, dp.BatchSize)
}

// Pointer receiver - modifies state
func (dp *DataProcessor) Activate() {
    dp.Active = true
    log.Printf("Activated processor: %s", dp.Name)
}

// Pointer receiver - processes data
func (dp *DataProcessor) ProcessBatch(data []string) error {
    if !dp.Active {
        return fmt.Errorf("processor %s is not active", dp.Name)
    }

    log.Printf("Processing %d items with %s", len(data), dp.Name)
    time.Sleep(50 * time.Millisecond) // Simulate processing
    return nil
}

// ProcessorFunc represents a function that can process data
type ProcessorFunc func([]string) error

// StatusFunc represents a function that returns status
type StatusFunc func() string

func demonstrateMethodValues() {
    fmt.Println("=== Method Values (Bound to Instance) ===")

    processor := DataProcessor{
        Name:      "MainProcessor",
        BatchSize: 100,
        Active:    false,
    }

    // Method value - binds method to specific processor instance
    // Value semantics: creates a copy of processor for the method
    getStatus := processor.GetStatus // No parentheses!
    fmt.Printf("Method value result: %s\n", getStatus())

    // Pointer method value - binds to the original instance
    activate := processor.Activate // Binds to processor instance
    activate()                     // Activates the original processor

    // Now check status again - should show activated
    fmt.Printf("After activation: %s\n", getStatus())

    // Important: getStatus still operates on the COPY (value semantics)
    // So it won't show the activation change!
    fmt.Printf("Status from method value (copy): %s\n", getStatus())
    fmt.Printf("Status from direct call (original): %s\n", processor.GetStatus())
}

func demonstrateMethodExpressions() {
    fmt.Println("\n=== Method Expressions (Unbound Functions) ===")

    processor1 := DataProcessor{Name: "Processor1", BatchSize: 50, Active: true}
    processor2 := DataProcessor{Name: "Processor2", BatchSize: 200, Active: false}

    // Method expression - requires passing receiver explicitly
    var getStatusExpr func(DataProcessor) string = DataProcessor.GetStatus

    fmt.Printf("Processor1 via expression: %s\n", getStatusExpr(processor1))
    fmt.Printf("Processor2 via expression: %s\n", getStatusExpr(processor2))

    // Pointer method expression
    var activateExpr func(*DataProcessor) = (*DataProcessor).Activate

    fmt.Println("\nActivating processor2 via expression:")
    activateExpr(&processor2) // Must pass pointer explicitly
    fmt.Printf("Processor2 after activation: %s\n", getStatusExpr(processor2))
}

func demonstrateFunctionVariables() {
    fmt.Println("\n=== Function Variables for Decoupling ===")

    processors := []DataProcessor{
        {Name: "FastProcessor", BatchSize: 50, Active: true},
        {Name: "SlowProcessor", BatchSize: 200, Active: true},
    }

    // Create function variables for different processing strategies
    var strategies []ProcessorFunc

    for i := range processors {
        // Method value - binds ProcessBatch to specific processor
        strategy := processors[i].ProcessBatch
        strategies = append(strategies, strategy)
    }

    // Use the strategies
    testData := []string{"item1", "item2", "item3"}

    for i, strategy := range strategies {
        fmt.Printf("\nUsing strategy %d:\n", i+1)
        if err := strategy(testData); err != nil {
            fmt.Printf("Error: %v\n", err)
        }
    }
}

func demonstrateDecouplingCost() {
    fmt.Println("\n=== Cost of Decoupling ===")

    processor := DataProcessor{
        Name:      "CostDemo",
        BatchSize: 100,
        Active:    true,
    }

    // Direct method call - no allocation
    fmt.Println("Direct call:")
    fmt.Printf("  %s\n", processor.GetStatus())

    // Function variable - causes allocation due to indirection
    fmt.Println("Function variable (with allocation cost):")
    statusFunc := processor.GetStatus // Creates copy + allocation
    fmt.Printf("  %s\n", statusFunc())

    // Pointer method - still has allocation cost
    activateFunc := processor.Activate // Points to original but still allocates
    fmt.Println("Pointer method variable (allocation for indirection):")
    activateFunc() // This works on original due to pointer semantics

    fmt.Printf("Final status: %s\n", processor.GetStatus())
}

func main() {
    demonstrateMethodValues()
    demonstrateMethodExpressions()
    demonstrateFunctionVariables()
    demonstrateDecouplingCost()
}
```

**Hands-on Exercise 3: Type Behavior Promotion and Decoupling Patterns**:

```go
// Named types, behavior promotion, and automation system decoupling
package main

import (
    "fmt"
    "log"
    "time"
)

// BaseProcessor represents a basic automation processor
type BaseProcessor struct {
    ID       string
    Name     string
    Priority int
    Active   bool
}

// Value receiver - read-only operation
func (bp BaseProcessor) GetInfo() string {
    status := "inactive"
    if bp.Active {
        status = "active"
    }
    return fmt.Sprintf("Processor %s (%s): %s, Priority: %d",
        bp.ID, bp.Name, status, bp.Priority)
}

// Pointer receiver - modifies state
func (bp *BaseProcessor) Activate() {
    bp.Active = true
    log.Printf("Activated processor: %s", bp.Name)
}

// Pointer receiver - processes work
func (bp *BaseProcessor) Process(workItem string) error {
    if !bp.Active {
        return fmt.Errorf("processor %s is not active", bp.Name)
    }

    log.Printf("Processing %s with %s", workItem, bp.Name)
    time.Sleep(100 * time.Millisecond)
    return nil
}

// SpecializedProcessor - new named type based on BaseProcessor
// IMPORTANT: This does NOT inherit behavior from BaseProcessor
type SpecializedProcessor BaseProcessor

// SpecializedProcessor has NO methods from BaseProcessor
// We must define methods explicitly if needed

func (sp SpecializedProcessor) GetSpecializedInfo() string {
    // We can access fields (memory model promotion)
    return fmt.Sprintf("Specialized Processor: %s (ID: %s)", sp.Name, sp.ID)
}

// If we want BaseProcessor behavior, we must convert
func (sp SpecializedProcessor) GetBaseInfo() string {
    // Convert to underlying type to access its methods
    base := BaseProcessor(sp)
    return base.GetInfo()
}

// ProcessorStrategy represents different processing strategies
type ProcessorStrategy interface {
    Execute(workItem string) error
    GetName() string
}

// StrategyFunc is a function type that implements ProcessorStrategy
type StrategyFunc struct {
    name string
    fn   func(string) error
}

func (sf StrategyFunc) Execute(workItem string) error {
    return sf.fn(workItem)
}

func (sf StrategyFunc) GetName() string {
    return sf.name
}

// AutomationOrchestrator demonstrates decoupling with function variables
type AutomationOrchestrator struct {
    Name       string
    Processors []BaseProcessor
    Strategy   ProcessorStrategy // Interface for maximum flexibility
}

// CreateProcessorStrategy creates a strategy from a specific processor
func (ao *AutomationOrchestrator) CreateProcessorStrategy(processorID string) ProcessorStrategy {
    // Find the processor
    for i := range ao.Processors {
        if ao.Processors[i].ID == processorID {
            processor := &ao.Processors[i]

            // Return a strategy that uses the processor's method
            return StrategyFunc{
                name: fmt.Sprintf("Strategy-%s", processor.Name),
                fn:   processor.Process, // Method value - decoupling!
            }
        }
    }

    // Return default strategy
    return StrategyFunc{
        name: "DefaultStrategy",
        fn: func(workItem string) error {
            return fmt.Errorf("no processor found for work item: %s", workItem)
        },
    }
}

// ExecuteWork processes work items using the configured strategy
func (ao *AutomationOrchestrator) ExecuteWork(workItems []string) {
    if ao.Strategy == nil {
        log.Println("No strategy configured")
        return
    }

    fmt.Printf("Using strategy: %s\n", ao.Strategy.GetName())

    for _, item := range workItems {
        if err := ao.Strategy.Execute(item); err != nil {
            log.Printf("Error processing %s: %v", item, err)
        }
    }
}

func demonstrateBehaviorPromotion() {
    fmt.Println("=== Behavior Promotion (or lack thereof) ===")

    // Create base processor
    base := BaseProcessor{
        ID:       "BP001",
        Name:     "BaseProcessor",
        Priority: 5,
        Active:   true,
    }

    fmt.Printf("Base processor: %s\n", base.GetInfo())

    // Create specialized processor from base
    specialized := SpecializedProcessor{
        ID:       "SP001",
        Name:     "SpecialProcessor",
        Priority: 10,
        Active:   true,
    }

    fmt.Printf("Specialized info: %s\n", specialized.GetSpecializedInfo())

    // This would NOT compile - no inherited behavior:
    // fmt.Printf("Specialized GetInfo: %s\n", specialized.GetInfo())

    // But we can convert to access base behavior
    fmt.Printf("Specialized as base: %s\n", specialized.GetBaseInfo())

    // Or explicit conversion
    convertedBase := BaseProcessor(specialized)
    fmt.Printf("Explicit conversion: %s\n", convertedBase.GetInfo())
}

func demonstrateDecouplingPatterns() {
    fmt.Println("\n=== Decoupling Patterns in Automation ===")

    // Create processors
    processors := []BaseProcessor{
        {ID: "P001", Name: "FastProcessor", Priority: 1, Active: true},
        {ID: "P002", Name: "SlowProcessor", Priority: 2, Active: true},
        {ID: "P003", Name: "BackupProcessor", Priority: 3, Active: false},
    }

    // Create orchestrator
    orchestrator := AutomationOrchestrator{
        Name:       "MainOrchestrator",
        Processors: processors,
    }

    workItems := []string{"Task-A", "Task-B", "Task-C"}

    // Strategy 1: Use fast processor
    fmt.Println("\n--- Using Fast Processor Strategy ---")
    orchestrator.Strategy = orchestrator.CreateProcessorStrategy("P001")
    orchestrator.ExecuteWork(workItems)

    // Strategy 2: Use slow processor
    fmt.Println("\n--- Using Slow Processor Strategy ---")
    orchestrator.Strategy = orchestrator.CreateProcessorStrategy("P002")
    orchestrator.ExecuteWork(workItems)

    // Strategy 3: Use non-existent processor
    fmt.Println("\n--- Using Non-existent Processor Strategy ---")
    orchestrator.Strategy = orchestrator.CreateProcessorStrategy("P999")
    orchestrator.ExecuteWork([]string{"Task-D"})

    // Strategy 4: Custom strategy
    fmt.Println("\n--- Using Custom Strategy ---")
    orchestrator.Strategy = StrategyFunc{
        name: "CustomStrategy",
        fn: func(workItem string) error {
            log.Printf("Custom processing for: %s", workItem)
            return nil
        },
    }
    orchestrator.ExecuteWork([]string{"Custom-Task"})
}

func demonstrateAllocationCosts() {
    fmt.Println("\n=== Allocation Costs of Decoupling ===")

    processor := BaseProcessor{
        ID:       "COST001",
        Name:     "CostAnalysis",
        Priority: 1,
        Active:   true,
    }

    // Direct method call - no allocation
    fmt.Println("Direct method call (no allocation):")
    err := processor.Process("DirectItem")
    if err != nil {
        fmt.Printf("Error: %v\n", err)
    }

    // Function variable - allocation due to indirection
    fmt.Println("Function variable (allocation cost):")
    processFunc := processor.Process // Method value - allocation!
    err = processFunc("IndirectItem")
    if err != nil {
        fmt.Printf("Error: %v\n", err)
    }

    fmt.Println("\nNote: Function variables provide decoupling but at the cost of:")
    fmt.Println("- Heap allocation (escape analysis)")
    fmt.Println("- Double indirection to reach code")
    fmt.Println("- Runtime overhead")
    fmt.Println("Use only when the decoupling value justifies the cost!")
}

func main() {
    demonstrateBehaviorPromotion()
    demonstrateDecouplingPatterns()
    demonstrateAllocationCosts()
}
```

**Prerequisites**: Module 16
