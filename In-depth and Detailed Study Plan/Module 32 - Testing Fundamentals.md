# Module 32: Testing Fundamentals

**Duration**: 1 hour (45 min theory + 15 min hands-on)

**Learning Objectives**:

- Master Go's built-in testing framework and conventions
- Understand unit testing philosophy and "Given, When, Should" structure
- Learn table-driven test patterns for comprehensive scenario coverage
- Apply testing strategies to automation system components
- Design maintainable and readable test suites with consistent output
- Implement basic mocking patterns for external dependencies using httptest

**Videos Covered**:

- 12.1 Topics (0:00:41)
- 12.2 Basic Unit Testing (0:13:54)
- 12.3 Table Unit Testing (0:03:19)
- 12.4 Mocking Web Server Response (0:06:59)

**Key Concepts**:

- Go testing conventions: `_test.go` files, `Test` function naming
- Testing package vs same package testing approaches
- Test structure: Given-When-Should pattern for clarity
- Error handling in tests: `t.Error()`, `t.Fatal()`, `t.Log()`
- Table-driven tests: testing multiple scenarios efficiently
- Test output consistency and verbosity for CI/CD environments
- HTTP mocking with `httptest` package for isolated testing
- Test organization and maintainability best practices

**Hands-on Exercise 1: Basic Unit Testing with Go Testing Framework**:

Understanding Go's testing conventions, test structure, and error handling patterns:

```go
// Basic unit testing demonstration for automation systems
package automation

import (
    "fmt"
    "net/http"
    "strings"
    "time"
)

// AutomationService represents a service in our automation platform
type AutomationService struct {
    name     string
    endpoint string
    timeout  time.Duration
    client   *http.Client
}

// NewAutomationService creates a new automation service
func NewAutomationService(name, endpoint string, timeout time.Duration) *AutomationService {
    return &AutomationService{
        name:     name,
        endpoint: endpoint,
        timeout:  timeout,
        client: &http.Client{
            Timeout: timeout,
        },
    }
}

// ProcessMessage processes a message through the automation service
func (as *AutomationService) ProcessMessage(message string) (string, error) {
    if message == "" {
        return "", fmt.Errorf("message cannot be empty")
    }

    // Simulate message processing
    processed := strings.ToUpper(message)
    processed = fmt.Sprintf("[%s] %s", as.name, processed)

    return processed, nil
}

// ValidateConfig validates service configuration
func (as *AutomationService) ValidateConfig() error {
    if as.name == "" {
        return fmt.Errorf("service name cannot be empty")
    }

    if as.endpoint == "" {
        return fmt.Errorf("service endpoint cannot be empty")
    }

    if as.timeout <= 0 {
        return fmt.Errorf("timeout must be positive")
    }

    return nil
}

// DownloadContent simulates downloading content from a URL
func DownloadContent(url string) (*http.Response, error) {
    if url == "" {
        return nil, fmt.Errorf("URL cannot be empty")
    }

    client := &http.Client{
        Timeout: 30 * time.Second,
    }

    resp, err := client.Get(url)
    if err != nil {
        return nil, fmt.Errorf("failed to download content: %w", err)
    }

    return resp, nil
}
```

**Test File (automation_test.go):**

```go
package automation_test // Using _test package to test exported API only

import (
    "automation" // Import our package like any other user
    "net/http"
    "strings"
    "testing"
    "time"
)

// Constants for consistent test output (Bill Kennedy's approach)
const (
    checkMark = "\u2713" // ✓
    ballotX   = "\u2717" // ✗
)

// PART 1: Basic Unit Tests Following "Given, When, Should" Structure
// These tests demonstrate fundamental Go testing patterns

func TestDownloadContent(t *testing.T) {
    t.Log("Given the need to test downloading content")
    {
        testID := 0
        t.Logf("\tTest %d:\tWhen downloading from a valid URL", testID)
        {
            // Given
            url := "https://www.goinggo.net/index.xml"
            statusCode := http.StatusOK

            // When
            resp, err := automation.DownloadContent(url)

            // Should
            if err != nil {
                t.Fatalf("\t%s\tTest %d:\tShould be able to make the call. Error: %v",
                    ballotX, testID, err)
            }
            t.Logf("\t%s\tTest %d:\tShould be able to make the call", checkMark, testID)

            defer resp.Body.Close()

            if resp.StatusCode != statusCode {
                t.Fatalf("\t%s\tTest %d:\tShould receive a %d status. Received: %d",
                    ballotX, testID, statusCode, resp.StatusCode)
            }
            t.Logf("\t%s\tTest %d:\tShould receive a %d status", checkMark, testID, statusCode)
        }

        testID++
        t.Logf("\tTest %d:\tWhen downloading with empty URL", testID)
        {
            // Given
            url := ""

            // When
            resp, err := automation.DownloadContent(url)

            // Should
            if err == nil {
                t.Fatalf("\t%s\tTest %d:\tShould return error for empty URL", ballotX, testID)
            }
            t.Logf("\t%s\tTest %d:\tShould return error for empty URL", checkMark, testID)

            if resp != nil {
                t.Fatalf("\t%s\tTest %d:\tShould not return response for empty URL", ballotX, testID)
            }
            t.Logf("\t%s\tTest %d:\tShould not return response for empty URL", checkMark, testID)
        }
    }
}

func TestAutomationService_ProcessMessage(t *testing.T) {
    t.Log("Given the need to process messages through automation service")
    {
        service := automation.NewAutomationService("TestService", "http://localhost:8080", 30*time.Second)

        testID := 0
        t.Logf("\tTest %d:\tWhen processing a valid message", testID)
        {
            // Given
            message := "hello world"
            expectedPrefix := "[TestService]"

            // When
            result, err := service.ProcessMessage(message)

            // Should
            if err != nil {
                t.Fatalf("\t%s\tTest %d:\tShould process message without error. Got: %v",
                    ballotX, testID, err)
            }
            t.Logf("\t%s\tTest %d:\tShould process message without error", checkMark, testID)

            if !strings.Contains(result, expectedPrefix) {
                t.Fatalf("\t%s\tTest %d:\tShould include service name prefix. Expected to contain %s, got %s",
                    ballotX, testID, expectedPrefix, result)
            }
            t.Logf("\t%s\tTest %d:\tShould include service name prefix", checkMark, testID)

            if !strings.Contains(result, "HELLO WORLD") {
                t.Fatalf("\t%s\tTest %d:\tShould convert message to uppercase. Expected to contain HELLO WORLD, got %s",
                    ballotX, testID, result)
            }
            t.Logf("\t%s\tTest %d:\tShould convert message to uppercase", checkMark, testID)
        }

        testID++
        t.Logf("\tTest %d:\tWhen processing an empty message", testID)
        {
            // Given
            message := ""

            // When
            result, err := service.ProcessMessage(message)

            // Should
            if err == nil {
                t.Fatalf("\t%s\tTest %d:\tShould return error for empty message. Got result: %s",
                    ballotX, testID, result)
            }
            t.Logf("\t%s\tTest %d:\tShould return error for empty message", checkMark, testID)

            if !strings.Contains(err.Error(), "cannot be empty") {
                t.Fatalf("\t%s\tTest %d:\tShould return descriptive error. Expected 'cannot be empty', got: %v",
                    ballotX, testID, err)
            }
            t.Logf("\t%s\tTest %d:\tShould return descriptive error", checkMark, testID)
        }
    }
}

func TestAutomationService_ValidateConfig(t *testing.T) {
    t.Log("Given the need to validate automation service configuration")
    {
        testID := 0
        t.Logf("\tTest %d:\tWhen validating a properly configured service", testID)
        {
            // Given
            service := automation.NewAutomationService("ValidService", "http://localhost:8080", 30*time.Second)

            // When
            err := service.ValidateConfig()

            // Should
            if err != nil {
                t.Fatalf("\t%s\tTest %d:\tShould validate without error. Got: %v",
                    ballotX, testID, err)
            }
            t.Logf("\t%s\tTest %d:\tShould validate without error", checkMark, testID)
        }

        testID++
        t.Logf("\tTest %d:\tWhen validating service with empty name", testID)
        {
            // Given
            service := automation.NewAutomationService("", "http://localhost:8080", 30*time.Second)

            // When
            err := service.ValidateConfig()

            // Should
            if err == nil {
                t.Fatalf("\t%s\tTest %d:\tShould return error for empty name", ballotX, testID)
            }
            t.Logf("\t%s\tTest %d:\tShould return error for empty name", checkMark, testID)

            if !strings.Contains(err.Error(), "name cannot be empty") {
                t.Fatalf("\t%s\tTest %d:\tShould return descriptive error. Expected 'name cannot be empty', got: %v",
                    ballotX, testID, err)
            }
            t.Logf("\t%s\tTest %d:\tShould return descriptive error", checkMark, testID)
        }

        testID++
        t.Logf("\tTest %d:\tWhen validating service with empty endpoint", testID)
        {
            // Given
            service := automation.NewAutomationService("TestService", "", 30*time.Second)

            // When
            err := service.ValidateConfig()

            // Should
            if err == nil {
                t.Fatalf("\t%s\tTest %d:\tShould return error for empty endpoint", ballotX, testID)
            }
            t.Logf("\t%s\tTest %d:\tShould return error for empty endpoint", checkMark, testID)
        }

        testID++
        t.Logf("\tTest %d:\tWhen validating service with invalid timeout", testID)
        {
            // Given
            service := automation.NewAutomationService("TestService", "http://localhost:8080", 0)

            // When
            err := service.ValidateConfig()

            // Should
            if err == nil {
                t.Fatalf("\t%s\tTest %d:\tShould return error for invalid timeout", ballotX, testID)
            }
            t.Logf("\t%s\tTest %d:\tShould return error for invalid timeout", checkMark, testID)

            if !strings.Contains(err.Error(), "timeout must be positive") {
                t.Fatalf("\t%s\tTest %d:\tShould return descriptive error. Expected 'timeout must be positive', got: %v",
                    ballotX, testID, err)
            }
            t.Logf("\t%s\tTest %d:\tShould return descriptive error", checkMark, testID)
        }
    }
}
```

**Hands-on Exercise 2: Table-Driven Tests for Comprehensive Scenario Coverage**:

Implementing table-driven test patterns to efficiently test multiple scenarios:

```go
// Table-driven tests for download functionality
package automation_test

import (
    "automation"
    "net/http"
    "strings"
    "testing"
)

// PART 2: Table-Driven Tests
// Table-driven tests allow testing multiple scenarios efficiently with consistent structure

func TestDownloadContent_TableDriven(t *testing.T) {
    t.Log("Given the need to test download functionality with various inputs")
    {
        tests := []struct {
            name           string
            url            string
            expectedStatus int
        }{
            {
                name:           "valid RSS feed",
                url:            "https://www.goinggo.net/index.xml",
                expectedStatus: http.StatusOK,
            },
            {
                name:           "invalid URL with typo",
                url:            "https://www.cnn.com/services/rss/cnn_topstoriess.rss",
                expectedStatus: http.StatusNotFound,
            },
        }

        for testID, tt := range tests {
            t.Run(tt.name, func(t *testing.T) {
                t.Logf("\tTest %d:\tWhen downloading from URL: %s", testID, tt.url)
                {
                    // When
                    resp, err := automation.DownloadContent(tt.url)

                    // Should
                    if err != nil {
                        t.Fatalf("\t%s\tTest %d:\tShould be able to make the call. Error: %v",
                            ballotX, testID, err)
                    }
                    t.Logf("\t%s\tTest %d:\tShould be able to make the call", checkMark, testID)

                    defer resp.Body.Close()

                    if resp.StatusCode != tt.expectedStatus {
                        t.Fatalf("\t%s\tTest %d:\tShould receive a %d status. Received: %d",
                            ballotX, testID, tt.expectedStatus, resp.StatusCode)
                    }
                    t.Logf("\t%s\tTest %d:\tShould receive a %d status",
                        checkMark, testID, tt.expectedStatus)
                }
            })
        }
    }
}

func TestAutomationService_ProcessMessage_TableDriven(t *testing.T) {
    t.Log("Given the need to test message processing with various inputs")
    {
        service := automation.NewAutomationService("TableTestService", "http://localhost:8080", 30*time.Second)

        tests := []struct {
            name           string
            input          string
            expectError    bool
            expectedPrefix string
            expectedUpper  bool
            errorContains  string
        }{
            {
                name:           "valid simple message",
                input:          "hello",
                expectError:    false,
                expectedPrefix: "[TableTestService]",
                expectedUpper:  true,
            },
            {
                name:           "valid message with spaces",
                input:          "hello world",
                expectError:    false,
                expectedPrefix: "[TableTestService]",
                expectedUpper:  true,
            },
            {
                name:           "valid message with special characters",
                input:          "hello@world.com!",
                expectError:    false,
                expectedPrefix: "[TableTestService]",
                expectedUpper:  true,
            },
            {
                name:          "empty message should error",
                input:         "",
                expectError:   true,
                errorContains: "cannot be empty",
            },
            {
                name:           "single character message",
                input:          "a",
                expectError:    false,
                expectedPrefix: "[TableTestService]",
                expectedUpper:  true,
            },
        }

        for testID, tt := range tests {
            t.Run(tt.name, func(t *testing.T) {
                t.Logf("\tTest %d:\tWhen processing message: %q", testID, tt.input)
                {
                    // When
                    result, err := service.ProcessMessage(tt.input)

                    // Should
                    if tt.expectError {
                        if err == nil {
                            t.Fatalf("\t%s\tTest %d:\tShould return error for input %q",
                                ballotX, testID, tt.input)
                        }
                        t.Logf("\t%s\tTest %d:\tShould return error for input %q",
                            checkMark, testID, tt.input)

                        if tt.errorContains != "" && !strings.Contains(err.Error(), tt.errorContains) {
                            t.Fatalf("\t%s\tTest %d:\tShould contain error text %q. Got: %v",
                                ballotX, testID, tt.errorContains, err)
                        }
                        t.Logf("\t%s\tTest %d:\tShould contain expected error text",
                            checkMark, testID)
                        return
                    }

                    if err != nil {
                        t.Fatalf("\t%s\tTest %d:\tShould not return error for input %q. Got: %v",
                            ballotX, testID, tt.input, err)
                    }
                    t.Logf("\t%s\tTest %d:\tShould not return error for input %q",
                        checkMark, testID, tt.input)

                    if tt.expectedPrefix != "" && !strings.Contains(result, tt.expectedPrefix) {
                        t.Fatalf("\t%s\tTest %d:\tShould contain prefix %q. Got: %s",
                            ballotX, testID, tt.expectedPrefix, result)
                    }
                    t.Logf("\t%s\tTest %d:\tShould contain expected prefix",
                        checkMark, testID)

                    if tt.expectedUpper {
                        upperInput := strings.ToUpper(tt.input)
                        if !strings.Contains(result, upperInput) {
                            t.Fatalf("\t%s\tTest %d:\tShould contain uppercase input %q. Got: %s",
                                ballotX, testID, upperInput, result)
                        }
                        t.Logf("\t%s\tTest %d:\tShould contain uppercase input",
                            checkMark, testID)
                    }
                }
            })
        }
    }
}
```

**Hands-on Exercise 3: Basic Mocking Patterns for External Dependencies**:

Implementing basic mocking techniques using httptest to test HTTP-dependent components:

```go
// Basic mocking patterns for testing HTTP clients and external services
package automation_test

import (
    "automation"
    "encoding/xml"
    "fmt"
    "io"
    "net/http"
    "net/http/httptest"
    "strings"
    "testing"
)

// PART 3: Basic Mocking Patterns using httptest
// Mocking allows us to test components that depend on external services

// RSS feed structures for testing XML unmarshaling
type Document struct {
    XMLName xml.Name `xml:"rss"`
    Channel Channel  `xml:"channel"`
}

type Channel struct {
    Title string `xml:"title"`
    Items []Item `xml:"item"`
}

type Item struct {
    Title string `xml:"title"`
    Link  string `xml:"link"`
}

// Mock RSS feed data - simulates what we expect from goinggo.net
const feed = `<?xml version="1.0" encoding="UTF-8"?>
<rss version="2.0">
    <channel>
        <title>Going Go Programming</title>
        <item>
            <title>Understanding Go Testing</title>
            <link>https://www.goinggo.net/2017/05/understanding-go-testing.html</link>
        </item>
        <item>
            <title>Table Driven Tests in Go</title>
            <link>https://www.goinggo.net/2017/05/table-driven-tests-in-go.html</link>
        </item>
    </channel>
</rss>`

// mockServer creates an HTTP test server that returns our mock RSS feed
func mockServer() *httptest.Server {
    f := func(w http.ResponseWriter, r *http.Request) {
        w.WriteHeader(http.StatusOK)
        w.Header().Set("Content-Type", "application/xml")
        fmt.Fprintln(w, feed)
    }

    return httptest.NewServer(http.HandlerFunc(f))
}

// Test using httptest.Server for mocking HTTP responses
func TestDownloadContent_MockServer(t *testing.T) {
    t.Log("Given the need to test download functionality with mocked server responses")
    {
        testID := 0
        t.Logf("\tTest %d:\tWhen downloading from mocked RSS feed", testID)
        {
            // Given - Create mock server using our mockServer function
            server := mockServer()
            defer server.Close()

            statusCode := http.StatusOK

            // When
            resp, err := automation.DownloadContent(server.URL)

            // Should
            if err != nil {
                t.Fatalf("\t%s\tTest %d:\tShould be able to make the call. Error: %v",
                    ballotX, testID, err)
            }
            t.Logf("\t%s\tTest %d:\tShould be able to make the call", checkMark, testID)

            defer resp.Body.Close()

            if resp.StatusCode != statusCode {
                t.Fatalf("\t%s\tTest %d:\tShould receive a %d status. Received: %d",
                    ballotX, testID, statusCode, resp.StatusCode)
            }
            t.Logf("\t%s\tTest %d:\tShould receive a %d status", checkMark, testID, statusCode)

            // Test that we can unmarshal the response
            body, err := io.ReadAll(resp.Body)
            if err != nil {
                t.Fatalf("\t%s\tTest %d:\tShould be able to read response body. Error: %v",
                    ballotX, testID, err)
            }
            t.Logf("\t%s\tTest %d:\tShould be able to read response body", checkMark, testID)

            var doc Document
            if err := xml.Unmarshal(body, &doc); err != nil {
                t.Fatalf("\t%s\tTest %d:\tShould be able to unmarshal XML. Error: %v",
                    ballotX, testID, err)
            }
            t.Logf("\t%s\tTest %d:\tShould be able to unmarshal XML", checkMark, testID)

            if len(doc.Channel.Items) != 2 {
                t.Fatalf("\t%s\tTest %d:\tShould have 2 items in feed. Got: %d",
                    ballotX, testID, len(doc.Channel.Items))
            }
            t.Logf("\t%s\tTest %d:\tShould have 2 items in feed", checkMark, testID)

            if doc.Channel.Title != "Going Go Programming" {
                t.Fatalf("\t%s\tTest %d:\tShould have correct channel title. Got: %s",
                    ballotX, testID, doc.Channel.Title)
            }
            t.Logf("\t%s\tTest %d:\tShould have correct channel title", checkMark, testID)
        }
    }
}
```

**Key Testing Commands**:

- `go test` - Run tests in current package
- `go test -v` - Run tests with verbose output
- `go test -run TestName` - Run specific test using regex pattern
- `go test ./...` - Run tests in all packages recursively

**Testing Best Practices**:

1. **File Naming**: Use `_test.go` suffix for test files
2. **Function Naming**: Test functions must start with `Test` followed by a capital letter
3. **Package Naming**: Use `package_test` for testing exported APIs only
4. **Test Structure**: Follow "Given, When, Should" pattern for clarity
5. **Error Handling**: Always check errors in tests, never use blank identifier
6. **Consistent Output**: Use consistent formatting for test output and logging
7. **Table-Driven Tests**: Use for testing multiple scenarios efficiently
8. **Mocking**: Use `httptest` package for HTTP mocking instead of external dependencies

**Prerequisites**: Module 31 (Advanced Channel Patterns and Context)
