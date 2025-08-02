# Module 12: Slices Part 2: Appending and Memory Management

**Duration**: 1 hour (45 min theory + 15 min hands-on)

**Learning Objectives**:

- Master slice growth and append mechanics
- Understand memory allocation patterns with slices
- Learn efficient slice management techniques
- Optimize slice usage for automation workloads

**Videos Covered**:

- 3.6 Slices Part 2 (Appending Slices) (0:15:32)

**Key Concepts**:

- append() function and slice growth
- Capacity doubling and memory allocation
- Slice reallocation and pointer invalidation
- Pre-allocation strategies for performance
- Memory efficiency with large slices

**Hands-on Exercise 1: Efficient Batch Processing with Slice Management**:

```go
// Efficient batch processing with slice management
package main

import (
    "fmt"
    "runtime"
)

type BatchProcessor struct {
    items    []string
    batches  [][]string
    batchSize int
}

func NewBatchProcessor(batchSize int) *BatchProcessor {
    return &BatchProcessor{
        items:     make([]string, 0, batchSize*10), // Pre-allocate
        batches:   make([][]string, 0, 10),
        batchSize: batchSize,
    }
}

func (bp *BatchProcessor) AddItem(item string) {
    bp.items = append(bp.items, item)

    // Create batch when we reach batch size
    if len(bp.items) >= bp.batchSize {
        bp.createBatch()
    }
}

func (bp *BatchProcessor) createBatch() {
    if len(bp.items) == 0 {
        return
    }

    // Create new slice for batch (copy to avoid sharing)
    batch := make([]string, len(bp.items))
    copy(batch, bp.items)

    bp.batches = append(bp.batches, batch)

    // Reset items slice but keep capacity
    bp.items = bp.items[:0]
}

func (bp *BatchProcessor) ProcessBatches() {
    // Process any remaining items
    if len(bp.items) > 0 {
        bp.createBatch()
    }

    fmt.Printf("Processing %d batches...\n", len(bp.batches))

    for i, batch := range bp.batches {
        fmt.Printf("Batch %d: %d items\n", i+1, len(batch))

        // Simulate processing
        for j, item := range batch {
            if j < 3 { // Show first 3 items
                fmt.Printf("  Processing: %s\n", item)
            } else if j == 3 {
                fmt.Printf("  ... and %d more items\n", len(batch)-3)
                break
            }
        }
    }
}

func (bp *BatchProcessor) GetMemoryStats() {
    var m runtime.MemStats
    runtime.ReadMemStats(&m)

    fmt.Printf("Memory stats:\n")
    fmt.Printf("  Items slice - len: %d, cap: %d\n", len(bp.items), cap(bp.items))
    fmt.Printf("  Batches - count: %d, cap: %d\n", len(bp.batches), cap(bp.batches))
    fmt.Printf("  Heap alloc: %d KB\n", m.HeapAlloc/1024)
}

func main() {
    processor := NewBatchProcessor(5)

    // Add items to processor
    for i := 0; i < 23; i++ {
        item := fmt.Sprintf("automation-task-%03d", i)
        processor.AddItem(item)

        if i%5 == 4 { // Every 5 items
            processor.GetMemoryStats()
            fmt.Println()
        }
    }

    // Process all batches
    processor.ProcessBatches()

    // Final memory stats
    fmt.Println("\nFinal stats:")
    processor.GetMemoryStats()
}
```

**Hands-on Exercise 2: Slice Growth Patterns and Memory Allocation**:

```go
// Demonstrate slice growth patterns and memory allocation strategies
package main

import (
    "fmt"
    "runtime"
)

func main() {
    demonstrateSliceGrowth()
    demonstratePreallocation()
    demonstrateMemoryEfficiency()
}

func demonstrateSliceGrowth() {
    fmt.Println("=== Slice Growth Patterns ===")

    var slice []int
    fmt.Printf("Initial: len=%d, cap=%d\n", len(slice), cap(slice))

    // Track capacity changes during growth
    for i := 0; i < 20; i++ {
        oldCap := cap(slice)
        slice = append(slice, i)
        newCap := cap(slice)

        if newCap != oldCap {
            fmt.Printf("After append(%d): len=%d, cap=%d (grew from %d)\n",
                i, len(slice), newCap, oldCap)
        }
    }

    // Show the growth pattern
    fmt.Println("\nGrowth pattern analysis:")
    fmt.Println("- Capacity starts at 0")
    fmt.Println("- First allocation: cap=1")
    fmt.Println("- Then doubles: 1→’2→’4→’8→’16→’32...")
    fmt.Println("- Growth factor may vary for large slices")
}

func demonstratePreallocation() {
    fmt.Println("\n=== Pre-allocation vs Dynamic Growth ===")

    const itemCount = 10000

    // Method 1: Dynamic growth (inefficient)
    var m1 runtime.MemStats
    runtime.GC()
    runtime.ReadMemStats(&m1)

    var dynamicSlice []int
    for i := 0; i < itemCount; i++ {
        dynamicSlice = append(dynamicSlice, i)
    }

    var m2 runtime.MemStats
    runtime.ReadMemStats(&m2)
    dynamicAllocs := m2.Mallocs - m1.Mallocs

    // Method 2: Pre-allocation (efficient)
    runtime.GC()
    runtime.ReadMemStats(&m1)

    preallocSlice := make([]int, 0, itemCount)
    for i := 0; i < itemCount; i++ {
        preallocSlice = append(preallocSlice, i)
    }

    runtime.ReadMemStats(&m2)
    preallocAllocs := m2.Mallocs - m1.Mallocs

    fmt.Printf("Dynamic growth: %d allocations\n", dynamicAllocs)
    fmt.Printf("Pre-allocation: %d allocations\n", preallocAllocs)
    fmt.Printf("Allocation reduction: %.1fx\n",
        float64(dynamicAllocs)/float64(preallocAllocs))

    // Method 3: Pre-sized slice
    runtime.GC()
    runtime.ReadMemStats(&m1)

    presizedSlice := make([]int, itemCount)
    for i := 0; i < itemCount; i++ {
        presizedSlice[i] = i
    }

    runtime.ReadMemStats(&m2)
    presizedAllocs := m2.Mallocs - m1.Mallocs

    fmt.Printf("Pre-sized slice: %d allocations\n", presizedAllocs)
}

func demonstrateMemoryEfficiency() {
    fmt.Println("\n=== Memory Efficiency Strategies ===")

    // Strategy 1: Reuse slices by resetting length
    reusableSlice := make([]string, 0, 100)

    for batch := 0; batch < 3; batch++ {
        fmt.Printf("\nBatch %d:\n", batch+1)

        // Add items to slice
        for i := 0; i < 5; i++ {
            item := fmt.Sprintf("batch%d-item%d", batch+1, i+1)
            reusableSlice = append(reusableSlice, item)
        }

        fmt.Printf("  After adding: len=%d, cap=%d\n",
            len(reusableSlice), cap(reusableSlice))

        // Process items (simulate)
        fmt.Printf("  Processing %d items...\n", len(reusableSlice))

        // Reset slice for reuse (keep capacity)
        reusableSlice = reusableSlice[:0]
        fmt.Printf("  After reset: len=%d, cap=%d\n",
            len(reusableSlice), cap(reusableSlice))
    }

    // Strategy 2: Efficient copying
    fmt.Println("\n=== Efficient Copying Strategies ===")

    source := []int{1, 2, 3, 4, 5, 6, 7, 8, 9, 10}

    // Method 1: Using copy() function
    dest1 := make([]int, len(source))
    copy(dest1, source)
    fmt.Printf("copy() method: %v\n", dest1)

    // Method 2: Using append with empty slice
    var dest2 []int
    dest2 = append(dest2, source...)
    fmt.Printf("append() method: %v\n", dest2)

    // Method 3: Manual copying (for partial copies)
    dest3 := make([]int, 5)
    for i := 0; i < 5; i++ {
        dest3[i] = source[i]
    }
    fmt.Printf("manual copy (first 5): %v\n", dest3)

    // Strategy 3: Avoiding memory leaks with large slices
    fmt.Println("\n=== Avoiding Memory Leaks ===")

    largeSlice := make([]byte, 1000000) // 1MB
    for i := range largeSlice {
        largeSlice[i] = byte(i % 256)
    }

    // BAD: This keeps the entire 1MB in memory
    badSubslice := largeSlice[0:10]
    fmt.Printf("Bad subslice: len=%d, cap=%d (keeps %d bytes)\n",
        len(badSubslice), cap(badSubslice), cap(badSubslice))

    // GOOD: Copy only what you need
    goodSubslice := make([]byte, 10)
    copy(goodSubslice, largeSlice[0:10])
    fmt.Printf("Good subslice: len=%d, cap=%d (uses %d bytes)\n",
        len(goodSubslice), cap(goodSubslice), cap(goodSubslice))

    // Now largeSlice can be garbage collected
    largeSlice = nil
    runtime.GC()

    fmt.Println("\nMemory efficiency tips:")
    fmt.Println("- Pre-allocate when size is known")
    fmt.Println("- Reuse slices by resetting length")
    fmt.Println("- Copy subslices from large slices")
    fmt.Println("- Use copy() for safe copying")
    fmt.Println("- Set large slices to nil when done")
}
```

**Hands-on Exercise 3: Advanced Append Patterns and Performance**:

```go
// Advanced append patterns for high-performance automation
package main

import (
    "fmt"
    "time"
)

type PerformanceTracker struct {
    operations []string
    timings    []time.Duration
    results    []bool
}

func NewPerformanceTracker() *PerformanceTracker {
    return &PerformanceTracker{
        operations: make([]string, 0, 1000),
        timings:    make([]time.Duration, 0, 1000),
        results:    make([]bool, 0, 1000),
    }
}

func (pt *PerformanceTracker) TrackOperation(name string, fn func() bool) {
    start := time.Now()
    result := fn()
    duration := time.Since(start)

    pt.operations = append(pt.operations, name)
    pt.timings = append(pt.timings, duration)
    pt.results = append(pt.results, result)
}

func (pt *PerformanceTracker) GetSummary() (int, int, time.Duration) {
    successful := 0
    var totalTime time.Duration

    for i, result := range pt.results {
        if result {
            successful++
        }
        totalTime += pt.timings[i]
    }

    return successful, len(pt.results), totalTime
}

func (pt *PerformanceTracker) GetFailures() []string {
    var failures []string

    for i, result := range pt.results {
        if !result {
            failures = append(failures, pt.operations[i])
        }
    }

    return failures
}

// Demonstrate different append patterns
func main() {
    demonstrateAppendPatterns()
    demonstrateSliceCapacityManagement()
    demonstrateBulkOperations()
}

func demonstrateAppendPatterns() {
    fmt.Println("=== Append Patterns ===")

    // Pattern 1: Single element append
    var single []int
    for i := 0; i < 5; i++ {
        single = append(single, i)
    }
    fmt.Printf("Single append: %v\n", single)

    // Pattern 2: Multiple element append
    var multiple []int
    multiple = append(multiple, 1, 2, 3, 4, 5)
    fmt.Printf("Multiple append: %v\n", multiple)

    // Pattern 3: Slice append (variadic)
    var slice1 []int = []int{1, 2, 3}
    var slice2 []int = []int{4, 5, 6}
    combined := append(slice1, slice2...)
    fmt.Printf("Slice append: %v\n", combined)

    // Pattern 4: Conditional append
    var filtered []int
    source := []int{1, 2, 3, 4, 5, 6, 7, 8, 9, 10}
    for _, v := range source {
        if v%2 == 0 { // Even numbers only
            filtered = append(filtered, v)
        }
    }
    fmt.Printf("Conditional append: %v\n", filtered)

    // Pattern 5: Append with transformation
    var transformed []string
    numbers := []int{1, 2, 3, 4, 5}
    for _, num := range numbers {
        transformed = append(transformed, fmt.Sprintf("item-%d", num))
    }
    fmt.Printf("Transform append: %v\n", transformed)
}

func demonstrateSliceCapacityManagement() {
    fmt.Println("\n=== Capacity Management ===")

    // Demonstrate capacity growth and reallocation
    tracker := NewPerformanceTracker()

    // Simulate various operations
    operations := []struct {
        name string
        fn   func() bool
    }{
        {"database_connect", func() bool { time.Sleep(10 * time.Millisecond); return true }},
        {"api_call_1", func() bool { time.Sleep(5 * time.Millisecond); return true }},
        {"file_process", func() bool { time.Sleep(15 * time.Millisecond); return false }},
        {"api_call_2", func() bool { time.Sleep(8 * time.Millisecond); return true }},
        {"cleanup", func() bool { time.Sleep(3 * time.Millisecond); return true }},
    }

    fmt.Println("Tracking operations...")
    for _, op := range operations {
        tracker.TrackOperation(op.name, op.fn)
        fmt.Printf("  %s: len=%d, cap=%d\n",
            op.name, len(tracker.operations), cap(tracker.operations))
    }

    successful, total, totalTime := tracker.GetSummary()
    fmt.Printf("\nSummary: %d/%d successful, total time: %v\n",
        successful, total, totalTime)

    failures := tracker.GetFailures()
    if len(failures) > 0 {
        fmt.Printf("Failures: %v\n", failures)
    }
}

func demonstrateBulkOperations() {
    fmt.Println("\n=== Bulk Operations ===")

    // Efficient bulk append vs individual appends
    const itemCount = 1000

    // Method 1: Individual appends (less efficient)
    start := time.Now()
    var individual []int
    for i := 0; i < itemCount; i++ {
        individual = append(individual, i)
    }
    individualTime := time.Since(start)

    // Method 2: Bulk append with pre-allocation
    start = time.Now()
    bulk := make([]int, 0, itemCount)
    batch := make([]int, 100) // Prepare batch
    for i := 0; i < itemCount; i += 100 {
        // Fill batch
        batchSize := 100
        if i+100 > itemCount {
            batchSize = itemCount - i
        }

        for j := 0; j < batchSize; j++ {
            batch[j] = i + j
        }

        // Bulk append
        bulk = append(bulk, batch[:batchSize]...)
    }
    bulkTime := time.Since(start)

    fmt.Printf("Individual appends: %v\n", individualTime)
    fmt.Printf("Bulk appends: %v\n", bulkTime)
    fmt.Printf("Bulk is %.2fx faster\n",
        float64(individualTime)/float64(bulkTime))

    // Demonstrate append with capacity monitoring
    fmt.Println("\n=== Capacity Monitoring ===")

    var monitored []string
    capacityChanges := 0
    lastCap := cap(monitored)

    for i := 0; i < 20; i++ {
        monitored = append(monitored, fmt.Sprintf("item-%d", i))

        currentCap := cap(monitored)
        if currentCap != lastCap {
            capacityChanges++
            fmt.Printf("Capacity changed at item %d: %d →’ %d\n",
                i, lastCap, currentCap)
            lastCap = currentCap
        }
    }

    fmt.Printf("Total capacity changes: %d\n", capacityChanges)
    fmt.Printf("Final: len=%d, cap=%d\n", len(monitored), cap(monitored))

    // Show memory efficiency of different approaches
    fmt.Println("\n=== Memory Efficiency Comparison ===")

    // Approach 1: No pre-allocation
    var noPrealloc []int
    for i := 0; i < 100; i++ {
        noPrealloc = append(noPrealloc, i)
    }
    fmt.Printf("No pre-allocation: len=%d, cap=%d, efficiency=%.1f%%\n",
        len(noPrealloc), cap(noPrealloc),
        float64(len(noPrealloc))/float64(cap(noPrealloc))*100)

    // Approach 2: Exact pre-allocation
    exactPrealloc := make([]int, 0, 100)
    for i := 0; i < 100; i++ {
        exactPrealloc = append(exactPrealloc, i)
    }
    fmt.Printf("Exact pre-allocation: len=%d, cap=%d, efficiency=%.1f%%\n",
        len(exactPrealloc), cap(exactPrealloc),
        float64(len(exactPrealloc))/float64(cap(exactPrealloc))*100)

    // Approach 3: Over pre-allocation
    overPrealloc := make([]int, 0, 200)
    for i := 0; i < 100; i++ {
        overPrealloc = append(overPrealloc, i)
    }
    fmt.Printf("Over pre-allocation: len=%d, cap=%d, efficiency=%.1f%%\n",
        len(overPrealloc), cap(overPrealloc),
        float64(len(overPrealloc))/float64(cap(overPrealloc))*100)
}
```

**Prerequisites**: Module 11
