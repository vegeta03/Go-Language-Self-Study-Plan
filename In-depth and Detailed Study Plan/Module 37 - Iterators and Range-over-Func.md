# Module 37: Iterators and Range-over-Func

**Duration**: 1 hour (45 min theory + 15 min hands-on)

**Learning Objectives**:

- Master Go 1.23+ range-over-func iterator patterns
- Understand iterator function signatures and yield semantics
- Learn to create custom iterators for automation data processing
- Apply iterator patterns to streaming data and lazy evaluation
- Implement efficient data pipeline processing with iterators
- Design composable iterator chains for complex data transformations

**Videos Covered**:

- Go 1.23 range-over-func feature introduction
- Iterator patterns and best practices
- Standard library iterator functions (Go 1.24+)

**Key Concepts**:

- Range-over-func: `for range` with function types `func(yield func(T) bool)`
- Iterator function signatures: single and two-value iterators
- Yield function: controlling iteration flow with boolean return values
- Lazy evaluation: processing data on-demand rather than eagerly
- Iterator composition: chaining iterators for complex transformations
- Standard library iterators: `strings.Lines`, `bytes.SplitSeq`, etc.
- Memory efficiency: avoiding intermediate slice allocations
- Early termination: breaking out of iterator loops efficiently

**Hands-on Exercise 1: Custom Iterators for Automation Data**:

Building custom iterators for processing automation system data streams:

```go
// Custom iterators for automation data processing
package iterators

import (
    "bufio"
    "context"
    "fmt"
    "io"
    "strconv"
    "strings"
    "time"
)

// LogEntry represents a log entry in automation systems
type LogEntry struct {
    Timestamp time.Time
    Level     string
    Service   string
    Message   string
    Metadata  map[string]string
}

// LogIterator creates an iterator over log entries from a reader
func LogIterator(reader io.Reader) func(yield func(LogEntry, error) bool) {
    return func(yield func(LogEntry, error) bool) {
        scanner := bufio.NewScanner(reader)
        
        for scanner.Scan() {
            line := scanner.Text()
            entry, err := parseLogEntry(line)
            
            // Yield the entry and error (if any)
            if !yield(entry, err) {
                return // Early termination requested
            }
        }
        
        // Check for scanner errors
        if err := scanner.Err(); err != nil {
            var zero LogEntry
            yield(zero, fmt.Errorf("scanner error: %w", err))
        }
    }
}

// parseLogEntry parses a log line into a LogEntry
func parseLogEntry(line string) (LogEntry, error) {
    parts := strings.SplitN(line, " ", 4)
    if len(parts) < 4 {
        return LogEntry{}, fmt.Errorf("invalid log format: %s", line)
    }
    
    timestamp, err := time.Parse(time.RFC3339, parts[0])
    if err != nil {
        return LogEntry{}, fmt.Errorf("invalid timestamp: %w", err)
    }
    
    return LogEntry{
        Timestamp: timestamp,
        Level:     parts[1],
        Service:   parts[2],
        Message:   parts[3],
        Metadata:  make(map[string]string),
    }, nil
}

// MetricPoint represents a metric data point
type MetricPoint struct {
    Name      string
    Value     float64
    Timestamp time.Time
    Tags      map[string]string
}

// MetricIterator creates an iterator over metric points
func MetricIterator(metrics []MetricPoint) func(yield func(MetricPoint) bool) {
    return func(yield func(MetricPoint) bool) {
        for _, metric := range metrics {
            if !yield(metric) {
                return
            }
        }
    }
}

// TimeWindowIterator creates an iterator that groups items by time windows
func TimeWindowIterator[T any](
    source func(yield func(T) bool),
    windowSize time.Duration,
    getTime func(T) time.Time,
) func(yield func([]T) bool) {
    return func(yield func([]T) bool) {
        var currentWindow []T
        var windowStart time.Time
        
        for item := range source {
            itemTime := getTime(item)
            
            // Initialize window start time
            if windowStart.IsZero() {
                windowStart = itemTime.Truncate(windowSize)
            }
            
            // Check if item belongs to current window
            if itemTime.Before(windowStart.Add(windowSize)) {
                currentWindow = append(currentWindow, item)
            } else {
                // Yield current window if not empty
                if len(currentWindow) > 0 {
                    if !yield(currentWindow) {
                        return
                    }
                }
                
                // Start new window
                windowStart = itemTime.Truncate(windowSize)
                currentWindow = []T{item}
            }
        }
        
        // Yield final window if not empty
        if len(currentWindow) > 0 {
            yield(currentWindow)
        }
    }
}

// FilterIterator creates an iterator that filters items based on a predicate
func FilterIterator[T any](
    source func(yield func(T) bool),
    predicate func(T) bool,
) func(yield func(T) bool) {
    return func(yield func(T) bool) {
        for item := range source {
            if predicate(item) {
                if !yield(item) {
                    return
                }
            }
        }
    }
}

// MapIterator creates an iterator that transforms items
func MapIterator[T, U any](
    source func(yield func(T) bool),
    transform func(T) U,
) func(yield func(U) bool) {
    return func(yield func(U) bool) {
        for item := range source {
            transformed := transform(item)
            if !yield(transformed) {
                return
            }
        }
    }
}

// TakeIterator creates an iterator that takes only the first n items
func TakeIterator[T any](
    source func(yield func(T) bool),
    n int,
) func(yield func(T) bool) {
    return func(yield func(T) bool) {
        count := 0
        for item := range source {
            if count >= n {
                return
            }
            
            if !yield(item) {
                return
            }
            count++
        }
    }
}

// SkipIterator creates an iterator that skips the first n items
func SkipIterator[T any](
    source func(yield func(T) bool),
    n int,
) func(yield func(T) bool) {
    return func(yield func(T) bool) {
        count := 0
        for item := range source {
            if count < n {
                count++
                continue
            }
            
            if !yield(item) {
                return
            }
        }
    }
}

// ChainIterator creates an iterator that chains multiple iterators
func ChainIterator[T any](
    iterators ...func(yield func(T) bool),
) func(yield func(T) bool) {
    return func(yield func(T) bool) {
        for _, iterator := range iterators {
            for item := range iterator {
                if !yield(item) {
                    return
                }
            }
        }
    }
}

// BatchIterator creates an iterator that groups items into batches
func BatchIterator[T any](
    source func(yield func(T) bool),
    batchSize int,
) func(yield func([]T) bool) {
    return func(yield func([]T) bool) {
        var batch []T
        
        for item := range source {
            batch = append(batch, item)
            
            if len(batch) >= batchSize {
                if !yield(batch) {
                    return
                }
                batch = nil // Reset batch
            }
        }
        
        // Yield final batch if not empty
        if len(batch) > 0 {
            yield(batch)
        }
    }
}

// ParallelIterator processes items in parallel and yields results
func ParallelIterator[T, U any](
    source func(yield func(T) bool),
    workers int,
    processor func(T) (U, error),
) func(yield func(U, error) bool) {
    return func(yield func(U, error) bool) {
        jobs := make(chan T, workers*2)
        results := make(chan result[U], workers*2)
        
        // Start workers
        for i := 0; i < workers; i++ {
            go func() {
                for job := range jobs {
                    result, err := processor(job)
                    results <- result[U]{value: result, err: err}
                }
            }()
        }
        
        // Send jobs
        go func() {
            defer close(jobs)
            for item := range source {
                jobs <- item
            }
        }()
        
        // Collect and yield results
        itemCount := 0
        for item := range source {
            itemCount++
        }
        
        for i := 0; i < itemCount; i++ {
            result := <-results
            if !yield(result.value, result.err) {
                return
            }
        }
    }
}

// result holds a value and error pair
type result[T any] struct {
    value T
    err   error
}
```

**Hands-on Exercise 2: Data Pipeline Processing with Iterators**:

Building efficient data processing pipelines using iterator composition:

```go
// Data pipeline processing with iterator composition
package pipeline

import (
    "context"
    "fmt"
    "strconv"
    "strings"
    "time"
)

// SensorReading represents a sensor data reading
type SensorReading struct {
    SensorID  string
    Value     float64
    Timestamp time.Time
    Quality   string
}

// ProcessedReading represents a processed sensor reading
type ProcessedReading struct {
    SensorID    string
    Value       float64
    Timestamp   time.Time
    Quality     string
    Anomaly     bool
    Smoothed    float64
    Derivative  float64
}

// SensorDataPipeline processes sensor data using iterator composition
type SensorDataPipeline struct {
    windowSize     time.Duration
    anomalyThreshold float64
    smoothingFactor  float64
}

// NewSensorDataPipeline creates a new sensor data pipeline
func NewSensorDataPipeline(windowSize time.Duration, anomalyThreshold, smoothingFactor float64) *SensorDataPipeline {
    return &SensorDataPipeline{
        windowSize:       windowSize,
        anomalyThreshold: anomalyThreshold,
        smoothingFactor:  smoothingFactor,
    }
}

// Process processes sensor readings through the pipeline
func (sdp *SensorDataPipeline) Process(
    ctx context.Context,
    readings func(yield func(SensorReading) bool),
) func(yield func(ProcessedReading) bool) {
    
    // Step 1: Filter out invalid readings
    validReadings := FilterIterator(readings, func(r SensorReading) bool {
        return r.Quality == "GOOD" && r.Value >= 0
    })
    
    // Step 2: Group by time windows
    windowedReadings := TimeWindowIterator(
        validReadings,
        sdp.windowSize,
        func(r SensorReading) time.Time { return r.Timestamp },
    )
    
    // Step 3: Process each window
    return func(yield func(ProcessedReading) bool) {
        var previousValues = make(map[string]float64)
        var smoothedValues = make(map[string]float64)
        
        for window := range windowedReadings {
            // Check context cancellation
            select {
            case <-ctx.Done():
                return
            default:
            }
            
            processedWindow := sdp.processWindow(window, previousValues, smoothedValues)
            
            for _, processed := range processedWindow {
                if !yield(processed) {
                    return
                }
            }
        }
    }
}

// processWindow processes a window of sensor readings
func (sdp *SensorDataPipeline) processWindow(
    window []SensorReading,
    previousValues, smoothedValues map[string]float64,
) []ProcessedReading {
    
    var processed []ProcessedReading
    
    for _, reading := range window {
        processedReading := ProcessedReading{
            SensorID:  reading.SensorID,
            Value:     reading.Value,
            Timestamp: reading.Timestamp,
            Quality:   reading.Quality,
        }
        
        // Calculate smoothed value using exponential smoothing
        if prevSmoothed, exists := smoothedValues[reading.SensorID]; exists {
            processedReading.Smoothed = sdp.smoothingFactor*reading.Value + 
                                      (1-sdp.smoothingFactor)*prevSmoothed
        } else {
            processedReading.Smoothed = reading.Value
        }
        smoothedValues[reading.SensorID] = processedReading.Smoothed
        
        // Calculate derivative (rate of change)
        if prevValue, exists := previousValues[reading.SensorID]; exists {
            processedReading.Derivative = reading.Value - prevValue
        }
        previousValues[reading.SensorID] = reading.Value
        
        // Detect anomalies
        if abs(processedReading.Derivative) > sdp.anomalyThreshold {
            processedReading.Anomaly = true
        }
        
        processed = append(processed, processedReading)
    }
    
    return processed
}

// abs returns the absolute value of a float64
func abs(x float64) float64 {
    if x < 0 {
        return -x
    }
    return x
}

// CSVIterator creates an iterator over CSV data
func CSVIterator(csvData string) func(yield func([]string) bool) {
    return func(yield func([]string) bool) {
        lines := strings.Split(csvData, "\n")
        
        for _, line := range lines {
            line = strings.TrimSpace(line)
            if line == "" {
                continue
            }
            
            fields := strings.Split(line, ",")
            // Trim whitespace from fields
            for i, field := range fields {
                fields[i] = strings.TrimSpace(field)
            }
            
            if !yield(fields) {
                return
            }
        }
    }
}

// SensorReadingFromCSV converts CSV fields to SensorReading
func SensorReadingFromCSV(fields []string) (SensorReading, error) {
    if len(fields) < 4 {
        return SensorReading{}, fmt.Errorf("insufficient fields: %d", len(fields))
    }
    
    value, err := strconv.ParseFloat(fields[1], 64)
    if err != nil {
        return SensorReading{}, fmt.Errorf("invalid value: %w", err)
    }
    
    timestamp, err := time.Parse(time.RFC3339, fields[2])
    if err != nil {
        return SensorReading{}, fmt.Errorf("invalid timestamp: %w", err)
    }
    
    return SensorReading{
        SensorID:  fields[0],
        Value:     value,
        Timestamp: timestamp,
        Quality:   fields[3],
    }, nil
}

// AggregateIterator creates an iterator that aggregates values
func AggregateIterator[T any, K comparable, V any](
    source func(yield func(T) bool),
    keyFunc func(T) K,
    valueFunc func(T) V,
    aggregateFunc func(V, V) V,
) func(yield func(K, V) bool) {
    return func(yield func(K, V) bool) {
        aggregates := make(map[K]V)
        
        for item := range source {
            key := keyFunc(item)
            value := valueFunc(item)
            
            if existing, exists := aggregates[key]; exists {
                aggregates[key] = aggregateFunc(existing, value)
            } else {
                aggregates[key] = value
            }
        }
        
        // Yield all aggregated results
        for key, value := range aggregates {
            if !yield(key, value) {
                return
            }
        }
    }
}

// DistinctIterator creates an iterator that yields only distinct items
func DistinctIterator[T comparable](
    source func(yield func(T) bool),
) func(yield func(T) bool) {
    return func(yield func(T) bool) {
        seen := make(map[T]bool)
        
        for item := range source {
            if !seen[item] {
                seen[item] = true
                if !yield(item) {
                    return
                }
            }
        }
    }
}

// SortedMergeIterator merges multiple sorted iterators
func SortedMergeIterator[T any](
    compare func(T, T) int,
    iterators ...func(yield func(T) bool),
) func(yield func(T) bool) {
    return func(yield func(T) bool) {
        // This is a simplified implementation
        // In practice, you'd use a priority queue for efficiency
        var allItems []T
        
        // Collect all items
        for _, iterator := range iterators {
            for item := range iterator {
                allItems = append(allItems, item)
            }
        }
        
        // Simple bubble sort (for demonstration)
        for i := 0; i < len(allItems); i++ {
            for j := 0; j < len(allItems)-1-i; j++ {
                if compare(allItems[j], allItems[j+1]) > 0 {
                    allItems[j], allItems[j+1] = allItems[j+1], allItems[j]
                }
            }
        }
        
        // Yield sorted items
        for _, item := range allItems {
            if !yield(item) {
                return
            }
        }
    }
}
```

**Hands-on Exercise 3: Advanced Iterator Patterns**:

Implementing advanced iterator patterns for complex automation scenarios:

```go
// Advanced iterator patterns for automation systems
package advanced

import (
    "context"
    "sync"
    "time"
)

// Event represents an automation system event
type Event struct {
    ID        string
    Type      string
    Timestamp time.Time
    Data      map[string]interface{}
    Source    string
}

// EventStream provides real-time event iteration
type EventStream struct {
    events chan Event
    done   chan struct{}
    mu     sync.RWMutex
    closed bool
}

// NewEventStream creates a new event stream
func NewEventStream() *EventStream {
    return &EventStream{
        events: make(chan Event, 100),
        done:   make(chan struct{}),
    }
}

// Publish adds an event to the stream
func (es *EventStream) Publish(event Event) bool {
    es.mu.RLock()
    defer es.mu.RUnlock()
    
    if es.closed {
        return false
    }
    
    select {
    case es.events <- event:
        return true
    default:
        return false // Channel full
    }
}

// Iterator returns an iterator over the event stream
func (es *EventStream) Iterator(ctx context.Context) func(yield func(Event) bool) {
    return func(yield func(Event) bool) {
        for {
            select {
            case event := <-es.events:
                if !yield(event) {
                    return
                }
            case <-es.done:
                return
            case <-ctx.Done():
                return
            }
        }
    }
}

// Close closes the event stream
func (es *EventStream) Close() {
    es.mu.Lock()
    defer es.mu.Unlock()
    
    if !es.closed {
        es.closed = true
        close(es.done)
        close(es.events)
    }
}

// BufferedIterator creates an iterator with buffering
func BufferedIterator[T any](
    source func(yield func(T) bool),
    bufferSize int,
) func(yield func(T) bool) {
    return func(yield func(T) bool) {
        buffer := make(chan T, bufferSize)
        done := make(chan struct{})
        
        // Fill buffer in background
        go func() {
            defer close(buffer)
            for item := range source {
                select {
                case buffer <- item:
                case <-done:
                    return
                }
            }
        }()
        
        // Yield from buffer
        defer close(done)
        for item := range buffer {
            if !yield(item) {
                return
            }
        }
    }
}

// ThrottledIterator creates an iterator that throttles output
func ThrottledIterator[T any](
    source func(yield func(T) bool),
    interval time.Duration,
) func(yield func(T) bool) {
    return func(yield func(T) bool) {
        ticker := time.NewTicker(interval)
        defer ticker.Stop()
        
        for item := range source {
            <-ticker.C // Wait for next tick
            if !yield(item) {
                return
            }
        }
    }
}

// RetryIterator creates an iterator that retries failed operations
func RetryIterator[T, U any](
    source func(yield func(T) bool),
    processor func(T) (U, error),
    maxRetries int,
    retryDelay time.Duration,
) func(yield func(U, error) bool) {
    return func(yield func(U, error) bool) {
        for item := range source {
            var result U
            var err error
            
            for attempt := 0; attempt <= maxRetries; attempt++ {
                result, err = processor(item)
                if err == nil {
                    break
                }
                
                if attempt < maxRetries {
                    time.Sleep(retryDelay * time.Duration(attempt+1))
                }
            }
            
            if !yield(result, err) {
                return
            }
        }
    }
}

// CacheIterator creates an iterator with caching
func CacheIterator[T any](
    source func(yield func(T) bool),
    cacheSize int,
) func(yield func(T) bool) {
    return func(yield func(T) bool) {
        cache := make([]T, 0, cacheSize)
        
        // First pass: fill cache and yield
        for item := range source {
            if len(cache) < cacheSize {
                cache = append(cache, item)
            }
            
            if !yield(item) {
                return
            }
        }
        
        // Subsequent iterations can use cache
        // (This is a simplified example - real implementation would be more complex)
    }
}

// ConditionalIterator creates an iterator that changes behavior based on conditions
func ConditionalIterator[T any](
    source func(yield func(T) bool),
    condition func(T) bool,
    trueIterator func(T) func(yield func(T) bool),
    falseIterator func(T) func(yield func(T) bool),
) func(yield func(T) bool) {
    return func(yield func(T) bool) {
        for item := range source {
            var iterator func(yield func(T) bool)
            
            if condition(item) {
                iterator = trueIterator(item)
            } else {
                iterator = falseIterator(item)
            }
            
            for subItem := range iterator {
                if !yield(subItem) {
                    return
                }
            }
        }
    }
}

// StatefulIterator creates an iterator that maintains state
func StatefulIterator[T, S any](
    source func(yield func(T) bool),
    initialState S,
    stateFunc func(S, T) (S, T, bool), // (newState, output, shouldYield)
) func(yield func(T) bool) {
    return func(yield func(T) bool) {
        state := initialState
        
        for item := range source {
            var output T
            var shouldYield bool
            
            state, output, shouldYield = stateFunc(state, item)
            
            if shouldYield {
                if !yield(output) {
                    return
                }
            }
        }
    }
}

// MulticastIterator creates an iterator that sends items to multiple consumers
func MulticastIterator[T any](
    source func(yield func(T) bool),
    consumers int,
) []func(yield func(T) bool) {
    channels := make([]chan T, consumers)
    for i := range channels {
        channels[i] = make(chan T, 10) // Buffered channels
    }
    
    // Start broadcaster
    go func() {
        defer func() {
            for _, ch := range channels {
                close(ch)
            }
        }()
        
        for item := range source {
            for _, ch := range channels {
                select {
                case ch <- item:
                default:
                    // Skip if channel is full
                }
            }
        }
    }()
    
    // Create iterators for each consumer
    iterators := make([]func(yield func(T) bool), consumers)
    for i, ch := range channels {
        ch := ch // Capture for closure
        iterators[i] = func(yield func(T) bool) {
            for item := range ch {
                if !yield(item) {
                    return
                }
            }
        }
    }
    
    return iterators
}
```

**Prerequisites**: Sessions 1-36 (Complete Go fundamentals and generics)

**Next Steps**: Module 38 (Modern Go Development Practices and Tooling)
