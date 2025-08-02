# Module 18: Interfaces Part 1: Polymorphism and Design

**Duration**: 1 hour (45 min theory + 15 min hands-on)

**Learning Objectives**:

- Understand polymorphism and its role in decoupling
- Master interface declaration and implementation concepts
- Learn interface design principles (behavior over nouns)
- Understand interface values and their internal structure
- Apply polymorphic design to automation systems
- Design efficient APIs that minimize allocations

**Videos Covered**:

- 4.5 Interfaces Part 1 (Polymorphism) (0:20:11)

**Key Concepts**:

- Polymorphism: code behavior changes based on concrete data
- Interface types are not real - they define method sets (contracts)
- Interfaces should define behavior (verbs) not things (nouns)
- Interface values store concrete data using two-pointer structure
- API design: minimize allocations and prevent misuse
- Concrete data drives polymorphism through behavior
- Static analysis verifies interface satisfaction at compile time

**Hands-on Exercise 1: Interface Fundamentals and Polymorphism**:

```go
// Understanding interface basics and polymorphism in automation systems
package main

import (
    "fmt"
    "log"
)

// Reader interface defines behavior for reading data
// Notice: focuses on behavior (verb "read") not nouns
type Reader interface {
    Read(data []byte) (int, error)
}

// FileReader represents reading from a file system
type FileReader struct {
    Filename string
}

// Value receiver - implements Reader interface using value semantics
func (fr FileReader) Read(data []byte) (int, error) {
    // Simulate reading from file
    content := fmt.Sprintf("Content from file: %s", fr.Filename)
    n := copy(data, []byte(content))
    log.Printf("FileReader read %d bytes from %s", n, fr.Filename)
    return n, nil
}

// NetworkReader represents reading from network
type NetworkReader struct {
    URL string
}

// Value receiver - implements Reader interface using value semantics
func (nr NetworkReader) Read(data []byte) (int, error) {
    // Simulate reading from network
    content := fmt.Sprintf("Data from URL: %s", nr.URL)
    n := copy(data, []byte(content))
    log.Printf("NetworkReader read %d bytes from %s", n, nr.URL)
    return n, nil
}

// Polymorphic function - works with ANY concrete type that implements Reader
// This function has NO knowledge of concrete types!
func RetrieveData(r Reader, buffer []byte) (int, error) {
    fmt.Printf("RetrieveData called with type: %T\n", r)

    // The behavior of Read() changes based on the concrete data passed in
    // This is polymorphism: same code, different behavior
    return r.Read(buffer)
}

func demonstratePolymorphism() {
    fmt.Println("=== Polymorphism Demonstration ===")

    // Create concrete data
    fileReader := FileReader{Filename: "automation.log"}
    networkReader := NetworkReader{URL: "https://api.automation.com/data"}

    // Create buffer for reading
    buffer := make([]byte, 100)

    // Same function, different behavior based on concrete data
    fmt.Println("\n--- Using FileReader ---")
    n1, err1 := RetrieveData(fileReader, buffer)
    if err1 == nil {
        fmt.Printf("Read: %s\n", string(buffer[:n1]))
    }

    // Clear buffer
    for i := range buffer {
        buffer[i] = 0
    }

    fmt.Println("\n--- Using NetworkReader ---")
    n2, err2 := RetrieveData(networkReader, buffer)
    if err2 == nil {
        fmt.Printf("Read: %s\n", string(buffer[:n2]))
    }

    fmt.Println("\nKey insight: RetrieveData function never changed,")
    fmt.Println("but its behavior changed based on the concrete data!")
}

func demonstrateInterfaceReality() {
    fmt.Println("\n=== Interface Reality Check ===")

    // Interface types are NOT real - they're just contracts
    var r Reader
    fmt.Printf("Zero-valued interface: %v\n", r)
    fmt.Printf("Is nil? %t\n", r == nil)

    // We can only store CONCRETE data in interfaces
    fileReader := FileReader{Filename: "test.txt"}

    // Now we store concrete data in the interface
    r = fileReader // This makes the interface "concrete"
    fmt.Printf("Interface with concrete data: %v\n", r)
    fmt.Printf("Is nil? %t\n", r == nil)
    fmt.Printf("Concrete type stored: %T\n", r)

    // The interface value is now meaningful because it contains concrete data
    buffer := make([]byte, 50)
    n, _ := r.Read(buffer)
    fmt.Printf("Read through interface: %s\n", string(buffer[:n]))
}

func demonstrateConventionOverConfiguration() {
    fmt.Println("\n=== Convention Over Configuration ===")

    // Go uses convention, not explicit configuration
    // We don't write: "class FileReader implements Reader"
    // We just implement the method with the right signature

    fmt.Println("FileReader implements Reader because:")
    fmt.Println("1. It has a method named 'Read'")
    fmt.Println("2. The method signature matches exactly: Read([]byte) (int, error)")
    fmt.Println("3. Go's compiler verifies this at compile time")

    // This is verified by static analysis - no runtime checking needed
    var readers []Reader = []Reader{
        FileReader{Filename: "file1.txt"},
        NetworkReader{URL: "http://example.com"},
    }

    fmt.Printf("\nSuccessfully created slice of %d readers\n", len(readers))
    fmt.Println("Compiler verified interface compliance at compile time!")
}

func main() {
    demonstratePolymorphism()
    demonstrateInterfaceReality()
    demonstrateConventionOverConfiguration()
}
```

**Hands-on Exercise 2: Interface Design Principles and API Efficiency**:

```go
// Interface design principles and allocation-efficient APIs
package main

import (
    "fmt"
    "log"
)

// GOOD: Interface focuses on behavior (verb)
type DataProcessor interface {
    Process(input []byte, output []byte) (int, error) // Allocation-efficient API
}

// BAD: Interface focuses on nouns (things)
// type AutomationThing interface {
//     GetAutomation() Automation
//     GetThing() Thing
// }

// GOOD: Behavior-focused interface
type Logger interface {
    Log(level string, message string) error
}

// GOOD: Small, focused interface (single responsibility)
type Validator interface {
    Validate(data []byte) bool
}

// BAD: Large interface with multiple responsibilities
// type MegaInterface interface {
//     Process([]byte) []byte
//     Log(string) error
//     Validate([]byte) bool
//     Send([]byte) error
//     Receive() []byte
//     Configure(map[string]string) error
// }

// Efficient processor implementation
type TextProcessor struct {
    Name string
}

// Allocation-efficient API: caller provides output buffer
func (tp TextProcessor) Process(input []byte, output []byte) (int, error) {
    // Transform input to uppercase
    processed := 0
    for i, b := range input {
        if i >= len(output) {
            break
        }
        if b >= 'a' && b <= 'z' {
            output[i] = b - 32 // Convert to uppercase
        } else {
            output[i] = b
        }
        processed++
    }

    log.Printf("TextProcessor processed %d bytes", processed)
    return processed, nil
}

// Inefficient processor (for comparison)
type InefficientProcessor struct {
    Name string
}

// BAD: Allocation-heavy API - creates new slice every time
func (ip InefficientProcessor) ProcessInefficient(input []byte) []byte {
    // This allocates a new slice every call!
    output := make([]byte, len(input)) // Allocation!

    for i, b := range input {
        if b >= 'a' && b <= 'z' {
            output[i] = b - 32
        } else {
            output[i] = b
        }
    }

    return output // Another potential allocation when returning up stack!
}

// Console logger with efficient design
type ConsoleLogger struct {
    Prefix string
}

func (cl ConsoleLogger) Log(level string, message string) error {
    // No allocations - just formatting and printing
    fmt.Printf("[%s] %s: %s\n", cl.Prefix, level, message)
    return nil
}

// File logger simulation
type FileLogger struct {
    Filename string
}

func (fl FileLogger) Log(level string, message string) error {
    // Simulate file writing without actual I/O
    log.Printf("Writing to %s: [%s] %s", fl.Filename, level, message)
    return nil
}

// Simple validator
type LengthValidator struct {
    MinLength int
    MaxLength int
}

func (lv LengthValidator) Validate(data []byte) bool {
    length := len(data)
    return length >= lv.MinLength && length <= lv.MaxLength
}

// Polymorphic function with efficient API design
func ProcessData(processor DataProcessor, logger Logger, validator Validator, input []byte) {
    logger.Log("INFO", "Starting data processing")

    // Validate input first
    if !validator.Validate(input) {
        logger.Log("ERROR", "Input validation failed")
        return
    }

    // Use pre-allocated buffer to avoid allocations
    output := make([]byte, len(input)) // Single allocation

    n, err := processor.Process(input, output)
    if err != nil {
        logger.Log("ERROR", fmt.Sprintf("Processing failed: %v", err))
        return
    }

    logger.Log("INFO", fmt.Sprintf("Processed %d bytes: %s", n, string(output[:n])))
}

func demonstrateInterfaceDesignPrinciples() {
    fmt.Println("=== Interface Design Principles ===")

    fmt.Println("GOOD Interface Design:")
    fmt.Println("1. Focus on behavior (verbs): Reader, Writer, Processor")
    fmt.Println("2. Keep interfaces small and focused")
    fmt.Println("3. Design for minimal allocations")
    fmt.Println("4. Use composition for complex behavior")

    fmt.Println("\nBAD Interface Design:")
    fmt.Println("1. Focus on nouns: Animal, User, Thing")
    fmt.Println("2. Large interfaces with many methods")
    fmt.Println("3. APIs that force allocations")
    fmt.Println("4. Monolithic interfaces")
}

func demonstrateAllocationEfficiency() {
    fmt.Println("\n=== API Allocation Efficiency ===")

    input := []byte("hello automation world")

    // Efficient approach: caller controls allocation
    processor := TextProcessor{Name: "EfficientProcessor"}
    logger := ConsoleLogger{Prefix: "EFFICIENT"}
    validator := LengthValidator{MinLength: 5, MaxLength: 100}

    fmt.Println("--- Efficient API (minimal allocations) ---")
    ProcessData(processor, logger, validator, input)

    // Inefficient approach (for comparison)
    fmt.Println("\n--- Inefficient API (many allocations) ---")
    inefficient := InefficientProcessor{Name: "WastefulProcessor"}

    // This would allocate every time it's called
    result := inefficient.ProcessInefficient(input)
    fmt.Printf("Inefficient result: %s\n", string(result))
    fmt.Println("Note: This allocated a new slice and potentially more on return!")
}

func demonstrateInterfaceComposition() {
    fmt.Println("\n=== Interface Composition ===")

    // Small, focused interfaces can be composed
    type ProcessorWithLogging interface {
        DataProcessor
        Logger
    }

    // This is better than one large interface because:
    // 1. Each interface has a single responsibility
    // 2. Types can implement just what they need
    // 3. Easier to test and mock
    // 4. More flexible composition

    fmt.Println("Interface composition allows:")
    fmt.Println("1. Single responsibility per interface")
    fmt.Println("2. Flexible implementation choices")
    fmt.Println("3. Easier testing and mocking")
    fmt.Println("4. Better code organization")
}

func demonstrateWhenToUseInterfaces() {
    fmt.Println("\n=== When to Use Interfaces ===")

    fmt.Println("Use interfaces when you need:")
    fmt.Println("1. Polymorphism - same code, different behavior")
    fmt.Println("2. Decoupling - reduce dependencies between components")
    fmt.Println("3. Testing - ability to mock dependencies")
    fmt.Println("4. Plugin architecture - swappable implementations")

    fmt.Println("\nDON'T use interfaces when:")
    fmt.Println("1. You only have one implementation")
    fmt.Println("2. The cost of indirection isn't justified")
    fmt.Println("3. You're just following patterns without purpose")
    fmt.Println("4. The interface would be larger than the implementation")
}

func main() {
    demonstrateInterfaceDesignPrinciples()
    demonstrateAllocationEfficiency()
    demonstrateInterfaceComposition()
    demonstrateWhenToUseInterfaces()
}
```

**Hands-on Exercise 3: Interface Values and Storage Mechanics**:

```go
// Understanding interface values and storage mechanics
package main

import (
    "fmt"
    "unsafe"
)

// Simple interface for demonstration
type Processor interface {
    Process(data string) string
}

// Small concrete type
type SmallProcessor struct {
    ID   int
    Name string
}

func (sp SmallProcessor) Process(data string) string {
    return fmt.Sprintf("[%s-%d] %s", sp.Name, sp.ID, data)
}

// Large concrete type
type LargeProcessor struct {
    ID       int
    Name     string
    Config   map[string]string
    Buffer   [1024]byte
    Metadata [100]string
}

func (lp LargeProcessor) Process(data string) string {
    return fmt.Sprintf("[LARGE-%s-%d] %s", lp.Name, lp.ID, data)
}

// Pointer-based processor
type PointerProcessor struct {
    ID   int
    Name string
}

func (pp *PointerProcessor) Process(data string) string {
    return fmt.Sprintf("[PTR-%s-%d] %s", pp.Name, pp.ID, data)
}

func demonstrateInterfaceStorage() {
    fmt.Println("=== Interface Storage Mechanics ===")

    // Interface values are two-word structures:
    // Word 1: Pointer to type information (itable)
    // Word 2: Pointer to data or data itself (if fits in a word)

    var p Processor
    fmt.Printf("Empty interface size: %d bytes\n", unsafe.Sizeof(p))
    fmt.Printf("Empty interface value: %v\n", p)
    fmt.Printf("Is nil? %t\n", p == nil)

    // Store small concrete type
    small := SmallProcessor{ID: 1, Name: "Small"}
    fmt.Printf("\nSmall concrete type size: %d bytes\n", unsafe.Sizeof(small))

    p = small // Interface stores a COPY of the concrete data
    fmt.Printf("Interface with small type: %v\n", p)
    fmt.Printf("Is nil? %t\n", p == nil)

    // The interface contains a copy - modifying original doesn't affect interface
    small.Name = "Modified"
    fmt.Printf("Original modified: %s\n", small.Name)
    fmt.Printf("Interface copy unchanged: %v\n", p)

    // Store large concrete type
    large := LargeProcessor{
        ID:   2,
        Name: "Large",
        Config: map[string]string{"key": "value"},
    }
    fmt.Printf("\nLarge concrete type size: %d bytes\n", unsafe.Sizeof(large))

    p = large // Interface stores a COPY of ALL the data!
    fmt.Printf("Interface with large type stored (copied %d bytes)\n", unsafe.Sizeof(large))

    // Store pointer to concrete type
    ptr := &PointerProcessor{ID: 3, Name: "Pointer"}
    fmt.Printf("\nPointer size: %d bytes\n", unsafe.Sizeof(ptr))

    p = ptr // Interface stores the pointer value (8 bytes on 64-bit)
    fmt.Printf("Interface with pointer type stored\n")

    // Modifying through pointer affects what interface points to
    ptr.Name = "ModifiedPointer"
    result := p.Process("test")
    fmt.Printf("Result after pointer modification: %s\n", result)
}

func demonstrateInterfaceConversions() {
    fmt.Println("\n=== Interface Conversions and Type Assertions ===")

    var p Processor

    // Store concrete type
    small := SmallProcessor{ID: 1, Name: "Test"}
    p = small

    // Type assertion - extract concrete type from interface
    if sp, ok := p.(SmallProcessor); ok {
        fmt.Printf("Successfully extracted SmallProcessor: %v\n", sp)
    }

    // Failed type assertion
    if _, ok := p.(LargeProcessor); !ok {
        fmt.Println("Cannot extract LargeProcessor from interface containing SmallProcessor")
    }

    // Type switch for multiple possibilities
    switch concrete := p.(type) {
    case SmallProcessor:
        fmt.Printf("Interface contains SmallProcessor: %s\n", concrete.Name)
    case LargeProcessor:
        fmt.Printf("Interface contains LargeProcessor: %s\n", concrete.Name)
    case *PointerProcessor:
        fmt.Printf("Interface contains PointerProcessor: %s\n", concrete.Name)
    default:
        fmt.Printf("Interface contains unknown type: %T\n", concrete)
    }
}

func demonstrateInterfaceComparison() {
    fmt.Println("\n=== Interface Comparison ===")

    var p1, p2 Processor

    // Both nil - equal
    fmt.Printf("Both nil: %t\n", p1 == p2)

    // Same concrete type and value
    small1 := SmallProcessor{ID: 1, Name: "Test"}
    small2 := SmallProcessor{ID: 1, Name: "Test"}

    p1 = small1
    p2 = small2
    fmt.Printf("Same concrete type and value: %t\n", p1 == p2)

    // Same concrete type, different value
    small3 := SmallProcessor{ID: 2, Name: "Different"}
    p2 = small3
    fmt.Printf("Same type, different value: %t\n", p1 == p2)

    // Different concrete types
    large := LargeProcessor{ID: 1, Name: "Test"}
    p2 = large
    fmt.Printf("Different concrete types: %t\n", p1 == p2)

    // Pointer comparisons
    ptr1 := &PointerProcessor{ID: 1, Name: "Test"}
    ptr2 := &PointerProcessor{ID: 1, Name: "Test"}

    p1 = ptr1
    p2 = ptr2
    fmt.Printf("Different pointer instances (same values): %t\n", p1 == p2)

    p2 = ptr1 // Same pointer
    fmt.Printf("Same pointer instance: %t\n", p1 == p2)
}

func demonstratePerformanceImplications() {
    fmt.Println("\n=== Performance Implications ===")

    // Direct concrete type call - no indirection
    small := SmallProcessor{ID: 1, Name: "Direct"}
    result1 := small.Process("test")
    fmt.Printf("Direct call result: %s\n", result1)

    // Interface call - double indirection
    var p Processor = small
    result2 := p.Process("test")
    fmt.Printf("Interface call result: %s\n", result2)

    fmt.Println("\nPerformance considerations:")
    fmt.Println("1. Interface calls have double indirection cost")
    fmt.Println("2. Large concrete types are copied entirely into interface")
    fmt.Println("3. Type assertions have runtime cost")
    fmt.Println("4. Interface comparisons can be expensive for large types")
    fmt.Println("5. Use pointers for large types to avoid copying")
}

func demonstrateInterfaceInternals() {
    fmt.Println("\n=== Interface Internals (Conceptual) ===")

    fmt.Println("Interface value structure (conceptual):")
    fmt.Println("┌─────────────────┬─────────────────┐")
    fmt.Println("│   Type Info     │      Data       │")
    fmt.Println("│   (itable)      │   (or pointer)  │")
    fmt.Println("│   8 bytes       │    8 bytes      │")
    fmt.Println("└─────────────────┴─────────────────┘˜")

    fmt.Println("\nType Info contains:")
    fmt.Println("- Pointer to interface method table")
    fmt.Println("- Runtime type information")
    fmt.Println("- Method dispatch information")

    fmt.Println("\nData contains:")
    fmt.Println("- Copy of concrete data (if small enough)")
    fmt.Println("- Pointer to concrete data (if too large)")
    fmt.Println("- Pointer value (for pointer receivers)")
}

func main() {
    demonstrateInterfaceStorage()
    demonstrateInterfaceConversions()
    demonstrateInterfaceComparison()
    demonstratePerformanceImplications()
    demonstrateInterfaceInternals()
}
```

**Prerequisites**: Module 17
