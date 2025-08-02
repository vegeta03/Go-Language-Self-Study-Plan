# Module 2: Variables and Type System

**Duration**: 1 hour (45 min theory + 15 min hands-on)

**Learning Objectives**:

- Master the fundamental principle that "Type is Life" in Go programming
- Understand how type provides size and representation information
- Learn Go's built-in types and their memory characteristics
- Master zero values as an integrity feature
- Understand variable declaration patterns and when to use each
- Grasp the relationship between type, memory, and performance
- Learn Go's conversion vs casting philosophy

**Videos Covered**:

- 2.1 Topics (0:00:48)
- 2.2 Variables (0:16:26)

**Key Concepts**:

**Type is Everything - Type is Life**:

- Type provides two critical pieces of information: size (memory footprint) and representation (what the data means)
- Without type information, you cannot determine the value of a bit pattern
- The same bit pattern can represent infinite different values depending on type
- Understanding type is essential for understanding cost and performance

**Built-in Types and Memory Layout**:

- Numeric types: int8, int16, int32, int64, uint8, uint16, uint32, uint64
- Architecture-dependent types: int, uint, uintptr (size depends on platform)
- On AMD64: int, uint, uintptr are 64-bit (8 bytes)
- On 32-bit systems: int, uint, uintptr are 32-bit (4 bytes)
- Floating-point: float32 (4 bytes), float64 (8 bytes)
- Boolean: bool (1 byte)
- String: 2-word data structure (8-16 bytes depending on architecture)

**Zero Values and Integrity**:

- All allocated memory is initialized to at least its zero value state
- Zero value is an integrity feature that prevents corruption
- Cost of integrity is minimal performance overhead
- Zero values: 0 for numerics, false for bool, "" for string, nil for pointers

**Variable Declaration Patterns**:

- Use `var` when initializing to zero value (readability marker)
- Use `:=` (short variable declaration) when initializing to non-zero value
- `var` guarantees zero value 100% of the time
- Consistency in declaration patterns improves code readability

**String Internal Structure**:

- String is a 2-word data structure: pointer + length
- First word: pointer to backing array of bytes
- Second word: number of bytes
- Empty string: nil pointer, zero length
- String size varies by architecture (8 bytes on 32-bit, 16 bytes on 64-bit)

**Conversion vs Casting**:

- Go has conversion, not casting
- Conversion may incur memory allocation cost but maintains integrity
- Casting (available via unsafe package) can corrupt memory
- Conversion over casting is an integrity play

**Hands-on Exercise 1: Type Analysis and Memory Layout**:

```go
// Type system exploration for automation systems
package main

import (
    "fmt"
    "unsafe"
)

func main() {
    // Demonstrate the principle "Type is Life"
    demonstrateTypeImportance()

    // Explore built-in types and their sizes
    analyzeBuiltinTypes()

    // Practice variable declaration patterns
    practiceDeclarationPatterns()

    // Understand string internal structure
    exploreStringStructure()
}

func demonstrateTypeImportance() {
    fmt.Println("=== Type is Life Demonstration ===")

    // Same bit pattern, different interpretations
    var value uint32 = 1065353216
    fmt.Printf("As uint32: %d\n", value)
    fmt.Printf("As float32: %f\n", *(*float32)(unsafe.Pointer(&value)))
    fmt.Printf("As bytes: %v\n", *(*[4]byte)(unsafe.Pointer(&value)))
    fmt.Println()
}

func analyzeBuiltinTypes() {
    fmt.Println("=== Built-in Types Analysis ===")

    // Numeric types
    var i8 int8
    var i16 int16
    var i32 int32
    var i64 int64
    var ui8 uint8
    var ui16 uint16
    var ui32 uint32
    var ui64 uint64

    // Architecture-dependent types
    var i int
    var ui uint
    var uptr uintptr

    // Floating-point types
    var f32 float32
    var f64 float64

    // Boolean and string
    var b bool
    var s string

    fmt.Printf("int8 size: %d bytes, value: %d\n", unsafe.Sizeof(i8), i8)
    fmt.Printf("int16 size: %d bytes, value: %d\n", unsafe.Sizeof(i16), i16)
    fmt.Printf("int32 size: %d bytes, value: %d\n", unsafe.Sizeof(i32), i32)
    fmt.Printf("int64 size: %d bytes, value: %d\n", unsafe.Sizeof(i64), i64)
    fmt.Printf("uint8 size: %d bytes, value: %d\n", unsafe.Sizeof(ui8), ui8)
    fmt.Printf("uint16 size: %d bytes, value: %d\n", unsafe.Sizeof(ui16), ui16)
    fmt.Printf("uint32 size: %d bytes, value: %d\n", unsafe.Sizeof(ui32), ui32)
    fmt.Printf("uint64 size: %d bytes, value: %d\n", unsafe.Sizeof(ui64), ui64)

    fmt.Printf("int size: %d bytes, value: %d (architecture-dependent)\n", unsafe.Sizeof(i), i)
    fmt.Printf("uint size: %d bytes, value: %d (architecture-dependent)\n", unsafe.Sizeof(ui), ui)
    fmt.Printf("uintptr size: %d bytes, value: %d (architecture-dependent)\n", unsafe.Sizeof(uptr), uptr)

    fmt.Printf("float32 size: %d bytes, value: %f\n", unsafe.Sizeof(f32), f32)
    fmt.Printf("float64 size: %d bytes, value: %f\n", unsafe.Sizeof(f64), f64)

    fmt.Printf("bool size: %d bytes, value: %t\n", unsafe.Sizeof(b), b)
    fmt.Printf("string size: %d bytes, value: %q\n", unsafe.Sizeof(s), s)
    fmt.Println()
}

func practiceDeclarationPatterns() {
    fmt.Println("=== Variable Declaration Patterns ===")

    // Use var for zero value initialization
    var serverPort int        // zero value: 0
    var databaseURL string    // zero value: ""
    var enableLogging bool    // zero value: false

    fmt.Printf("Zero values - Port: %d, URL: %q, Logging: %t\n",
        serverPort, databaseURL, enableLogging)

    // Use := for non-zero value initialization
    activePort := 8080
    activeURL := "localhost:5432"
    activeLogging := true

    fmt.Printf("Active values - Port: %d, URL: %q, Logging: %t\n",
        activePort, activeURL, activeLogging)

    // Demonstrate conversion
    var precisePort int32 = 8080
    var genericPort int = int(precisePort) // explicit conversion required

    fmt.Printf("Converted port from int32 to int: %d\n", genericPort)
    fmt.Println()
}

func exploreStringStructure() {
    fmt.Println("=== String Internal Structure ===")

    var emptyString string
    helloString := "Hello, Automation!"

    fmt.Printf("Empty string size: %d bytes\n", unsafe.Sizeof(emptyString))
    fmt.Printf("Hello string size: %d bytes (same as empty - it's the header)\n", unsafe.Sizeof(helloString))
    fmt.Printf("Hello string length: %d bytes\n", len(helloString))
    fmt.Printf("Hello string content: %q\n", helloString)

    // Demonstrate string is 2-word structure
    type stringHeader struct {
        data uintptr
        len  int
    }

    header := (*stringHeader)(unsafe.Pointer(&helloString))
    fmt.Printf("String header - Data pointer: %x, Length: %d\n", header.data, header.len)
}
```

**Hands-on Exercise 2: Automation Configuration System**:

```go
// Configuration management system demonstrating type system
package main

import (
    "fmt"
    "time"
)

// Configuration types for automation system
type DatabaseConfig struct {
    Host     string
    Port     int
    Username string
    Password string
    Timeout  time.Duration
}

type ServerConfig struct {
    ListenPort    int
    MaxConnections int
    EnableTLS     bool
    CertPath      string
}

type AutomationConfig struct {
    Database DatabaseConfig
    Server   ServerConfig
    LogLevel string
    Debug    bool
}

func main() {
    // Demonstrate zero value initialization
    var defaultConfig AutomationConfig
    fmt.Println("=== Default Configuration (Zero Values) ===")
    displayConfig("Default", defaultConfig)

    // Demonstrate non-zero value initialization
    productionConfig := AutomationConfig{
        Database: DatabaseConfig{
            Host:     "prod-db.company.com",
            Port:     5432,
            Username: "automation_user",
            Password: "secure_password",
            Timeout:  30 * time.Second,
        },
        Server: ServerConfig{
            ListenPort:     8080,
            MaxConnections: 1000,
            EnableTLS:      true,
            CertPath:       "/etc/ssl/certs/automation.pem",
        },
        LogLevel: "INFO",
        Debug:    false,
    }

    fmt.Println("\n=== Production Configuration ===")
    displayConfig("Production", productionConfig)

    // Demonstrate type conversion in configuration
    demonstrateTypeConversion()
}

func displayConfig(name string, config AutomationConfig) {
    fmt.Printf("%s Config:\n", name)
    fmt.Printf("  Database: %s:%d (timeout: %v)\n",
        config.Database.Host, config.Database.Port, config.Database.Timeout)
    fmt.Printf("  Server: port %d, max connections %d, TLS: %t\n",
        config.Server.ListenPort, config.Server.MaxConnections, config.Server.EnableTLS)
    fmt.Printf("  Logging: level %s, debug: %t\n", config.LogLevel, config.Debug)
}

func demonstrateTypeConversion() {
    fmt.Println("\n=== Type Conversion in Configuration ===")

    // Configuration values might come as different types
    var portString = "8080"
    var timeoutString = "30"

    // Convert string to int (would need strconv in real code)
    // This is a simplified example
    var port int = 8080  // In reality: strconv.Atoi(portString)
    var timeout int = 30 // In reality: strconv.Atoi(timeoutString)

    // Convert to appropriate types for configuration
    var serverPort int = port
    var timeoutDuration time.Duration = time.Duration(timeout) * time.Second

    fmt.Printf("Converted port: %d (from string %q)\n", serverPort, portString)
    fmt.Printf("Converted timeout: %v (from string %q)\n", timeoutDuration, timeoutString)
}
```

**Hands-on Exercise 3: Memory Cost Analysis**:

```go
// Memory cost analysis for automation data structures
package main

import (
    "fmt"
    "unsafe"
)

type SmallConfig struct {
    Enabled bool
    Port    int16
}

type LargeConfig struct {
    Enabled     bool
    Port        int64
    Description string
    Tags        []string
    Metadata    map[string]interface{}
}

func main() {
    fmt.Println("=== Memory Cost Analysis ===")

    // Analyze small vs large structures
    var small SmallConfig
    var large LargeConfig

    fmt.Printf("SmallConfig size: %d bytes\n", unsafe.Sizeof(small))
    fmt.Printf("LargeConfig size: %d bytes\n", unsafe.Sizeof(large))

    // Demonstrate cost of different types
    analyzeCostOfTypes()

    // Show impact of type choices on memory usage
    demonstrateTypeChoiceImpact()
}

func analyzeCostOfTypes() {
    fmt.Println("\n=== Cost of Different Types ===")

    var b bool
    var i8 int8
    var i16 int16
    var i32 int32
    var i64 int64
    var f32 float32
    var f64 float64
    var s string
    var slice []int
    var m map[string]int

    fmt.Printf("bool: %d bytes\n", unsafe.Sizeof(b))
    fmt.Printf("int8: %d bytes\n", unsafe.Sizeof(i8))
    fmt.Printf("int16: %d bytes\n", unsafe.Sizeof(i16))
    fmt.Printf("int32: %d bytes\n", unsafe.Sizeof(i32))
    fmt.Printf("int64: %d bytes\n", unsafe.Sizeof(i64))
    fmt.Printf("float32: %d bytes\n", unsafe.Sizeof(f32))
    fmt.Printf("float64: %d bytes\n", unsafe.Sizeof(f64))
    fmt.Printf("string: %d bytes (header only)\n", unsafe.Sizeof(s))
    fmt.Printf("slice: %d bytes (header only)\n", unsafe.Sizeof(slice))
    fmt.Printf("map: %d bytes (header only)\n", unsafe.Sizeof(m))
}

func demonstrateTypeChoiceImpact() {
    fmt.Println("\n=== Impact of Type Choices ===")

    // Scenario: storing 1000 port numbers
    const count = 1000

    // Using int64 (8 bytes each)
    ports64 := make([]int64, count)
    fmt.Printf("1000 ports as int64: %d bytes\n", unsafe.Sizeof(ports64[0]) * count)

    // Using int16 (2 bytes each) - sufficient for port numbers
    ports16 := make([]int16, count)
    fmt.Printf("1000 ports as int16: %d bytes\n", unsafe.Sizeof(ports16[0]) * count)

    savings := (unsafe.Sizeof(ports64[0]) - unsafe.Sizeof(ports16[0])) * count
    fmt.Printf("Memory savings: %d bytes (%.1f%% reduction)\n",
        savings, float64(savings)/float64(unsafe.Sizeof(ports64[0]) * count) * 100)
}
```

**Prerequisites**: Module 1
