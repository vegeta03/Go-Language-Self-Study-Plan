# Module 36: Go Generics and Type Parameters

**Duration**: 1 hour (45 min theory + 15 min hands-on)

**Learning Objectives**:

- Master Go generics syntax and type parameter constraints
- Understand type inference and constraint satisfaction
- Learn generic function and type design patterns
- Apply generics to automation system components for type safety and reusability
- Implement generic data structures and algorithms
- Design clean generic APIs with proper constraint boundaries

**Videos Covered**:

- Modern Go generics concepts (Go 1.18+)
- Type parameters and constraints best practices
- Generic type aliases (Go 1.24+)

**Key Concepts**:

- Type parameters: defining generic functions and types with `[T any]` syntax
- Type constraints: using interfaces to constrain type parameters
- Built-in constraints: `any`, `comparable`, and constraint packages
- Type inference: automatic type parameter deduction from function arguments
- Generic type aliases: parameterized type aliases for cleaner APIs
- Constraint satisfaction: ensuring types meet interface requirements
- Generic instantiation: creating concrete types from generic definitions
- Performance considerations: compile-time vs runtime type checking

**Hands-on Exercise 1: Generic Data Structures for Automation**:

Building type-safe generic data structures for automation systems:

```go
// Generic data structures for automation systems
package generics

import (
    "context"
    "fmt"
    "sync"
    "time"
)

// Constraint interfaces for automation data types
type Identifiable interface {
    GetID() string
}

type Timestamped interface {
    GetTimestamp() time.Time
}

type AutomationData interface {
    Identifiable
    Timestamped
    comparable
}

// Generic cache for automation data
type Cache[T AutomationData] struct {
    mu    sync.RWMutex
    data  map[string]T
    ttl   time.Duration
    timer *time.Timer
}

// NewCache creates a new generic cache
func NewCache[T AutomationData](ttl time.Duration) *Cache[T] {
    cache := &Cache[T]{
        data: make(map[string]T),
        ttl:  ttl,
    }
    
    // Start cleanup timer
    cache.startCleanup()
    return cache
}

// Set stores an item in the cache
func (c *Cache[T]) Set(item T) {
    c.mu.Lock()
    defer c.mu.Unlock()
    
    c.data[item.GetID()] = item
}

// Get retrieves an item from the cache
func (c *Cache[T]) Get(id string) (T, bool) {
    c.mu.RLock()
    defer c.mu.RUnlock()
    
    item, exists := c.data[id]
    if !exists {
        var zero T
        return zero, false
    }
    
    // Check if item has expired
    if time.Since(item.GetTimestamp()) > c.ttl {
        var zero T
        return zero, false
    }
    
    return item, true
}

// GetAll returns all non-expired items
func (c *Cache[T]) GetAll() []T {
    c.mu.RLock()
    defer c.mu.RUnlock()
    
    var result []T
    now := time.Now()
    
    for _, item := range c.data {
        if now.Sub(item.GetTimestamp()) <= c.ttl {
            result = append(result, item)
        }
    }
    
    return result
}

// startCleanup starts the background cleanup process
func (c *Cache[T]) startCleanup() {
    c.timer = time.AfterFunc(c.ttl/2, func() {
        c.cleanup()
        c.startCleanup() // Reschedule
    })
}

// cleanup removes expired items
func (c *Cache[T]) cleanup() {
    c.mu.Lock()
    defer c.mu.Unlock()
    
    now := time.Now()
    for id, item := range c.data {
        if now.Sub(item.GetTimestamp()) > c.ttl {
            delete(c.data, id)
        }
    }
}

// Generic queue for automation tasks
type Queue[T any] struct {
    mu    sync.Mutex
    items []T
    cond  *sync.Cond
}

// NewQueue creates a new generic queue
func NewQueue[T any]() *Queue[T] {
    q := &Queue[T]{}
    q.cond = sync.NewCond(&q.mu)
    return q
}

// Enqueue adds an item to the queue
func (q *Queue[T]) Enqueue(item T) {
    q.mu.Lock()
    defer q.mu.Unlock()
    
    q.items = append(q.items, item)
    q.cond.Signal()
}

// Dequeue removes and returns an item from the queue
func (q *Queue[T]) Dequeue(ctx context.Context) (T, error) {
    q.mu.Lock()
    defer q.mu.Unlock()
    
    for len(q.items) == 0 {
        // Check context cancellation
        select {
        case <-ctx.Done():
            var zero T
            return zero, ctx.Err()
        default:
        }
        
        q.cond.Wait()
    }
    
    item := q.items[0]
    q.items = q.items[1:]
    return item, nil
}

// Size returns the current queue size
func (q *Queue[T]) Size() int {
    q.mu.Lock()
    defer q.mu.Unlock()
    return len(q.items)
}

// Generic result type for automation operations
type Result[T any, E error] struct {
    value T
    err   E
}

// NewResult creates a successful result
func NewResult[T any, E error](value T) Result[T, E] {
    return Result[T, E]{value: value}
}

// NewError creates an error result
func NewError[T any, E error](err E) Result[T, E] {
    var zero T
    return Result[T, E]{value: zero, err: err}
}

// IsSuccess returns true if the result is successful
func (r Result[T, E]) IsSuccess() bool {
    return r.err == nil
}

// Value returns the result value and error
func (r Result[T, E]) Value() (T, E) {
    return r.value, r.err
}

// Map transforms the result value if successful
func Map[T, U any, E error](r Result[T, E], fn func(T) U) Result[U, E] {
    if r.err != nil {
        return NewError[U, E](r.err)
    }
    return NewResult[U, E](fn(r.value))
}

// FlatMap chains result operations
func FlatMap[T, U any, E error](r Result[T, E], fn func(T) Result[U, E]) Result[U, E] {
    if r.err != nil {
        return NewError[U, E](r.err)
    }
    return fn(r.value)
}
```

**Hands-on Exercise 2: Generic Functions and Algorithms**:

Implementing generic utility functions and algorithms for automation systems:

```go
// Generic utility functions and algorithms
package algorithms

import (
    "context"
    "fmt"
    "sync"
)

// Ordered constraint for sortable types
type Ordered interface {
    ~int | ~int8 | ~int16 | ~int32 | ~int64 |
    ~uint | ~uint8 | ~uint16 | ~uint32 | ~uint64 |
    ~float32 | ~float64 | ~string
}

// Numeric constraint for mathematical operations
type Numeric interface {
    ~int | ~int8 | ~int16 | ~int32 | ~int64 |
    ~uint | ~uint8 | ~uint16 | ~uint32 | ~uint64 |
    ~float32 | ~float64
}

// Filter applies a predicate function to filter a slice
func Filter[T any](slice []T, predicate func(T) bool) []T {
    var result []T
    for _, item := range slice {
        if predicate(item) {
            result = append(result, item)
        }
    }
    return result
}

// Map transforms each element in a slice
func Map[T, U any](slice []T, transform func(T) U) []U {
    result := make([]U, len(slice))
    for i, item := range slice {
        result[i] = transform(item)
    }
    return result
}

// Reduce aggregates slice elements into a single value
func Reduce[T, U any](slice []T, initial U, reducer func(U, T) U) U {
    result := initial
    for _, item := range slice {
        result = reducer(result, item)
    }
    return result
}

// Find returns the first element matching the predicate
func Find[T any](slice []T, predicate func(T) bool) (T, bool) {
    for _, item := range slice {
        if predicate(item) {
            return item, true
        }
    }
    var zero T
    return zero, false
}

// GroupBy groups slice elements by a key function
func GroupBy[T any, K comparable](slice []T, keyFunc func(T) K) map[K][]T {
    groups := make(map[K][]T)
    for _, item := range slice {
        key := keyFunc(item)
        groups[key] = append(groups[key], item)
    }
    return groups
}

// Partition splits a slice into two based on a predicate
func Partition[T any](slice []T, predicate func(T) bool) ([]T, []T) {
    var trueSlice, falseSlice []T
    for _, item := range slice {
        if predicate(item) {
            trueSlice = append(trueSlice, item)
        } else {
            falseSlice = append(falseSlice, item)
        }
    }
    return trueSlice, falseSlice
}

// Unique removes duplicate elements from a slice
func Unique[T comparable](slice []T) []T {
    seen := make(map[T]bool)
    var result []T
    
    for _, item := range slice {
        if !seen[item] {
            seen[item] = true
            result = append(result, item)
        }
    }
    return result
}

// Max returns the maximum value in a slice
func Max[T Ordered](slice []T) (T, error) {
    if len(slice) == 0 {
        var zero T
        return zero, fmt.Errorf("empty slice")
    }
    
    max := slice[0]
    for _, item := range slice[1:] {
        if item > max {
            max = item
        }
    }
    return max, nil
}

// Min returns the minimum value in a slice
func Min[T Ordered](slice []T) (T, error) {
    if len(slice) == 0 {
        var zero T
        return zero, fmt.Errorf("empty slice")
    }
    
    min := slice[0]
    for _, item := range slice[1:] {
        if item < min {
            min = item
        }
    }
    return min, nil
}

// Sum calculates the sum of numeric values
func Sum[T Numeric](slice []T) T {
    var sum T
    for _, item := range slice {
        sum += item
    }
    return sum
}

// Average calculates the average of numeric values
func Average[T Numeric](slice []T) (float64, error) {
    if len(slice) == 0 {
        return 0, fmt.Errorf("empty slice")
    }
    
    sum := Sum(slice)
    return float64(sum) / float64(len(slice)), nil
}

// Parallel processing with generics
type ParallelProcessor[T, U any] struct {
    workers int
}

// NewParallelProcessor creates a new parallel processor
func NewParallelProcessor[T, U any](workers int) *ParallelProcessor[T, U] {
    return &ParallelProcessor[T, U]{workers: workers}
}

// Process processes items in parallel
func (pp *ParallelProcessor[T, U]) Process(ctx context.Context, items []T, 
                                          processor func(T) (U, error)) ([]U, error) {
    if len(items) == 0 {
        return nil, nil
    }
    
    jobs := make(chan T, len(items))
    results := make(chan indexedResult[U], len(items))
    
    // Start workers
    var wg sync.WaitGroup
    for i := 0; i < pp.workers; i++ {
        wg.Add(1)
        go func() {
            defer wg.Done()
            for {
                select {
                case item, ok := <-jobs:
                    if !ok {
                        return
                    }
                    
                    result, err := processor(item)
                    results <- indexedResult[U]{result: result, err: err}
                    
                case <-ctx.Done():
                    return
                }
            }
        }()
    }
    
    // Send jobs
    go func() {
        defer close(jobs)
        for _, item := range items {
            select {
            case jobs <- item:
            case <-ctx.Done():
                return
            }
        }
    }()
    
    // Collect results
    go func() {
        wg.Wait()
        close(results)
    }()
    
    var finalResults []U
    for result := range results {
        if result.err != nil {
            return nil, result.err
        }
        finalResults = append(finalResults, result.result)
    }
    
    return finalResults, nil
}

// indexedResult holds a result with its index
type indexedResult[T any] struct {
    result T
    err    error
}
```

**Hands-on Exercise 3: Generic Automation Workflows**:

Creating generic workflow systems for automation tasks:

```go
// Generic workflow system for automation
package workflow

import (
    "context"
    "fmt"
    "sync"
    "time"
)

// Step represents a workflow step with input and output types
type Step[In, Out any] interface {
    Execute(ctx context.Context, input In) (Out, error)
    Name() string
}

// Workflow represents a generic workflow
type Workflow[T any] struct {
    name  string
    steps []WorkflowStep[T]
    mu    sync.RWMutex
}

// WorkflowStep wraps a step with metadata
type WorkflowStep[T any] struct {
    step     Step[T, T]
    timeout  time.Duration
    retries  int
    parallel bool
}

// NewWorkflow creates a new generic workflow
func NewWorkflow[T any](name string) *Workflow[T] {
    return &Workflow[T]{
        name: name,
    }
}

// AddStep adds a step to the workflow
func (w *Workflow[T]) AddStep(step Step[T, T], options ...StepOption) *Workflow[T] {
    w.mu.Lock()
    defer w.mu.Unlock()
    
    workflowStep := WorkflowStep[T]{
        step:    step,
        timeout: 30 * time.Second, // Default timeout
        retries: 3,                // Default retries
    }
    
    // Apply options
    for _, option := range options {
        option(&workflowStep)
    }
    
    w.steps = append(w.steps, workflowStep)
    return w
}

// StepOption configures a workflow step
type StepOption func(*WorkflowStep[any])

// WithTimeout sets the step timeout
func WithTimeout[T any](timeout time.Duration) StepOption {
    return func(step *WorkflowStep[any]) {
        step.timeout = timeout
    }
}

// WithRetries sets the number of retries
func WithRetries[T any](retries int) StepOption {
    return func(step *WorkflowStep[any]) {
        step.retries = retries
    }
}

// WithParallel enables parallel execution
func WithParallel[T any](parallel bool) StepOption {
    return func(step *WorkflowStep[any]) {
        step.parallel = parallel
    }
}

// Execute runs the workflow
func (w *Workflow[T]) Execute(ctx context.Context, input T) (T, error) {
    w.mu.RLock()
    steps := make([]WorkflowStep[T], len(w.steps))
    copy(steps, w.steps)
    w.mu.RUnlock()
    
    current := input
    
    for i, workflowStep := range steps {
        stepCtx, cancel := context.WithTimeout(ctx, workflowStep.timeout)
        
        var result T
        var err error
        
        // Execute with retries
        for attempt := 0; attempt <= workflowStep.retries; attempt++ {
            result, err = workflowStep.step.Execute(stepCtx, current)
            if err == nil {
                break
            }
            
            if attempt < workflowStep.retries {
                select {
                case <-time.After(time.Duration(attempt+1) * time.Second):
                    continue
                case <-stepCtx.Done():
                    cancel()
                    return current, fmt.Errorf("step %d (%s) timeout: %w", 
                                             i, workflowStep.step.Name(), stepCtx.Err())
                }
            }
        }
        
        cancel()
        
        if err != nil {
            return current, fmt.Errorf("step %d (%s) failed after %d retries: %w", 
                                     i, workflowStep.step.Name(), workflowStep.retries, err)
        }
        
        current = result
    }
    
    return current, nil
}

// Concrete step implementations for automation

// ValidationStep validates automation data
type ValidationStep[T Validatable] struct {
    name string
}

// Validatable interface for data that can be validated
type Validatable interface {
    Validate() error
}

// NewValidationStep creates a new validation step
func NewValidationStep[T Validatable](name string) *ValidationStep[T] {
    return &ValidationStep[T]{name: name}
}

// Execute validates the input data
func (vs *ValidationStep[T]) Execute(ctx context.Context, input T) (T, error) {
    if err := input.Validate(); err != nil {
        return input, fmt.Errorf("validation failed: %w", err)
    }
    return input, nil
}

// Name returns the step name
func (vs *ValidationStep[T]) Name() string {
    return vs.name
}

// TransformStep transforms data from one type to another
type TransformStep[In, Out any] struct {
    name        string
    transformer func(In) (Out, error)
}

// NewTransformStep creates a new transform step
func NewTransformStep[In, Out any](name string, transformer func(In) (Out, error)) *TransformStep[In, Out] {
    return &TransformStep[In, Out]{
        name:        name,
        transformer: transformer,
    }
}

// Execute transforms the input data
func (ts *TransformStep[In, Out]) Execute(ctx context.Context, input In) (Out, error) {
    return ts.transformer(input)
}

// Name returns the step name
func (ts *TransformStep[In, Out]) Name() string {
    return ts.name
}

// BatchProcessor processes items in batches
type BatchProcessor[T any] struct {
    name      string
    batchSize int
    processor func([]T) ([]T, error)
}

// NewBatchProcessor creates a new batch processor
func NewBatchProcessor[T any](name string, batchSize int, 
                             processor func([]T) ([]T, error)) *BatchProcessor[T] {
    return &BatchProcessor[T]{
        name:      name,
        batchSize: batchSize,
        processor: processor,
    }
}

// Execute processes items in batches
func (bp *BatchProcessor[T]) Execute(ctx context.Context, input []T) ([]T, error) {
    if len(input) == 0 {
        return input, nil
    }
    
    var results []T
    
    for i := 0; i < len(input); i += bp.batchSize {
        end := i + bp.batchSize
        if end > len(input) {
            end = len(input)
        }
        
        batch := input[i:end]
        processed, err := bp.processor(batch)
        if err != nil {
            return results, fmt.Errorf("batch processing failed: %w", err)
        }
        
        results = append(results, processed...)
        
        // Check for cancellation
        select {
        case <-ctx.Done():
            return results, ctx.Err()
        default:
        }
    }
    
    return results, nil
}

// Name returns the processor name
func (bp *BatchProcessor[T]) Name() string {
    return bp.name
}
```

**Prerequisites**: Sessions 1-34 (Complete Go fundamentals)

**Next Steps**: Module 37 (Iterators and Range-over-Func) or Module 38 (Modern Go Development Practices)
