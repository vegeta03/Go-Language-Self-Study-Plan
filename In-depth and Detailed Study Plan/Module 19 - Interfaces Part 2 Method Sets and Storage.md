# Module 19: Interfaces Part 2: Method Sets and Storage

**Duration**: 1 hour (45 min theory + 15 min hands-on)

**Learning Objectives**:

- Understand method sets and interface satisfaction rules
- Master interface value storage mechanics (two-word structure)
- Learn address-of-value vs value semantics with interfaces
- Understand interface conversion and type assertions
- Apply interface storage patterns in automation systems

**Videos Covered**:

- 4.6 Interfaces Part 2 (Method Sets) (0:11:51)
- 4.7 Interfaces Part 3 (Storage by Value) (0:05:34)

**Key Concepts**:

- Method sets determine interface satisfaction
- Interface values: two-word structure (type info + data pointer)
- Value vs pointer method sets and interface compatibility
- Storage by value: interfaces store copies of concrete data
- Type assertions and interface conversions
- Performance implications of interface storage

**Hands-on Exercise 1: Method Sets and Interface Satisfaction Rules**:

```go
// Understanding method sets and interface satisfaction in automation systems
package main

import (
    "fmt"
    "log"
)

// Notifier interface requires notification behavior
type Notifier interface {
    Notify(message string) error
}

// StatusReporter interface requires status reporting behavior
type StatusReporter interface {
    GetStatus() string
}

// Combined interface requiring both behaviors
type NotifierWithStatus interface {
    Notifier
    StatusReporter
}

// AutomationService represents a service in our automation system
type AutomationService struct {
    Name    string
    Active  bool
    Version string
}

// Value receiver method - part of VALUE method set
// This method can be called on both values and pointers
func (as AutomationService) GetStatus() string {
    status := "inactive"
    if as.Active {
        status = "active"
    }
    return fmt.Sprintf("Service %s v%s is %s", as.Name, as.Version, status)
}

// Pointer receiver method - part of POINTER method set ONLY
// This method can ONLY be called on pointers, not values
func (as *AutomationService) Notify(message string) error {
    log.Printf("NOTIFICATION [%s]: %s", as.Name, message)
    return nil
}

// Another pointer receiver method
func (as *AutomationService) Activate() {
    as.Active = true
    log.Printf("Service %s activated", as.Name)
}

func demonstrateMethodSetRules() {
    fmt.Println("=== Method Set Rules ===")

    fmt.Println("Method Set Rules from Go Specification:")
    fmt.Println("1. For VALUE of type T: only VALUE receiver methods")
    fmt.Println("2. For POINTER of type T: both VALUE and POINTER receiver methods")
    fmt.Println()

    service := AutomationService{
        Name:    "DataProcessor",
        Active:  false,
        Version: "1.0",
    }

    // VALUE method set demonstration
    fmt.Println("--- VALUE Method Set ---")
    fmt.Printf("Value can call GetStatus(): %s\n", service.GetStatus())

    // This would work because Go automatically takes address for pointer methods
    // when called directly on a value (syntactic sugar)
    service.Notify("Direct call works due to Go's syntactic sugar")

    // But when it comes to interface satisfaction, the rules are strict!

    // VALUE can satisfy StatusReporter (value receiver method)
    var reporter StatusReporter = service
    fmt.Printf("StatusReporter interface: %s\n", reporter.GetStatus())

    // VALUE CANNOT satisfy Notifier interface (pointer receiver method)
    // This would NOT compile:
    // var notifier Notifier = service // ERROR!

    fmt.Println("\n--- POINTER Method Set ---")
    // POINTER can satisfy both interfaces
    var notifierWithStatus NotifierWithStatus = &service
    fmt.Printf("Combined interface status: %s\n", notifierWithStatus.GetStatus())
    notifierWithStatus.Notify("Pointer satisfies both interfaces!")
}

func demonstrateWhyTheseRules() {
    fmt.Println("\n=== Why These Rules Exist ===")

    fmt.Println("Reason 1: Not every value has an address")

    // Example: constants and literals don't have addresses
    type Duration int

    // If Duration had a pointer receiver method, this wouldn't work:
    // Duration(42).SomePointerMethod() // No address for literal!

    fmt.Println("Literals like Duration(42) have no address")
    fmt.Println("So pointer receiver methods can't be in value method sets")

    fmt.Println("\nReason 2: Semantic integrity (the BIG reason)")
    fmt.Println("If you choose POINTER semantics, you should only SHARE")
    fmt.Println("If you choose VALUE semantics, you should make COPIES")
    fmt.Println("Never go from pointer semantics to value semantics!")

    service := AutomationService{Name: "TestService", Active: true, Version: "2.0"}

    // If this were allowed (it's not), it would violate semantic integrity:
    // var notifier Notifier = service // Would copy the value that pointer points to!

    fmt.Println("\nThe compiler prevents semantic violations:")
    fmt.Println("- Pointer receiver = pointer semantics = sharing only")
    fmt.Println("- Value receiver = value semantics = copying allowed")
}

func demonstrateInterfaceComplianceChecking() {
    fmt.Println("\n=== Interface Compliance Checking ===")

    service := AutomationService{Name: "ComplianceTest", Active: false, Version: "1.5"}

    // Compile-time verification examples

    // ✅… This compiles - value satisfies StatusReporter
    var _ StatusReporter = service
    fmt.Println("✅… Value satisfies StatusReporter (value receiver)")

    // ✅… This compiles - pointer satisfies StatusReporter
    var _ StatusReporter = &service
    fmt.Println("✅… Pointer satisfies StatusReporter (value receiver)")

    // ✅… This compiles - pointer satisfies Notifier
    var _ Notifier = &service
    fmt.Println("✅… Pointer satisfies Notifier (pointer receiver)")

    // ❌ This would NOT compile - value cannot satisfy Notifier
    // var _ Notifier = service
    fmt.Println("❌ Value CANNOT satisfy Notifier (pointer receiver)")

    // ✅… This compiles - pointer satisfies combined interface
    var _ NotifierWithStatus = &service
    fmt.Println("✅… Pointer satisfies NotifierWithStatus (both receivers)")

    // ❌ This would NOT compile - value missing pointer receiver methods
    // var _ NotifierWithStatus = service
    fmt.Println("❌ Value CANNOT satisfy NotifierWithStatus (missing pointer receiver)")
}

func demonstrateSemanticConsistency() {
    fmt.Println("\n=== Semantic Consistency in Practice ===")

    service := AutomationService{Name: "SemanticTest", Active: false, Version: "3.0"}

    // Correct: Consistent pointer semantics
    fmt.Println("--- Consistent Pointer Semantics ---")
    var notifier Notifier = &service // Pointer semantics
    notifier.Notify("Using pointer semantics consistently")

    // If we could do this (we can't), it would be inconsistent:
    // var badNotifier Notifier = service // Would be value semantics!

    fmt.Println("\n--- Why Consistency Matters ---")
    fmt.Println("1. Predictable behavior across the codebase")
    fmt.Println("2. Clear ownership and mutation rules")
    fmt.Println("3. Performance characteristics are obvious")
    fmt.Println("4. Easier to reason about data flow")

    // Demonstrate the difference
    fmt.Println("\n--- Value vs Pointer Semantics ---")

    // Value semantics - creates copy
    var statusReporter StatusReporter = service
    service.Active = true // Modify original
    fmt.Printf("Original after change: %s\n", service.GetStatus())
    fmt.Printf("Interface copy (unchanged): %s\n", statusReporter.GetStatus())

    // Pointer semantics - shares data
    var statusReporter2 StatusReporter = &service
    service.Version = "3.1" // Modify original
    fmt.Printf("Interface pointer (reflects change): %s\n", statusReporter2.GetStatus())
}

func main() {
    demonstrateMethodSetRules()
    demonstrateWhyTheseRules()
    demonstrateInterfaceComplianceChecking()
    demonstrateSemanticConsistency()
}
```

**Hands-on Exercise 2: Address-of-Value and Semantic Consistency**:

```go
// Address-of-value semantics and consistency in automation systems
package main

import (
    "fmt"
    "log"
)

// Processor interface with pointer receiver requirement
type Processor interface {
    Process(data string) error
}

// ConfigManager interface with value receiver
type ConfigManager interface {
    GetConfig() map[string]string
}

// AutomationTask represents a task in our system
type AutomationTask struct {
    ID       string
    Name     string
    Config   map[string]string
    Status   string
    Priority int
}

// Value receiver - can be called on values and pointers
func (at AutomationTask) GetConfig() map[string]string {
    // Return a copy to maintain value semantics
    config := make(map[string]string)
    for k, v := range at.Config {
        config[k] = v
    }
    return config
}

// Pointer receiver - can only be called on pointers
func (at *AutomationTask) Process(data string) error {
    log.Printf("Processing task %s with data: %s", at.ID, data)
    at.Status = "completed"
    return nil
}

// Pointer receiver - modifies state
func (at *AutomationTask) UpdatePriority(priority int) {
    at.Priority = priority
    log.Printf("Task %s priority updated to %d", at.ID, priority)
}

func demonstrateAddressOfValue() {
    fmt.Println("=== Address-of-Value Demonstration ===")

    task := AutomationTask{
        ID:       "TASK-001",
        Name:     "Data Processing",
        Config:   map[string]string{"timeout": "30s"},
        Status:   "pending",
        Priority: 5,
    }

    fmt.Printf("Task created: %+v\n", task)

    // Value can satisfy ConfigManager (value receiver)
    var configMgr ConfigManager = task
    config := configMgr.GetConfig()
    fmt.Printf("Config via interface: %v\n", config)

    // Value CANNOT satisfy Processor interface directly
    // This would NOT compile:
    // var processor Processor = task // ERROR!

    // But we can take the address to satisfy Processor
    var processor Processor = &task // ✅… This works!
    processor.Process("sample data")

    fmt.Printf("Task after processing: %+v\n", task)
}

func demonstrateSemanticViolations() {
    fmt.Println("\n=== Semantic Violations (What Go Prevents) ===")

    task := AutomationTask{
        ID:       "TASK-002",
        Name:     "Validation",
        Config:   map[string]string{"strict": "true"},
        Status:   "pending",
        Priority: 3,
    }

    fmt.Println("Why Go prevents value->interface for pointer receivers:")
    fmt.Println("1. Would violate semantic consistency")
    fmt.Println("2. Would create a copy when sharing was intended")
    fmt.Println("3. Would break the 'pointer semantics = sharing' rule")

    // If this were allowed (it's not):
    // var processor Processor = task // Would copy task into interface
    // processor.Process("data")      // Would modify the COPY, not original!

    fmt.Println("\nCorrect approach - maintain semantic consistency:")

    // Use pointer semantics consistently
    var processor Processor = &task
    processor.Process("validation data")

    fmt.Printf("Original task modified: %s\n", task.Status)

    // The interface holds a pointer, so changes affect the original
    if taskPtr, ok := processor.(*AutomationTask); ok {
        taskPtr.UpdatePriority(1)
        fmt.Printf("Priority updated via interface: %d\n", task.Priority)
    }
}

func demonstrateConstantAddressing() {
    fmt.Println("\n=== Constants and Addressing ===")

    // Custom type based on string
    type TaskID string

    // If TaskID had a pointer receiver method, this wouldn't work:
    // TaskID("CONST-001").SomePointerMethod() // No address!

    fmt.Println("Constants and literals have no address:")
    fmt.Println("- TaskID(\"CONST-001\") exists only at compile time")
    fmt.Println("- Cannot take address of compile-time constants")
    fmt.Println("- This is why pointer receivers aren't in value method sets")

    // This works because we create a variable (has address)
    taskID := TaskID("VAR-001")
    fmt.Printf("Variable taskID has address: %p\n", &taskID)

    // But this literal has no address:
    // fmt.Printf("Literal address: %p\n", &TaskID("LITERAL")) // Won't compile!

    fmt.Println("\nThis is the 'minor' reason for method set rules")
    fmt.Println("The major reason is semantic consistency")
}

func demonstrateSemanticChoices() {
    fmt.Println("\n=== Making Semantic Choices ===")

    fmt.Println("When designing types, choose semantics based on:")
    fmt.Println("1. Is this data meant to be shared or copied?")
    fmt.Println("2. Does the type represent a value or an entity?")
    fmt.Println("3. What are the performance characteristics?")
    fmt.Println("4. How will this type be used in the system?")

    // Example: Small, immutable data - value semantics
    type Coordinate struct {
        X, Y float64
    }

    // Value receiver - copying is cheap and safe
    func (c Coordinate) Distance() float64 {
        return c.X*c.X + c.Y*c.Y
    }

    coord := Coordinate{X: 3, Y: 4}
    fmt.Printf("Coordinate distance: %.2f\n", coord.Distance())

    // Example: Large, mutable data - pointer semantics
    type LargeDataSet struct {
        Data    [10000]int
        Metrics map[string]float64
        Status  string
    }

    // Pointer receiver - copying would be expensive
    func (lds *LargeDataSet) UpdateStatus(status string) {
        lds.Status = status
    }

    dataset := LargeDataSet{
        Metrics: make(map[string]float64),
        Status:  "initializing",
    }

    dataset.UpdateStatus("ready")
    fmt.Printf("Dataset status: %s\n", dataset.Status)
}

func demonstrateInterfaceSemantics() {
    fmt.Println("\n=== Interface Semantic Guidelines ===")

    task1 := AutomationTask{
        ID:     "TASK-A",
        Name:   "Config Task",
        Config: map[string]string{"env": "prod"},
        Status: "ready",
    }

    task2 := AutomationTask{
        ID:     "TASK-B",
        Name:   "Process Task",
        Config: map[string]string{"workers": "4"},
        Status: "pending",
    }

    fmt.Println("Guidelines for interface usage:")
    fmt.Println("1. If type uses value semantics, prefer value interface storage")
    fmt.Println("2. If type uses pointer semantics, use pointer interface storage")
    fmt.Println("3. Don't mix semantics unnecessarily")
    fmt.Println("4. Be consistent across your API")

    // Consistent value semantics for read-only operations
    var configMgr1 ConfigManager = task1 // Copy for read-only
    var configMgr2 ConfigManager = task2 // Copy for read-only

    fmt.Printf("Config 1: %v\n", configMgr1.GetConfig())
    fmt.Printf("Config 2: %v\n", configMgr2.GetConfig())

    // Consistent pointer semantics for stateful operations
    var processor1 Processor = &task1 // Share for mutation
    var processor2 Processor = &task2 // Share for mutation

    processor1.Process("production data")
    processor2.Process("test data")

    fmt.Printf("Task 1 status: %s\n", task1.Status)
    fmt.Printf("Task 2 status: %s\n", task2.Status)
}

func main() {
    demonstrateAddressOfValue()
    demonstrateSemanticViolations()
    demonstrateConstantAddressing()
    demonstrateSemanticChoices()
    demonstrateInterfaceSemantics()
}
```

**Hands-on Exercise 3: Interface Storage by Value and Allocation Costs**:

```go
// Interface storage mechanics and allocation costs in automation systems
package main

import (
    "fmt"
    "unsafe"
)

// Printer interface for demonstrating storage mechanics
type Printer interface {
    Print() string
}

// Small concrete type - efficient to copy
type SmallTask struct {
    ID   int
    Name string
}

func (st SmallTask) Print() string {
    return fmt.Sprintf("SmallTask[%d]: %s", st.ID, st.Name)
}

// Large concrete type - expensive to copy
type LargeTask struct {
    ID       int
    Name     string
    Data     [1024]byte
    Metadata map[string]string
    Config   [100]string
}

func (lt LargeTask) Print() string {
    return fmt.Sprintf("LargeTask[%d]: %s (size: %d bytes)",
        lt.ID, lt.Name, unsafe.Sizeof(lt))
}

// Pointer-based task
type PointerTask struct {
    ID   int
    Name string
}

func (pt *PointerTask) Print() string {
    return fmt.Sprintf("PointerTask[%d]: %s", pt.ID, pt.Name)
}

func demonstrateInterfaceStorage() {
    fmt.Println("=== Interface Storage Mechanics ===")

    fmt.Printf("Interface size: %d bytes (two words)\n", unsafe.Sizeof((*Printer)(nil)))
    fmt.Println("Interface structure: [type info pointer | data pointer/value]")

    // Small type storage
    small := SmallTask{ID: 1, Name: "Small"}
    fmt.Printf("SmallTask size: %d bytes\n", unsafe.Sizeof(small))

    var printer1 Printer = small // Interface stores a COPY
    fmt.Printf("Stored SmallTask in interface (copied %d bytes)\n", unsafe.Sizeof(small))

    // Large type storage
    large := LargeTask{
        ID:       2,
        Name:     "Large",
        Metadata: make(map[string]string),
    }
    fmt.Printf("LargeTask size: %d bytes\n", unsafe.Sizeof(large))

    var printer2 Printer = large // Interface stores a COPY of ALL data!
    fmt.Printf("Stored LargeTask in interface (copied %d bytes!)\n", unsafe.Sizeof(large))

    // Pointer storage
    pointer := &PointerTask{ID: 3, Name: "Pointer"}
    fmt.Printf("Pointer size: %d bytes\n", unsafe.Sizeof(pointer))

    var printer3 Printer = pointer // Interface stores the pointer (8 bytes)
    fmt.Printf("Stored pointer in interface (stored %d bytes)\n", unsafe.Sizeof(pointer))

    // Demonstrate the copies vs references
    fmt.Println("\n--- Demonstrating Copies vs References ---")

    // Modify originals
    small.Name = "Modified Small"
    large.Name = "Modified Large"
    pointer.Name = "Modified Pointer"

    fmt.Printf("Original small: %s\n", small.Print())
    fmt.Printf("Interface copy: %s\n", printer1.Print()) // Shows old name!

    fmt.Printf("Original large: %s\n", large.Print())
    fmt.Printf("Interface copy: %s\n", printer2.Print()) // Shows old name!

    fmt.Printf("Original pointer: %s\n", pointer.Print())
    fmt.Printf("Interface reference: %s\n", printer3.Print()) // Shows new name!
}

func demonstrateAllocationCosts() {
    fmt.Println("\n=== Allocation Costs of Interface Storage ===")

    fmt.Println("Every interface storage causes allocation:")
    fmt.Println("1. Interface value itself (16 bytes)")
    fmt.Println("2. Copy of concrete data (if stored by value)")
    fmt.Println("3. Type information structures")

    // Create a slice of interface values
    var printers []Printer

    // Each append causes allocation
    small1 := SmallTask{ID: 1, Name: "Task1"}
    small2 := SmallTask{ID: 2, Name: "Task2"}

    fmt.Println("\n--- Value Semantics (Copies) ---")
    printers = append(printers, small1) // Allocation: interface + copy of small1
    printers = append(printers, small2) // Allocation: interface + copy of small2

    fmt.Printf("Stored 2 SmallTasks by value\n")
    fmt.Printf("Memory cost: 2 Ã— (%d interface + %d data) = %d bytes\n",
        unsafe.Sizeof((*Printer)(nil)), unsafe.Sizeof(small1),
        2*(int(unsafe.Sizeof((*Printer)(nil)))+int(unsafe.Sizeof(small1))))

    // Modify originals - interface copies unchanged
    small1.Name = "Modified1"
    small2.Name = "Modified2"

    fmt.Println("After modifying originals:")
    for i, p := range printers {
        fmt.Printf("  Interface %d: %s\n", i, p.Print())
    }

    fmt.Println("\n--- Pointer Semantics (References) ---")
    var pointerPrinters []Printer

    ptr1 := &PointerTask{ID: 1, Name: "PtrTask1"}
    ptr2 := &PointerTask{ID: 2, Name: "PtrTask2"}

    pointerPrinters = append(pointerPrinters, ptr1) // Allocation: interface only
    pointerPrinters = append(pointerPrinters, ptr2) // Allocation: interface only

    fmt.Printf("Stored 2 PointerTasks by reference\n")
    fmt.Printf("Memory cost: 2 Ã— %d interface = %d bytes\n",
        unsafe.Sizeof((*Printer)(nil)), 2*int(unsafe.Sizeof((*Printer)(nil))))

    // Modify originals - interface references see changes
    ptr1.Name = "ModifiedPtr1"
    ptr2.Name = "ModifiedPtr2"

    fmt.Println("After modifying originals:")
    for i, p := range pointerPrinters {
        fmt.Printf("  Interface %d: %s\n", i, p.Print())
    }
}

func demonstrateForRangeSemantics() {
    fmt.Println("\n=== For Range Semantics with Interfaces ===")

    // Create slice of interface values
    printers := []Printer{
        SmallTask{ID: 1, Name: "Task1"},
        &PointerTask{ID: 2, Name: "Task2"},
    }

    fmt.Println("Interface slice contains:")
    fmt.Println("- Index 0: SmallTask (stored by value)")
    fmt.Println("- Index 1: *PointerTask (stored by pointer)")

    fmt.Println("\n--- Value Semantics For Range ---")
    fmt.Println("for _, printer := range printers")

    // Value semantics for range - creates copies of interface values
    for i, printer := range printers {
        fmt.Printf("Iteration %d:\n", i)
        fmt.Printf("  Interface variable 'printer' is a COPY\n")
        fmt.Printf("  Result: %s\n", printer.Print())

        if i == 0 {
            fmt.Printf("  This is a copy of interface containing copy of SmallTask\n")
        } else {
            fmt.Printf("  This is a copy of interface containing pointer to PointerTask\n")
        }
    }

    fmt.Println("\nKey insight: 'printer' variable is reused but gets new copies each iteration")
    fmt.Println("- For value-stored interfaces: copy of copy")
    fmt.Println("- For pointer-stored interfaces: copy of interface (same pointer)")
}

func demonstratePolymorphismWithStorage() {
    fmt.Println("\n=== Polymorphism and Storage Interaction ===")

    // Same interface, different storage, different behavior
    printers := []Printer{
        SmallTask{ID: 1, Name: "ValueTask"},
        &PointerTask{ID: 2, Name: "PointerTask"},
    }

    fmt.Println("Polymorphism in action:")
    fmt.Println("- Same Print() call")
    fmt.Println("- Different concrete types")
    fmt.Println("- Different storage mechanisms")
    fmt.Println("- Different behaviors")

    for i, printer := range printers {
        fmt.Printf("\nIteration %d:\n", i)
        fmt.Printf("  Concrete type: %T\n", printer)
        fmt.Printf("  Print result: %s\n", printer.Print())

        // The behavior changes based on concrete data
        // But the storage mechanism also affects performance
    }

    fmt.Println("\nPolymorphism = same code, different behavior")
    fmt.Println("Storage semantics = different performance characteristics")
    fmt.Println("Both are driven by the concrete data and how it's stored")
}

func demonstratePerformanceImplications() {
    fmt.Println("\n=== Performance Implications ===")

    fmt.Println("Interface storage performance considerations:")
    fmt.Println()

    fmt.Println("1. Value Storage:")
    fmt.Println("   + No indirection for data access")
    fmt.Println("   + Data locality (interface and data together)")
    fmt.Println("   - Copying cost (especially for large types)")
    fmt.Println("   - Memory usage (full copy stored)")

    fmt.Println("\n2. Pointer Storage:")
    fmt.Println("   + No copying cost")
    fmt.Println("   + Memory efficient (only pointer stored)")
    fmt.Println("   - Extra indirection for data access")
    fmt.Println("   - Potential cache misses")

    fmt.Println("\n3. Method Calls:")
    fmt.Println("   - All interface method calls have double indirection")
    fmt.Println("   - Type info lookup + method dispatch")
    fmt.Println("   - Cannot be inlined by compiler")

    fmt.Println("\nChoose based on:")
    fmt.Println("- Size of concrete type")
    fmt.Println("- Frequency of copying vs method calls")
    fmt.Println("- Memory vs CPU trade-offs")
    fmt.Println("- Semantic consistency requirements")
}

func main() {
    demonstrateInterfaceStorage()
    demonstrateAllocationCosts()
    demonstrateForRangeSemantics()
    demonstratePolymorphismWithStorage()
    demonstratePerformanceImplications()
}
```

**Prerequisites**: Module 18
