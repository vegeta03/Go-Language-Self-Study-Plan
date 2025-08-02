# Module 11: Slices Part 1: Declaration, Length, and Reference Types

**Duration**: 1 hour (45 min theory + 15 min hands-on)

**Learning Objectives**:

- Understand slice internal structure and mechanics
- Master slice declaration and initialization patterns
- Learn the difference between length and capacity
- Apply slices effectively in automation data processing

**Videos Covered**:

- 3.5 Slices Part 1 (Declare and Length) (0:08:46)

**Key Concepts**:

- Slice header structure (pointer, length, capacity)
- nil slices vs empty slices
- Slice literals and make() function
- Length vs capacity concepts
- Reference semantics with slices

**Hands-on Exercise 1: Dynamic Log Processing with Slices**:

```go
// Dynamic log processing with slices
package main

import (
    "fmt"
    "strings"
    "time"
)

type LogProcessor struct {
    entries []string
    errors  []string
}

func NewLogProcessor() *LogProcessor {
    return &LogProcessor{
        entries: make([]string, 0, 100), // Initial capacity of 100
        errors:  make([]string, 0, 10),  // Smaller capacity for errors
    }
}

func (lp *LogProcessor) AddEntry(entry string) {
    lp.entries = append(lp.entries, entry)

    // Check for error entries
    if strings.Contains(strings.ToLower(entry), "error") {
        lp.errors = append(lp.errors, entry)
    }
}

func (lp *LogProcessor) GetStats() (int, int, int, int) {
    return len(lp.entries), cap(lp.entries), len(lp.errors), cap(lp.errors)
}

func (lp *LogProcessor) GetRecentEntries(count int) []string {
    if count > len(lp.entries) {
        count = len(lp.entries)
    }

    start := len(lp.entries) - count
    return lp.entries[start:] // Slice operation
}

func main() {
    processor := NewLogProcessor()

    // Simulate log entries
    logEntries := []string{
        "INFO: Service started",
        "DEBUG: Processing request",
        "ERROR: Database connection failed",
        "INFO: Retrying connection",
        "ERROR: Authentication failed",
        "INFO: Service running normally",
    }

    fmt.Println("Processing log entries...")
    for i, entry := range logEntries {
        processor.AddEntry(fmt.Sprintf("[%s] %s",
            time.Now().Format("15:04:05"), entry))

        entryLen, entryCap, errorLen, errorCap := processor.GetStats()
        fmt.Printf("Entry %d: entries=%d/%d, errors=%d/%d\n",
            i+1, entryLen, entryCap, errorLen, errorCap)
    }

    // Get recent entries
    recent := processor.GetRecentEntries(3)
    fmt.Printf("\nRecent entries:\n")
    for _, entry := range recent {
        fmt.Printf("  %s\n", entry)
    }

    // Show all errors
    fmt.Printf("\nError entries:\n")
    for _, error := range processor.errors {
        fmt.Printf("  %s\n", error)
    }
}
```

**Hands-on Exercise 2: Slice Internal Structure and nil vs Empty**:

```go
// Explore slice internals and different initialization patterns
package main

import (
    "fmt"
    "unsafe"
)

// SliceHeader represents the internal structure of a slice
type SliceHeader struct {
    Data uintptr
    Len  int
    Cap  int
}

func main() {
    demonstrateSliceInternals()
    demonstrateNilVsEmpty()
    demonstrateInitializationPatterns()
}

func demonstrateSliceInternals() {
    fmt.Println("=== Slice Internal Structure ===")

    // Create different slices
    var nilSlice []int
    emptySlice := []int{}
    madeSlice := make([]int, 5, 10)
    literalSlice := []int{1, 2, 3}

    // Show internal structure
    printSliceInfo("nil slice", nilSlice)
    printSliceInfo("empty slice", emptySlice)
    printSliceInfo("made slice", madeSlice)
    printSliceInfo("literal slice", literalSlice)

    fmt.Printf("Slice header size: %d bytes\n", unsafe.Sizeof(nilSlice))
}

func printSliceInfo(name string, slice []int) {
    header := (*SliceHeader)(unsafe.Pointer(&slice))
    fmt.Printf("%s: len=%d, cap=%d, data=%x\n",
        name, header.Len, header.Cap, header.Data)
}

func demonstrateNilVsEmpty() {
    fmt.Println("\n=== nil vs Empty Slices ===")

    var nilSlice []string
    emptySlice := []string{}
    madeEmpty := make([]string, 0)

    fmt.Printf("nil slice == nil: %t\n", nilSlice == nil)
    fmt.Printf("empty slice == nil: %t\n", emptySlice == nil)
    fmt.Printf("made empty == nil: %t\n", madeEmpty == nil)

    fmt.Printf("len(nil slice): %d\n", len(nilSlice))
    fmt.Printf("len(empty slice): %d\n", len(emptySlice))
    fmt.Printf("len(made empty): %d\n", len(madeEmpty))

    // All can be appended to
    nilSlice = append(nilSlice, "first")
    emptySlice = append(emptySlice, "first")
    madeEmpty = append(madeEmpty, "first")

    fmt.Printf("After append - nil slice: %v\n", nilSlice)
    fmt.Printf("After append - empty slice: %v\n", emptySlice)
    fmt.Printf("After append - made empty: %v\n", madeEmpty)

    // JSON marshaling difference
    fmt.Println("\nJSON behavior:")
    fmt.Printf("nil slice would marshal to: null\n")
    fmt.Printf("empty slice would marshal to: []\n")
}

func demonstrateInitializationPatterns() {
    fmt.Println("\n=== Slice Initialization Patterns ===")

    // Pattern 1: nil slice (zero value)
    var tasks []string
    fmt.Printf("nil slice: len=%d, cap=%d\n", len(tasks), cap(tasks))

    // Pattern 2: empty slice literal
    configs := []string{}
    fmt.Printf("empty literal: len=%d, cap=%d\n", len(configs), cap(configs))

    // Pattern 3: make with length
    buffer := make([]byte, 10)
    fmt.Printf("make with length: len=%d, cap=%d\n", len(buffer), cap(buffer))

    // Pattern 4: make with length and capacity
    queue := make([]int, 0, 20)
    fmt.Printf("make with capacity: len=%d, cap=%d\n", len(queue), cap(queue))

    // Pattern 5: slice literal with values
    priorities := []int{1, 2, 3, 4, 5}
    fmt.Printf("literal with values: len=%d, cap=%d\n", len(priorities), cap(priorities))

    // Pattern 6: slice from array
    array := [5]string{"a", "b", "c", "d", "e"}
    slice := array[1:4]
    fmt.Printf("slice from array: len=%d, cap=%d, values=%v\n",
        len(slice), cap(slice), slice)

    // Show capacity calculation from array slice
    fmt.Printf("array slice [1:4] capacity = %d (from index 1 to end)\n",
        len(array)-1)

    // Different slice expressions
    slice1 := array[:]     // full slice
    slice2 := array[1:]    // from index 1 to end
    slice3 := array[:3]    // from start to index 3
    slice4 := array[1:3]   // from index 1 to 3

    fmt.Printf("array[:] -> len=%d, cap=%d\n", len(slice1), cap(slice1))
    fmt.Printf("array[1:] -> len=%d, cap=%d\n", len(slice2), cap(slice2))
    fmt.Printf("array[:3] -> len=%d, cap=%d\n", len(slice3), cap(slice3))
    fmt.Printf("array[1:3] -> len=%d, cap=%d\n", len(slice4), cap(slice4))
}
```

**Hands-on Exercise 3: Reference Semantics and Slice Sharing**:

```go
// Demonstrate slice reference semantics and data sharing
package main

import "fmt"

type DataProcessor struct {
    rawData     []int
    processedData []int
}

func NewDataProcessor(data []int) *DataProcessor {
    return &DataProcessor{
        rawData: data, // Shares the same backing array
    }
}

func (dp *DataProcessor) ProcessData() {
    // Create a slice that shares the backing array
    dp.processedData = make([]int, len(dp.rawData))

    // Process data (double each value)
    for i, value := range dp.rawData {
        dp.processedData[i] = value * 2
    }
}

func (dp *DataProcessor) ModifyRawData(index, value int) {
    if index >= 0 && index < len(dp.rawData) {
        dp.rawData[index] = value
    }
}

func (dp *DataProcessor) GetSlice(start, end int) []int {
    if start < 0 || end > len(dp.rawData) || start > end {
        return nil
    }
    // Returns a slice that shares the backing array
    return dp.rawData[start:end]
}

func main() {
    demonstrateReferenceSemantics()
    demonstrateSliceSharing()
    demonstrateSliceModification()
}

func demonstrateReferenceSemantics() {
    fmt.Println("=== Slice Reference Semantics ===")

    // Original data
    originalData := []int{1, 2, 3, 4, 5}
    fmt.Printf("Original data: %v\n", originalData)

    // Create processor with reference to data
    processor := NewDataProcessor(originalData)

    // Modify through processor
    processor.ModifyRawData(0, 99)

    // Original data is affected because slices share backing array
    fmt.Printf("After modification through processor: %v\n", originalData)
    fmt.Printf("Processor raw data: %v\n", processor.rawData)

    // Process the data
    processor.ProcessData()
    fmt.Printf("Processed data: %v\n", processor.processedData)
}

func demonstrateSliceSharing() {
    fmt.Println("\n=== Slice Sharing and Backing Arrays ===")

    // Create original slice
    original := []int{10, 20, 30, 40, 50, 60}
    fmt.Printf("Original: %v (len=%d, cap=%d)\n",
        original, len(original), cap(original))

    // Create slices that share the backing array
    slice1 := original[1:4]  // [20, 30, 40]
    slice2 := original[2:5]  // [30, 40, 50]
    slice3 := original[:3]   // [10, 20, 30]

    fmt.Printf("slice1 [1:4]: %v (len=%d, cap=%d)\n",
        slice1, len(slice1), cap(slice1))
    fmt.Printf("slice2 [2:5]: %v (len=%d, cap=%d)\n",
        slice2, len(slice2), cap(slice2))
    fmt.Printf("slice3 [:3]: %v (len=%d, cap=%d)\n",
        slice3, len(slice3), cap(slice3))

    // Modify through slice1
    slice1[1] = 999  // This modifies original[2]

    fmt.Printf("\nAfter slice1[1] = 999:\n")
    fmt.Printf("Original: %v\n", original)
    fmt.Printf("slice1: %v\n", slice1)
    fmt.Printf("slice2: %v\n", slice2)  // slice2[0] is also affected
    fmt.Printf("slice3: %v\n", slice3)  // slice3[2] is also affected
}

func demonstrateSliceModification() {
    fmt.Println("\n=== Safe vs Unsafe Slice Operations ===")

    // Create a slice for demonstration
    data := []string{"task1", "task2", "task3", "task4", "task5"}
    fmt.Printf("Original data: %v\n", data)

    // Safe operation: create a copy
    safeCopy := make([]string, len(data))
    copy(safeCopy, data)
    safeCopy[0] = "modified_task1"

    fmt.Printf("After safe modification:\n")
    fmt.Printf("  Original: %v\n", data)
    fmt.Printf("  Safe copy: %v\n", safeCopy)

    // Unsafe operation: direct slice sharing
    unsafeSlice := data[1:4]  // Shares backing array
    unsafeSlice[0] = "modified_task2"

    fmt.Printf("After unsafe modification:\n")
    fmt.Printf("  Original: %v\n", data)  // Original is affected!
    fmt.Printf("  Unsafe slice: %v\n", unsafeSlice)

    // Demonstrate append behavior with shared slices
    fmt.Println("\n=== Append with Shared Slices ===")

    baseData := []int{1, 2, 3, 4, 5}
    fmt.Printf("Base data: %v (len=%d, cap=%d)\n",
        baseData, len(baseData), cap(baseData))

    // Create a slice with room to grow
    subSlice := baseData[:3]  // [1, 2, 3] but cap=5
    fmt.Printf("Sub slice: %v (len=%d, cap=%d)\n",
        subSlice, len(subSlice), cap(subSlice))

    // Append to sub slice - this will overwrite baseData[3]!
    subSlice = append(subSlice, 99)

    fmt.Printf("After append(subSlice, 99):\n")
    fmt.Printf("  Base data: %v\n", baseData)  // baseData[3] is now 99!
    fmt.Printf("  Sub slice: %v\n", subSlice)

    // Safe append: use full slice expression to limit capacity
    safeSlice := baseData[:3:3]  // len=3, cap=3
    fmt.Printf("\nSafe slice [0:3:3]: %v (len=%d, cap=%d)\n",
        safeSlice, len(safeSlice), cap(safeSlice))

    // Reset base data
    baseData = []int{1, 2, 3, 4, 5}
    safeSlice = baseData[:3:3]

    // This append will allocate new backing array
    safeSlice = append(safeSlice, 99)

    fmt.Printf("After safe append:\n")
    fmt.Printf("  Base data: %v (unchanged)\n", baseData)
    fmt.Printf("  Safe slice: %v (new backing array)\n", safeSlice)
}
```

**Prerequisites**: Module 10
