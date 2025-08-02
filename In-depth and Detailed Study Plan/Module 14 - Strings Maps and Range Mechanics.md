# Module 14: Strings, Maps, and Range Mechanics

**Duration**: 1 hour (45 min theory + 15 min hands-on)

**Learning Objectives**:

- Master string operations and UTF-8 handling
- Understand map operations and performance characteristics
- Learn range mechanics for different data types
- Apply strings and maps in automation configuration

**Videos Covered**:

- 3.9 Slices Part 5 (Strings and Slices) (0:08:29)
- 3.10 Slices Part 6 (Range Mechanics) (0:04:35)
- 3.11 Maps (0:08:03)

**Key Concepts**:

- String immutability and UTF-8 encoding
- String to []byte conversions and performance
- Map declaration, initialization, and operations
- Map performance characteristics and hash collisions
- Range over strings, slices, arrays, and maps

**Hands-on Exercise 1: Configuration Management with Strings and Maps**:

```go
// Configuration management with strings and maps
package main

import (
    "fmt"
    "strings"
    "unicode/utf8"
)

type ConfigManager struct {
    settings map[string]string
    metadata map[string]map[string]interface{}
}

func NewConfigManager() *ConfigManager {
    return &ConfigManager{
        settings: make(map[string]string),
        metadata: make(map[string]map[string]interface{}),
    }
}

func (cm *ConfigManager) SetConfig(key, value string) {
    cm.settings[key] = value

    // Store metadata about the configuration
    if cm.metadata[key] == nil {
        cm.metadata[key] = make(map[string]interface{})
    }

    cm.metadata[key]["length"] = utf8.RuneCountInString(value)
    cm.metadata[key]["bytes"] = len(value)
    cm.metadata[key]["type"] = cm.inferType(value)
}

func (cm *ConfigManager) inferType(value string) string {
    value = strings.TrimSpace(value)

    if value == "true" || value == "false" {
        return "boolean"
    }

    if strings.Contains(value, ".") {
        return "float"
    }

    // Check if it's a number
    for _, r := range value {
        if r < '0' || r > '9' {
            return "string"
        }
    }

    return "integer"
}

func (cm *ConfigManager) GetConfig(key string) (string, bool) {
    value, exists := cm.settings[key]
    return value, exists
}

func (cm *ConfigManager) ListConfigs() {
    fmt.Println("Configuration Settings:")

    // Range over map
    for key, value := range cm.settings {
        meta := cm.metadata[key]
        fmt.Printf("  %s = %s [type: %s, runes: %d, bytes: %d]\n",
            key, value, meta["type"], meta["length"], meta["bytes"])
    }
}

func (cm *ConfigManager) SearchConfigs(pattern string) []string {
    var matches []string

    pattern = strings.ToLower(pattern)

    for key, value := range cm.settings {
        // Search in key
        if strings.Contains(strings.ToLower(key), pattern) {
            matches = append(matches, key)
            continue
        }

        // Search in value
        if strings.Contains(strings.ToLower(value), pattern) {
            matches = append(matches, key)
        }
    }

    return matches
}

func (cm *ConfigManager) ProcessStringData(data string) map[string]int {
    wordCount := make(map[string]int)

    // Range over string (by rune)
    words := strings.Fields(data)
    for _, word := range words {
        // Clean word and count
        word = strings.ToLower(strings.Trim(word, ".,!?"))
        if word != "" {
            wordCount[word]++
        }
    }

    return wordCount
}

func main() {
    config := NewConfigManager()

    // Set various configuration values
    config.SetConfig("server_port", "8080")
    config.SetConfig("database_url", "postgresql://localhost:5432/mydb")
    config.SetConfig("enable_logging", "true")
    config.SetConfig("max_connections", "100")
    config.SetConfig("timeout_seconds", "30.5")
    config.SetConfig("service_name", "Automation Service 🔥š€")

    // List all configurations
    config.ListConfigs()

    // Search configurations
    fmt.Println("\nSearching for 'server':")
    matches := config.SearchConfigs("server")
    for _, match := range matches {
        if value, exists := config.GetConfig(match); exists {
            fmt.Printf("  Found: %s = %s\n", match, value)
        }
    }

    // Process text data
    logData := "Error connecting to database. Retrying connection. Connection successful."
    wordStats := config.ProcessStringData(logData)

    fmt.Println("\nWord frequency in log data:")
    for word, count := range wordStats {
        fmt.Printf("  '%s': %d\n", word, count)
    }
}
```

**Hands-on Exercise 2: String Processing and UTF-8 Handling**:

```go
// Advanced string processing with UTF-8 support for automation
package main

import (
    "fmt"
    "strings"
    "unicode"
    "unicode/utf8"
)

type TextProcessor struct {
    content string
    stats   map[string]interface{}
}

func NewTextProcessor(content string) *TextProcessor {
    tp := &TextProcessor{
        content: content,
        stats:   make(map[string]interface{}),
    }
    tp.analyzeText()
    return tp
}

func (tp *TextProcessor) analyzeText() {
    tp.stats["byte_length"] = len(tp.content)
    tp.stats["rune_count"] = utf8.RuneCountInString(tp.content)
    tp.stats["is_valid_utf8"] = utf8.ValidString(tp.content)

    // Count different character types
    var letters, digits, spaces, punctuation, other int

    for _, r := range tp.content {
        switch {
        case unicode.IsLetter(r):
            letters++
        case unicode.IsDigit(r):
            digits++
        case unicode.IsSpace(r):
            spaces++
        case unicode.IsPunct(r):
            punctuation++
        default:
            other++
        }
    }

    tp.stats["letters"] = letters
    tp.stats["digits"] = digits
    tp.stats["spaces"] = spaces
    tp.stats["punctuation"] = punctuation
    tp.stats["other"] = other
}

func (tp *TextProcessor) GetStats() map[string]interface{} {
    return tp.stats
}

func (tp *TextProcessor) ExtractWords() []string {
    var words []string

    // Split by whitespace and clean
    fields := strings.Fields(tp.content)
    for _, field := range fields {
        // Remove punctuation from start and end
        word := strings.TrimFunc(field, func(r rune) bool {
            return unicode.IsPunct(r) || unicode.IsSymbol(r)
        })

        if word != "" {
            words = append(words, strings.ToLower(word))
        }
    }

    return words
}

func (tp *TextProcessor) FindEmojis() []string {
    var emojis []string

    for _, r := range tp.content {
        if unicode.Is(unicode.So, r) || // Symbol, other
           unicode.Is(unicode.Sm, r) || // Symbol, math
           (r >= 0x1F600 && r <= 0x1F64F) || // Emoticons
           (r >= 0x1F300 && r <= 0x1F5FF) || // Misc symbols
           (r >= 0x1F680 && r <= 0x1F6FF) || // Transport
           (r >= 0x2600 && r <= 0x26FF) {    // Misc symbols
            emojis = append(emojis, string(r))
        }
    }

    return emojis
}

func (tp *TextProcessor) ConvertToBytes() []byte {
    return []byte(tp.content)
}

func (tp *TextProcessor) SafeSubstring(start, length int) string {
    runes := []rune(tp.content)

    if start < 0 || start >= len(runes) {
        return ""
    }

    end := start + length
    if end > len(runes) {
        end = len(runes)
    }

    return string(runes[start:end])
}

func main() {
    demonstrateStringBasics()
    demonstrateUTF8Processing()
    demonstrateStringConversions()
}

func demonstrateStringBasics() {
    fmt.Println("=== String Basics ===")

    text := "Hello, ä¸–ç•Œ! 🔥Œ Automation Service"
    processor := NewTextProcessor(text)

    fmt.Printf("Original text: %s\n", text)

    stats := processor.GetStats()
    for key, value := range stats {
        fmt.Printf("  %s: %v\n", key, value)
    }

    words := processor.ExtractWords()
    fmt.Printf("Words: %v\n", words)

    emojis := processor.FindEmojis()
    fmt.Printf("Emojis: %v\n", emojis)
}

func demonstrateUTF8Processing() {
    fmt.Println("\n=== UTF-8 Processing ===")

    // Different string representations
    strings := []string{
        "ASCII text",
        "CafÃ© rÃ©sumÃ© naÃ¯ve",
        "æ—¥æœ¬èªž Chinese ä¸­æ–‡",
        "Emoji test 🔥š€🔥”¥🔥’»",
        "Mixed: Helloä¸–ç•Œ🔥Œ",
    }

    for _, s := range strings {
        fmt.Printf("\nString: %s\n", s)
        fmt.Printf("  Bytes: %d\n", len(s))
        fmt.Printf("  Runes: %d\n", utf8.RuneCountInString(s))
        fmt.Printf("  Valid UTF-8: %t\n", utf8.ValidString(s))

        // Show byte vs rune iteration
        fmt.Printf("  Byte iteration: ")
        for i := 0; i < len(s) && i < 10; i++ {
            fmt.Printf("%02x ", s[i])
        }
        fmt.Println()

        fmt.Printf("  Rune iteration: ")
        count := 0
        for _, r := range s {
            if count >= 5 {
                fmt.Printf("...")
                break
            }
            fmt.Printf("U+%04X ", r)
            count++
        }
        fmt.Println()
    }
}

func demonstrateStringConversions() {
    fmt.Println("\n=== String Conversions ===")

    original := "Hello, ä¸–ç•Œ! 🔥Œ"
    fmt.Printf("Original: %s\n", original)

    // String to []byte conversion
    bytes := []byte(original)
    fmt.Printf("As bytes: %v\n", bytes)
    fmt.Printf("Byte length: %d\n", len(bytes))

    // []byte to string conversion
    backToString := string(bytes)
    fmt.Printf("Back to string: %s\n", backToString)
    fmt.Printf("Equal: %t\n", original == backToString)

    // String to []rune conversion
    runes := []rune(original)
    fmt.Printf("As runes: %v\n", runes)
    fmt.Printf("Rune length: %d\n", len(runes))

    // []rune to string conversion
    backFromRunes := string(runes)
    fmt.Printf("Back from runes: %s\n", backFromRunes)

    // Safe substring operations
    processor := NewTextProcessor(original)

    fmt.Println("\nSafe substring operations:")
    fmt.Printf("  First 5 runes: '%s'\n", processor.SafeSubstring(0, 5))
    fmt.Printf("  Runes 7-10: '%s'\n", processor.SafeSubstring(7, 3))
    fmt.Printf("  Last 3 runes: '%s'\n", processor.SafeSubstring(len(runes)-3, 3))

    // Demonstrate string immutability
    fmt.Println("\n=== String Immutability ===")

    s1 := "original"
    s2 := s1 // Shares the same backing array
    s3 := s1 + " modified" // Creates new string

    fmt.Printf("s1: %s\n", s1)
    fmt.Printf("s2: %s\n", s2)
    fmt.Printf("s3: %s\n", s3)
    fmt.Printf("s1 == s2: %t\n", s1 == s2)
    fmt.Printf("s1 == s3: %t\n", s1 == s3)

    // String building efficiency
    fmt.Println("\n=== Efficient String Building ===")

    // Inefficient: string concatenation
    result1 := ""
    for i := 0; i < 5; i++ {
        result1 += fmt.Sprintf("item%d ", i)
    }
    fmt.Printf("Concatenation result: %s\n", result1)

    // Efficient: strings.Builder
    var builder strings.Builder
    for i := 0; i < 5; i++ {
        builder.WriteString(fmt.Sprintf("item%d ", i))
    }
    result2 := builder.String()
    fmt.Printf("Builder result: %s\n", result2)

    // Efficient: strings.Join
    var parts []string
    for i := 0; i < 5; i++ {
        parts = append(parts, fmt.Sprintf("item%d", i))
    }
    result3 := strings.Join(parts, " ") + " "
    fmt.Printf("Join result: %s\n", result3)
}
```

**Hands-on Exercise 3: Map Operations and Performance Patterns**:

```go
// Advanced map operations for automation data management
package main

import (
    "fmt"
    "sort"
    "time"
)

type ServiceRegistry struct {
    services map[string]*ServiceInfo
    stats    map[string]int
}

type ServiceInfo struct {
    Name        string
    URL         string
    Status      string
    LastCheck   time.Time
    ResponseTime time.Duration
    Tags        []string
}

func NewServiceRegistry() *ServiceRegistry {
    return &ServiceRegistry{
        services: make(map[string]*ServiceInfo),
        stats:    make(map[string]int),
    }
}

func (sr *ServiceRegistry) RegisterService(id string, info *ServiceInfo) {
    sr.services[id] = info
    sr.updateStats("registered")
}

func (sr *ServiceRegistry) UpdateServiceStatus(id, status string, responseTime time.Duration) bool {
    service, exists := sr.services[id]
    if !exists {
        sr.updateStats("update_failed")
        return false
    }

    service.Status = status
    service.LastCheck = time.Now()
    service.ResponseTime = responseTime
    sr.updateStats("updated")
    return true
}

func (sr *ServiceRegistry) GetService(id string) (*ServiceInfo, bool) {
    service, exists := sr.services[id]
    sr.updateStats("accessed")
    return service, exists
}

func (sr *ServiceRegistry) GetServicesByStatus(status string) []*ServiceInfo {
    var services []*ServiceInfo

    for _, service := range sr.services {
        if service.Status == status {
            services = append(services, service)
        }
    }

    return services
}

func (sr *ServiceRegistry) GetServicesByTag(tag string) []*ServiceInfo {
    var services []*ServiceInfo

    for _, service := range sr.services {
        for _, serviceTag := range service.Tags {
            if serviceTag == tag {
                services = append(services, service)
                break
            }
        }
    }

    return services
}

func (sr *ServiceRegistry) GetStats() map[string]int {
    // Return a copy to prevent external modification
    statsCopy := make(map[string]int)
    for k, v := range sr.stats {
        statsCopy[k] = v
    }
    return statsCopy
}

func (sr *ServiceRegistry) updateStats(operation string) {
    sr.stats[operation]++
    sr.stats["total_operations"]++
}

func (sr *ServiceRegistry) ListServices() {
    fmt.Printf("Registered Services (%d total):\n", len(sr.services))

    // Sort services by ID for consistent output
    var ids []string
    for id := range sr.services {
        ids = append(ids, id)
    }
    sort.Strings(ids)

    for _, id := range ids {
        service := sr.services[id]
        fmt.Printf("  %s: %s [%s] - %v ago\n",
            id, service.Name, service.Status,
            time.Since(service.LastCheck).Round(time.Second))
    }
}

func main() {
    demonstrateMapBasics()
    demonstrateMapPerformance()
    demonstrateServiceRegistry()
}

func demonstrateMapBasics() {
    fmt.Println("=== Map Basics ===")

    // Different map initialization methods
    var nilMap map[string]int
    emptyMap := map[string]int{}
    madeMap := make(map[string]int)
    literalMap := map[string]int{
        "one":   1,
        "two":   2,
        "three": 3,
    }

    fmt.Printf("nil map: %v (len=%d)\n", nilMap, len(nilMap))
    fmt.Printf("empty map: %v (len=%d)\n", emptyMap, len(emptyMap))
    fmt.Printf("made map: %v (len=%d)\n", madeMap, len(madeMap))
    fmt.Printf("literal map: %v (len=%d)\n", literalMap, len(literalMap))

    // Map operations
    madeMap["key1"] = 100
    madeMap["key2"] = 200

    // Check existence
    if value, exists := madeMap["key1"]; exists {
        fmt.Printf("key1 exists with value: %d\n", value)
    }

    if _, exists := madeMap["key3"]; !exists {
        fmt.Println("key3 does not exist")
    }

    // Delete operation
    delete(madeMap, "key1")
    fmt.Printf("After deletion: %v\n", madeMap)

    // Range over map
    fmt.Println("Iterating over literal map:")
    for key, value := range literalMap {
        fmt.Printf("  %s: %d\n", key, value)
    }
}

func demonstrateMapPerformance() {
    fmt.Println("\n=== Map Performance ===")

    // Create maps with different sizes
    sizes := []int{100, 1000, 10000}

    for _, size := range sizes {
        fmt.Printf("\nTesting with %d elements:\n", size)

        // Test map creation and insertion
        start := time.Now()
        testMap := make(map[int]string, size) // Pre-allocate capacity
        for i := 0; i < size; i++ {
            testMap[i] = fmt.Sprintf("value%d", i)
        }
        insertTime := time.Since(start)

        // Test lookups
        start = time.Now()
        found := 0
        for i := 0; i < size; i++ {
            if _, exists := testMap[i]; exists {
                found++
            }
        }
        lookupTime := time.Since(start)

        // Test iteration
        start = time.Now()
        count := 0
        for range testMap {
            count++
        }
        iterTime := time.Since(start)

        fmt.Printf("  Insert: %v\n", insertTime)
        fmt.Printf("  Lookup: %v (%d found)\n", lookupTime, found)
        fmt.Printf("  Iterate: %v (%d counted)\n", iterTime, count)
    }
}

func demonstrateServiceRegistry() {
    fmt.Println("\n=== Service Registry Demo ===")

    registry := NewServiceRegistry()

    // Register services
    services := []*ServiceInfo{
        {
            Name: "User Service",
            URL:  "http://user-service:8080",
            Status: "healthy",
            Tags: []string{"api", "user", "core"},
        },
        {
            Name: "Payment Service",
            URL:  "http://payment-service:8081",
            Status: "healthy",
            Tags: []string{"api", "payment", "critical"},
        },
        {
            Name: "Notification Service",
            URL:  "http://notification-service:8082",
            Status: "degraded",
            Tags: []string{"api", "notification"},
        },
        {
            Name: "Analytics Service",
            URL:  "http://analytics-service:8083",
            Status: "down",
            Tags: []string{"analytics", "reporting"},
        },
    }

    for i, service := range services {
        id := fmt.Sprintf("service-%d", i+1)
        registry.RegisterService(id, service)
    }

    // Update some service statuses
    registry.UpdateServiceStatus("service-1", "healthy", 50*time.Millisecond)
    registry.UpdateServiceStatus("service-2", "healthy", 120*time.Millisecond)
    registry.UpdateServiceStatus("service-3", "healthy", 200*time.Millisecond)

    // List all services
    registry.ListServices()

    // Query by status
    fmt.Println("\nHealthy services:")
    healthyServices := registry.GetServicesByStatus("healthy")
    for _, service := range healthyServices {
        fmt.Printf("  %s: %v response time\n", service.Name, service.ResponseTime)
    }

    // Query by tag
    fmt.Println("\nAPI services:")
    apiServices := registry.GetServicesByTag("api")
    for _, service := range apiServices {
        fmt.Printf("  %s: %s\n", service.Name, service.Status)
    }

    // Show statistics
    fmt.Println("\nRegistry statistics:")
    stats := registry.GetStats()
    for operation, count := range stats {
        fmt.Printf("  %s: %d\n", operation, count)
    }
}
```

**Prerequisites**: Module 13
