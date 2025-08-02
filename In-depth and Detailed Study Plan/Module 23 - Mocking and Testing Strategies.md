# Module 23: Mocking and Testing Strategies

**Duration**: 1 hour (45 min theory + 15 min hands-on)

**Learning Objectives**:

- Understand mocking through interfaces in Go
- Learn dependency injection patterns for testability
- Master test doubles and mock implementations
- Apply testing strategies to automation systems
- Design testable automation components

**Videos Covered**:

- 5.8 Mocking (0:05:53)

**Key Concepts**:

- Mocking through interfaces: dependency injection
- Test doubles: mocks, stubs, fakes
- Testable design: interfaces for external dependencies
- Dependency injection patterns in Go
- Testing automation workflows and integrations

**Hands-on Exercise 1: Dependency Injection and Testable Design**:

```go
// Demonstrating dependency injection patterns for testable automation systems
package main

import (
    "fmt"
    "time"
)

// STEP 1: Identify external dependencies that need to be testable

// External dependencies as interfaces - these are the seams for testing
type MetricsCollector interface {
    RecordMetric(name string, value float64, tags map[string]string) error
    GetMetrics(filter string) ([]Metric, error)
}

type NotificationService interface {
    SendNotification(recipient, message string) error
    SendAlert(level string, message string) error
}

type ConfigurationStore interface {
    GetConfig(key string) (string, error)
    SetConfig(key, value string) error
}

// Data structures
type Metric struct {
    Name      string
    Value     float64
    Tags      map[string]string
    Timestamp time.Time
}

// STEP 2: Create production implementations

type PrometheusCollector struct {
    endpoint string
    apiKey   string
}

func NewPrometheusCollector(endpoint, apiKey string) *PrometheusCollector {
    return &PrometheusCollector{
        endpoint: endpoint,
        apiKey:   apiKey,
    }
}

func (pc *PrometheusCollector) RecordMetric(name string, value float64, tags map[string]string) error {
    fmt.Printf("PROMETHEUS: Recording metric %s = %.2f with tags %v\n", name, value, tags)
    // In real implementation: HTTP POST to Prometheus pushgateway
    time.Sleep(10 * time.Millisecond) // Simulate network call
    return nil
}

func (pc *PrometheusCollector) GetMetrics(filter string) ([]Metric, error) {
    fmt.Printf("PROMETHEUS: Querying metrics with filter: %s\n", filter)
    // In real implementation: HTTP GET from Prometheus API
    time.Sleep(20 * time.Millisecond) // Simulate network call

    return []Metric{
        {Name: "cpu_usage", Value: 75.5, Tags: map[string]string{"host": "server1"}, Timestamp: time.Now()},
        {Name: "memory_usage", Value: 82.3, Tags: map[string]string{"host": "server1"}, Timestamp: time.Now()},
    }, nil
}

type SlackNotificationService struct {
    webhookURL string
    channel    string
}

func NewSlackNotificationService(webhookURL, channel string) *SlackNotificationService {
    return &SlackNotificationService{
        webhookURL: webhookURL,
        channel:    channel,
    }
}

func (sns *SlackNotificationService) SendNotification(recipient, message string) error {
    fmt.Printf("SLACK: Sending notification to %s in #%s: %s\n", recipient, sns.channel, message)
    // In real implementation: HTTP POST to Slack webhook
    time.Sleep(15 * time.Millisecond) // Simulate network call
    return nil
}

func (sns *SlackNotificationService) SendAlert(level, message string) error {
    emoji := "ℹ¹ï¸"
    if level == "error" {
        emoji = "🔥š¨"
    } else if level == "warning" {
        emoji = "⚙️ ï¸"
    }

    fmt.Printf("SLACK: Sending %s alert in #%s: %s %s\n", level, sns.channel, emoji, message)
    time.Sleep(15 * time.Millisecond) // Simulate network call
    return nil
}

type EtcdConfigStore struct {
    endpoints []string
    keyPrefix string
}

func NewEtcdConfigStore(endpoints []string, keyPrefix string) *EtcdConfigStore {
    return &EtcdConfigStore{
        endpoints: endpoints,
        keyPrefix: keyPrefix,
    }
}

func (ecs *EtcdConfigStore) GetConfig(key string) (string, error) {
    fullKey := ecs.keyPrefix + "/" + key
    fmt.Printf("ETCD: Getting config %s from %v\n", fullKey, ecs.endpoints)
    // In real implementation: etcd client Get operation
    time.Sleep(5 * time.Millisecond) // Simulate network call

    // Simulate some config values
    configs := map[string]string{
        "processing_interval": "30s",
        "max_retries":        "3",
        "alert_threshold":    "90.0",
    }

    if value, exists := configs[key]; exists {
        return value, nil
    }
    return "", fmt.Errorf("config key %s not found", key)
}

func (ecs *EtcdConfigStore) SetConfig(key, value string) error {
    fullKey := ecs.keyPrefix + "/" + key
    fmt.Printf("ETCD: Setting config %s = %s in %v\n", fullKey, value, ecs.endpoints)
    // In real implementation: etcd client Put operation
    time.Sleep(5 * time.Millisecond) // Simulate network call
    return nil
}

// STEP 3: Create automation service with dependency injection

type AutomationMonitor struct {
    metrics      MetricsCollector
    notifications NotificationService
    config       ConfigurationStore
    name         string
}

// Constructor uses dependency injection - accepts interfaces, not concrete types
func NewAutomationMonitor(
    metrics MetricsCollector,
    notifications NotificationService,
    config ConfigurationStore,
    name string,
) *AutomationMonitor {
    return &AutomationMonitor{
        metrics:       metrics,
        notifications: notifications,
        config:        config,
        name:          name,
    }
}

func (am *AutomationMonitor) MonitorSystem() error {
    fmt.Printf("=== %s: Starting system monitoring ===\n", am.name)

    // Get configuration
    thresholdStr, err := am.config.GetConfig("alert_threshold")
    if err != nil {
        return fmt.Errorf("failed to get alert threshold: %w", err)
    }

    intervalStr, err := am.config.GetConfig("processing_interval")
    if err != nil {
        return fmt.Errorf("failed to get processing interval: %w", err)
    }

    fmt.Printf("Using threshold: %s, interval: %s\n", thresholdStr, intervalStr)

    // Collect current metrics
    currentMetrics, err := am.metrics.GetMetrics("host=server1")
    if err != nil {
        return fmt.Errorf("failed to get metrics: %w", err)
    }

    // Process metrics and check thresholds
    for _, metric := range currentMetrics {
        fmt.Printf("Processing metric: %s = %.2f\n", metric.Name, metric.Value)

        // Record processing metric
        processingTags := map[string]string{
            "monitor": am.name,
            "metric":  metric.Name,
        }

        if err := am.metrics.RecordMetric("metrics_processed", 1, processingTags); err != nil {
            fmt.Printf("Warning: failed to record processing metric: %v\n", err)
        }

        // Check if alert is needed (simplified threshold check)
        if metric.Value > 80.0 {
            alertMsg := fmt.Sprintf("High %s detected: %.2f%% on %s",
                metric.Name, metric.Value, metric.Tags["host"])

            if err := am.notifications.SendAlert("warning", alertMsg); err != nil {
                fmt.Printf("Warning: failed to send alert: %v\n", err)
            }
        }
    }

    // Send completion notification
    completionMsg := fmt.Sprintf("Monitor %s completed processing %d metrics",
        am.name, len(currentMetrics))

    if err := am.notifications.SendNotification("ops-team", completionMsg); err != nil {
        fmt.Printf("Warning: failed to send completion notification: %v\n", err)
    }

    fmt.Printf("=== %s: Monitoring cycle completed ===\n", am.name)
    return nil
}

func (am *AutomationMonitor) UpdateConfiguration(key, value string) error {
    fmt.Printf("%s: Updating configuration %s = %s\n", am.name, key, value)

    if err := am.config.SetConfig(key, value); err != nil {
        return fmt.Errorf("failed to update config: %w", err)
    }

    // Notify about configuration change
    notifyMsg := fmt.Sprintf("Configuration updated: %s = %s", key, value)
    if err := am.notifications.SendNotification("admin", notifyMsg); err != nil {
        fmt.Printf("Warning: failed to notify about config change: %v\n", err)
    }

    return nil
}

func main() {
    fmt.Println("=== Dependency Injection and Testable Design ===")

    // STEP 4: Wire up production dependencies
    fmt.Println("\n=== Production Setup ===")

    // Create production implementations
    metricsCollector := NewPrometheusCollector("http://prometheus:9090", "api-key-123")
    notificationService := NewSlackNotificationService("https://hooks.slack.com/webhook", "monitoring")
    configStore := NewEtcdConfigStore([]string{"etcd1:2379", "etcd2:2379"}, "/automation/config")

    // Inject dependencies into automation monitor
    monitor := NewAutomationMonitor(
        metricsCollector,
        notificationService,
        configStore,
        "ProductionMonitor",
    )

    // Run monitoring cycle
    if err := monitor.MonitorSystem(); err != nil {
        fmt.Printf("Monitoring failed: %v\n", err)
    }

    // Update configuration
    if err := monitor.UpdateConfiguration("alert_threshold", "85.0"); err != nil {
        fmt.Printf("Config update failed: %v\n", err)
    }

    fmt.Println("\n=== Key Design Principles ===")
    fmt.Println("1. Dependencies are injected as interfaces, not concrete types")
    fmt.Println("2. Constructor accepts all external dependencies")
    fmt.Println("3. Business logic doesn't know about specific implementations")
    fmt.Println("4. Easy to swap implementations for different environments")
    fmt.Println("5. Clear separation between business logic and external concerns")
    fmt.Println("6. Testable design without tight coupling to external systems")
}
```

**Hands-on Exercise 2: Mock Implementations and Test Doubles**:

```go
// Demonstrating different types of test doubles for automation testing
package main

import (
    "fmt"
    "strings"
    "testing"
    "time"
)

// External dependencies that need test doubles
type DatabaseClient interface {
    Query(sql string) ([]map[string]interface{}, error)
    Execute(sql string) error
    BeginTransaction() (Transaction, error)
}

type Transaction interface {
    Commit() error
    Rollback() error
    Execute(sql string) error
}

type EmailService interface {
    SendEmail(to, subject, body string) error
    ValidateEmail(email string) error
}

type FileSystem interface {
    ReadFile(filename string) ([]byte, error)
    WriteFile(filename string, data []byte) error
    FileExists(filename string) bool
}

// Service under test
type AutomationService struct {
    db    DatabaseClient
    email EmailService
    fs    FileSystem
    name  string
}

func NewAutomationService(db DatabaseClient, email EmailService, fs FileSystem, name string) *AutomationService {
    return &AutomationService{
        db:    db,
        email: email,
        fs:    fs,
        name:  name,
    }
}

func (as *AutomationService) ProcessUserRegistration(userEmail, userData string) error {
    // Validate email format
    if err := as.email.ValidateEmail(userEmail); err != nil {
        return fmt.Errorf("invalid email: %w", err)
    }

    // Check if user already exists
    results, err := as.db.Query("SELECT id FROM users WHERE email = '" + userEmail + "'")
    if err != nil {
        return fmt.Errorf("database query failed: %w", err)
    }

    if len(results) > 0 {
        return fmt.Errorf("user already exists: %s", userEmail)
    }

    // Save user data to file
    filename := fmt.Sprintf("user_%s.json", strings.ReplaceAll(userEmail, "@", "_at_"))
    if err := as.fs.WriteFile(filename, []byte(userData)); err != nil {
        return fmt.Errorf("failed to save user data: %w", err)
    }

    // Insert user into database
    insertSQL := fmt.Sprintf("INSERT INTO users (email, data_file) VALUES ('%s', '%s')", userEmail, filename)
    if err := as.db.Execute(insertSQL); err != nil {
        return fmt.Errorf("failed to insert user: %w", err)
    }

    // Send welcome email
    welcomeMsg := fmt.Sprintf("Welcome to our automation platform! Your data has been saved to %s", filename)
    if err := as.email.SendEmail(userEmail, "Welcome!", welcomeMsg); err != nil {
        return fmt.Errorf("failed to send welcome email: %w", err)
    }

    return nil
}

// TEST DOUBLE TYPE 1: STUB - Returns predefined responses
type StubDatabaseClient struct {
    queryResponse []map[string]interface{}
    queryError    error
    executeError  error
}

func (sdc *StubDatabaseClient) Query(sql string) ([]map[string]interface{}, error) {
    return sdc.queryResponse, sdc.queryError
}

func (sdc *StubDatabaseClient) Execute(sql string) error {
    return sdc.executeError
}

func (sdc *StubDatabaseClient) BeginTransaction() (Transaction, error) {
    return &StubTransaction{}, nil
}

type StubTransaction struct {
    commitError   error
    rollbackError error
    executeError  error
}

func (st *StubTransaction) Commit() error   { return st.commitError }
func (st *StubTransaction) Rollback() error { return st.rollbackError }
func (st *StubTransaction) Execute(sql string) error { return st.executeError }

// TEST DOUBLE TYPE 2: MOCK - Records interactions and verifies behavior
type MockEmailService struct {
    // State for verification
    validateEmailCalls []string
    sendEmailCalls     []EmailCall

    // Behavior configuration
    validateEmailError error
    sendEmailError     error
}

type EmailCall struct {
    To      string
    Subject string
    Body    string
}

func (mes *MockEmailService) ValidateEmail(email string) error {
    mes.validateEmailCalls = append(mes.validateEmailCalls, email)
    return mes.validateEmailError
}

func (mes *MockEmailService) SendEmail(to, subject, body string) error {
    mes.sendEmailCalls = append(mes.sendEmailCalls, EmailCall{
        To:      to,
        Subject: subject,
        Body:    body,
    })
    return mes.sendEmailError
}

// Verification methods for mock
func (mes *MockEmailService) VerifyValidateEmailCalled(email string) bool {
    for _, call := range mes.validateEmailCalls {
        if call == email {
            return true
        }
    }
    return false
}

func (mes *MockEmailService) VerifySendEmailCalled(to, subject string) bool {
    for _, call := range mes.sendEmailCalls {
        if call.To == to && call.Subject == subject {
            return true
        }
    }
    return false
}

func (mes *MockEmailService) GetSendEmailCallCount() int {
    return len(mes.sendEmailCalls)
}

// TEST DOUBLE TYPE 3: FAKE - Working implementation with shortcuts
type FakeFileSystem struct {
    files map[string][]byte // In-memory storage instead of real files

    // Behavior configuration
    readError  error
    writeError error
}

func NewFakeFileSystem() *FakeFileSystem {
    return &FakeFileSystem{
        files: make(map[string][]byte),
    }
}

func (ffs *FakeFileSystem) ReadFile(filename string) ([]byte, error) {
    if ffs.readError != nil {
        return nil, ffs.readError
    }

    if data, exists := ffs.files[filename]; exists {
        return data, nil
    }

    return nil, fmt.Errorf("file not found: %s", filename)
}

func (ffs *FakeFileSystem) WriteFile(filename string, data []byte) error {
    if ffs.writeError != nil {
        return ffs.writeError
    }

    ffs.files[filename] = data
    return nil
}

func (ffs *FakeFileSystem) FileExists(filename string) bool {
    _, exists := ffs.files[filename]
    return exists
}

// Helper methods for testing
func (ffs *FakeFileSystem) GetFileContent(filename string) ([]byte, bool) {
    data, exists := ffs.files[filename]
    return data, exists
}

func (ffs *FakeFileSystem) GetFileCount() int {
    return len(ffs.files)
}

// TEST EXAMPLES

func TestAutomationService_ProcessUserRegistration_Success(t *testing.T) {
    // Arrange - set up test doubles
    stubDB := &StubDatabaseClient{
        queryResponse: []map[string]interface{}{}, // Empty result = user doesn't exist
        queryError:    nil,
        executeError:  nil,
    }

    mockEmail := &MockEmailService{
        validateEmailError: nil,
        sendEmailError:     nil,
    }

    fakeFS := NewFakeFileSystem()

    service := NewAutomationService(stubDB, mockEmail, fakeFS, "TestService")

    // Act
    err := service.ProcessUserRegistration("test@example.com", `{"name": "Test User", "role": "admin"}`)

    // Assert
    if err != nil {
        t.Errorf("Expected no error, got: %v", err)
    }

    // Verify mock interactions
    if !mockEmail.VerifyValidateEmailCalled("test@example.com") {
        t.Error("Expected ValidateEmail to be called with test@example.com")
    }

    if !mockEmail.VerifySendEmailCalled("test@example.com", "Welcome!") {
        t.Error("Expected SendEmail to be called with welcome message")
    }

    if mockEmail.GetSendEmailCallCount() != 1 {
        t.Errorf("Expected 1 email to be sent, got %d", mockEmail.GetSendEmailCallCount())
    }

    // Verify fake file system state
    expectedFilename := "user_test_at_example.com.json"
    if !fakeFS.FileExists(expectedFilename) {
        t.Errorf("Expected file %s to be created", expectedFilename)
    }

    content, exists := fakeFS.GetFileContent(expectedFilename)
    if !exists {
        t.Error("Expected user data file to exist")
    } else if string(content) != `{"name": "Test User", "role": "admin"}` {
        t.Errorf("Expected user data to be saved correctly, got: %s", string(content))
    }
}

func TestAutomationService_ProcessUserRegistration_UserExists(t *testing.T) {
    // Arrange - stub returns existing user
    stubDB := &StubDatabaseClient{
        queryResponse: []map[string]interface{}{
            {"id": 1}, // User exists
        },
        queryError:   nil,
        executeError: nil,
    }

    mockEmail := &MockEmailService{}
    fakeFS := NewFakeFileSystem()

    service := NewAutomationService(stubDB, mockEmail, fakeFS, "TestService")

    // Act
    err := service.ProcessUserRegistration("existing@example.com", `{"name": "Existing User"}`)

    // Assert
    if err == nil {
        t.Error("Expected error when user already exists")
    }

    if !strings.Contains(err.Error(), "user already exists") {
        t.Errorf("Expected 'user already exists' error, got: %v", err)
    }

    // Verify no email was sent when user exists
    if mockEmail.GetSendEmailCallCount() != 0 {
        t.Error("Expected no email to be sent when user already exists")
    }

    // Verify no file was created when user exists
    if fakeFS.GetFileCount() != 0 {
        t.Error("Expected no files to be created when user already exists")
    }
}

func TestAutomationService_ProcessUserRegistration_EmailValidationFails(t *testing.T) {
    // Arrange - mock email service returns validation error
    stubDB := &StubDatabaseClient{}

    mockEmail := &MockEmailService{
        validateEmailError: fmt.Errorf("invalid email format"),
    }

    fakeFS := NewFakeFileSystem()

    service := NewAutomationService(stubDB, mockEmail, fakeFS, "TestService")

    // Act
    err := service.ProcessUserRegistration("invalid-email", `{"name": "Test User"}`)

    // Assert
    if err == nil {
        t.Error("Expected error when email validation fails")
    }

    if !strings.Contains(err.Error(), "invalid email") {
        t.Errorf("Expected 'invalid email' error, got: %v", err)
    }

    // Verify email validation was called
    if !mockEmail.VerifyValidateEmailCalled("invalid-email") {
        t.Error("Expected ValidateEmail to be called")
    }

    // Verify no welcome email was sent
    if mockEmail.GetSendEmailCallCount() != 0 {
        t.Error("Expected no welcome email when validation fails")
    }
}

func main() {
    fmt.Println("=== Mock Implementations and Test Doubles ===")

    fmt.Println("\n=== Running Tests ===")

    // Simulate running tests
    t := &testing.T{}

    fmt.Println("Test 1: Successful user registration")
    TestAutomationService_ProcessUserRegistration_Success(t)

    fmt.Println("Test 2: User already exists")
    TestAutomationService_ProcessUserRegistration_UserExists(t)

    fmt.Println("Test 3: Email validation fails")
    TestAutomationService_ProcessUserRegistration_EmailValidationFails(t)

    fmt.Println("\n=== Test Double Types Summary ===")
    fmt.Println("1. STUB: Returns predefined responses, minimal logic")
    fmt.Println("2. MOCK: Records interactions, verifies behavior")
    fmt.Println("3. FAKE: Working implementation with shortcuts (in-memory)")
    fmt.Println("4. SPY: Records calls while delegating to real implementation")
    fmt.Println("5. DUMMY: Placeholder objects, never actually used")

    fmt.Println("\n=== Best Practices ===")
    fmt.Println("- Use stubs for simple return value testing")
    fmt.Println("- Use mocks to verify interactions and behavior")
    fmt.Println("- Use fakes for complex state-based testing")
    fmt.Println("- Keep test doubles simple and focused")
    fmt.Println("- Verify both state and behavior as appropriate")
    fmt.Println("- Don't over-mock - test real behavior when possible")
}
```

**Hands-on Exercise 3: Consumer-Defined Interfaces for Testing**:

```go
// Demonstrating the Go approach: consumers define interfaces they need
package main

import (
    "fmt"
    "time"
)

// SCENARIO: You're building a message bus client library
// The library provides concrete implementations, consumers define interfaces for testing

// ===== LIBRARY CODE (pubsub package) =====
// This is what the library author provides - concrete types only

type PubSubClient struct {
    brokerURL    string
    clientID     string
    connected    bool
    subscriptions map[string][]MessageHandler
}

type Message struct {
    Topic     string
    Payload   []byte
    Timestamp time.Time
    MessageID string
}

type MessageHandler func(msg Message) error

// Constructor returns concrete type, not interface
func NewPubSubClient(brokerURL, clientID string) *PubSubClient {
    return &PubSubClient{
        brokerURL:     brokerURL,
        clientID:      clientID,
        connected:     false,
        subscriptions: make(map[string][]MessageHandler),
    }
}

func (psc *PubSubClient) Connect() error {
    fmt.Printf("PubSub: Connecting to %s with client ID %s\n", psc.brokerURL, psc.clientID)
    // In real implementation: establish connection to message broker
    time.Sleep(50 * time.Millisecond) // Simulate connection time
    psc.connected = true
    fmt.Printf("PubSub: Connected successfully\n")
    return nil
}

func (psc *PubSubClient) Disconnect() error {
    fmt.Printf("PubSub: Disconnecting client %s\n", psc.clientID)
    // In real implementation: close connection to message broker
    psc.connected = false
    fmt.Printf("PubSub: Disconnected successfully\n")
    return nil
}

func (psc *PubSubClient) Publish(topic string, payload []byte) error {
    if !psc.connected {
        return fmt.Errorf("client not connected")
    }

    fmt.Printf("PubSub: Publishing to topic '%s': %s\n", topic, string(payload))
    // In real implementation: send message to broker
    time.Sleep(10 * time.Millisecond) // Simulate network call
    return nil
}

func (psc *PubSubClient) Subscribe(topic string, handler MessageHandler) error {
    if !psc.connected {
        return fmt.Errorf("client not connected")
    }

    fmt.Printf("PubSub: Subscribing to topic '%s'\n", topic)

    if psc.subscriptions[topic] == nil {
        psc.subscriptions[topic] = make([]MessageHandler, 0)
    }
    psc.subscriptions[topic] = append(psc.subscriptions[topic], handler)

    // In real implementation: register subscription with broker
    return nil
}

func (psc *PubSubClient) SimulateIncomingMessage(topic string, payload []byte) {
    // This simulates receiving a message from the broker
    if handlers, exists := psc.subscriptions[topic]; exists {
        msg := Message{
            Topic:     topic,
            Payload:   payload,
            Timestamp: time.Now(),
            MessageID: fmt.Sprintf("msg_%d", time.Now().Unix()),
        }

        for _, handler := range handlers {
            if err := handler(msg); err != nil {
                fmt.Printf("PubSub: Handler error for topic '%s': %v\n", topic, err)
            }
        }
    }
}

func (psc *PubSubClient) GetConnectionStatus() bool {
    return psc.connected
}

func (psc *PubSubClient) GetSubscriptionCount() int {
    count := 0
    for _, handlers := range psc.subscriptions {
        count += len(handlers)
    }
    return count
}

// ===== CONSUMER CODE =====
// This is what the application developer writes

// STEP 1: Consumer defines interface for ONLY the methods they use
// This is the key insight: the consumer defines the interface, not the library

type MessagePublisher interface {
    Connect() error
    Publish(topic string, payload []byte) error
    Disconnect() error
}

type MessageSubscriber interface {
    Connect() error
    Subscribe(topic string, handler MessageHandler) error
    Disconnect() error
}

// Some consumers might need both
type MessageBroker interface {
    MessagePublisher
    MessageSubscriber
}

// STEP 2: Application service uses the interface
type AutomationEventService struct {
    publisher MessagePublisher
    subscriber MessageSubscriber
    serviceName string
}

// Constructor accepts interfaces, not concrete types
func NewAutomationEventService(pub MessagePublisher, sub MessageSubscriber, name string) *AutomationEventService {
    return &AutomationEventService{
        publisher:   pub,
        subscriber:  sub,
        serviceName: name,
    }
}

func (aes *AutomationEventService) Start() error {
    fmt.Printf("EventService %s: Starting...\n", aes.serviceName)

    // Connect publisher and subscriber
    if err := aes.publisher.Connect(); err != nil {
        return fmt.Errorf("failed to connect publisher: %w", err)
    }

    if err := aes.subscriber.Connect(); err != nil {
        return fmt.Errorf("failed to connect subscriber: %w", err)
    }

    // Subscribe to automation events
    if err := aes.subscriber.Subscribe("automation.task.completed", aes.handleTaskCompleted); err != nil {
        return fmt.Errorf("failed to subscribe to task events: %w", err)
    }

    if err := aes.subscriber.Subscribe("automation.alert.critical", aes.handleCriticalAlert); err != nil {
        return fmt.Errorf("failed to subscribe to alert events: %w", err)
    }

    fmt.Printf("EventService %s: Started successfully\n", aes.serviceName)
    return nil
}

func (aes *AutomationEventService) PublishTaskStarted(taskID, taskType string) error {
    payload := fmt.Sprintf(`{"task_id":"%s","task_type":"%s","status":"started","timestamp":"%s"}`,
        taskID, taskType, time.Now().Format(time.RFC3339))

    return aes.publisher.Publish("automation.task.started", []byte(payload))
}

func (aes *AutomationEventService) PublishTaskCompleted(taskID string, success bool) error {
    status := "completed"
    if !success {
        status = "failed"
    }

    payload := fmt.Sprintf(`{"task_id":"%s","status":"%s","timestamp":"%s"}`,
        taskID, status, time.Now().Format(time.RFC3339))

    return aes.publisher.Publish("automation.task.completed", []byte(payload))
}

func (aes *AutomationEventService) handleTaskCompleted(msg Message) error {
    fmt.Printf("EventService %s: Task completed event received: %s\n", aes.serviceName, string(msg.Payload))
    // Process task completion logic here
    return nil
}

func (aes *AutomationEventService) handleCriticalAlert(msg Message) error {
    fmt.Printf("EventService %s: CRITICAL ALERT received: %s\n", aes.serviceName, string(msg.Payload))
    // Handle critical alert logic here
    return nil
}

func (aes *AutomationEventService) Stop() error {
    fmt.Printf("EventService %s: Stopping...\n", aes.serviceName)

    if err := aes.publisher.Disconnect(); err != nil {
        fmt.Printf("Warning: failed to disconnect publisher: %v\n", err)
    }

    if err := aes.subscriber.Disconnect(); err != nil {
        fmt.Printf("Warning: failed to disconnect subscriber: %v\n", err)
    }

    fmt.Printf("EventService %s: Stopped\n", aes.serviceName)
    return nil
}

// STEP 3: Consumer creates test doubles for THEIR interfaces
type MockMessagePublisher struct {
    connected      bool
    publishedMsgs  []PublishedMessage
    connectError   error
    publishError   error
    disconnectError error
}

type PublishedMessage struct {
    Topic   string
    Payload string
}

func (mmp *MockMessagePublisher) Connect() error {
    if mmp.connectError != nil {
        return mmp.connectError
    }
    mmp.connected = true
    return nil
}

func (mmp *MockMessagePublisher) Publish(topic string, payload []byte) error {
    if !mmp.connected {
        return fmt.Errorf("not connected")
    }

    if mmp.publishError != nil {
        return mmp.publishError
    }

    mmp.publishedMsgs = append(mmp.publishedMsgs, PublishedMessage{
        Topic:   topic,
        Payload: string(payload),
    })
    return nil
}

func (mmp *MockMessagePublisher) Disconnect() error {
    mmp.connected = false
    return mmp.disconnectError
}

func (mmp *MockMessagePublisher) GetPublishedMessages() []PublishedMessage {
    return mmp.publishedMsgs
}

func (mmp *MockMessagePublisher) WasMessagePublished(topic string) bool {
    for _, msg := range mmp.publishedMsgs {
        if msg.Topic == topic {
            return true
        }
    }
    return false
}

type MockMessageSubscriber struct {
    connected       bool
    subscriptions   map[string]MessageHandler
    connectError    error
    subscribeError  error
    disconnectError error
}

func NewMockMessageSubscriber() *MockMessageSubscriber {
    return &MockMessageSubscriber{
        subscriptions: make(map[string]MessageHandler),
    }
}

func (mms *MockMessageSubscriber) Connect() error {
    if mms.connectError != nil {
        return mms.connectError
    }
    mms.connected = true
    return nil
}

func (mms *MockMessageSubscriber) Subscribe(topic string, handler MessageHandler) error {
    if !mms.connected {
        return fmt.Errorf("not connected")
    }

    if mms.subscribeError != nil {
        return mms.subscribeError
    }

    mms.subscriptions[topic] = handler
    return nil
}

func (mms *MockMessageSubscriber) Disconnect() error {
    mms.connected = false
    return mms.disconnectError
}

// Test helper to simulate incoming messages
func (mms *MockMessageSubscriber) SimulateMessage(topic string, payload []byte) error {
    if handler, exists := mms.subscriptions[topic]; exists {
        msg := Message{
            Topic:     topic,
            Payload:   payload,
            Timestamp: time.Now(),
            MessageID: "test-msg-123",
        }
        return handler(msg)
    }
    return fmt.Errorf("no handler for topic: %s", topic)
}

func (mms *MockMessageSubscriber) IsSubscribedTo(topic string) bool {
    _, exists := mms.subscriptions[topic]
    return exists
}

// STEP 4: Tests using consumer-defined interfaces and mocks
func TestAutomationEventService_PublishTaskStarted() {
    fmt.Println("=== Test: PublishTaskStarted ===")

    // Arrange
    mockPub := &MockMessagePublisher{}
    mockSub := NewMockMessageSubscriber()

    service := NewAutomationEventService(mockPub, mockSub, "TestService")
    service.Start()

    // Act
    err := service.PublishTaskStarted("task-123", "data-processing")

    // Assert
    if err != nil {
        fmt.Printf("❌ Unexpected error: %v\n", err)
        return
    }

    if !mockPub.WasMessagePublished("automation.task.started") {
        fmt.Printf("❌ Expected task started message to be published\n")
        return
    }

    messages := mockPub.GetPublishedMessages()
    if len(messages) != 1 {
        fmt.Printf("❌ Expected 1 message, got %d\n", len(messages))
        return
    }

    fmt.Printf("✅… Task started message published successfully: %s\n", messages[0].Payload)
}

func TestAutomationEventService_HandleTaskCompleted() {
    fmt.Println("\n=== Test: HandleTaskCompleted ===")

    // Arrange
    mockPub := &MockMessagePublisher{}
    mockSub := NewMockMessageSubscriber()

    service := NewAutomationEventService(mockPub, mockSub, "TestService")
    service.Start()

    // Act - simulate incoming message
    testPayload := []byte(`{"task_id":"task-456","status":"completed","timestamp":"2023-01-01T10:00:00Z"}`)
    err := mockSub.SimulateMessage("automation.task.completed", testPayload)

    // Assert
    if err != nil {
        fmt.Printf("❌ Unexpected error handling message: %v\n", err)
        return
    }

    if !mockSub.IsSubscribedTo("automation.task.completed") {
        fmt.Printf("❌ Expected service to be subscribed to task completed events\n")
        return
    }

    fmt.Printf("✅… Task completed message handled successfully\n")
}

func main() {
    fmt.Println("=== Consumer-Defined Interfaces for Testing ===")

    fmt.Println("\n=== Production Usage ===")

    // Production: use concrete PubSub client
    pubsubClient := NewPubSubClient("amqp://localhost:5672", "automation-service-1")

    // Consumer's service uses the concrete client through interfaces
    eventService := NewAutomationEventService(pubsubClient, pubsubClient, "ProductionService")

    eventService.Start()

    // Publish some events
    eventService.PublishTaskStarted("task-001", "data-backup")
    eventService.PublishTaskCompleted("task-001", true)

    // Simulate incoming messages (in real app, these come from the broker)
    pubsubClient.SimulateIncomingMessage("automation.task.completed",
        []byte(`{"task_id":"task-002","status":"completed"}`))

    pubsubClient.SimulateIncomingMessage("automation.alert.critical",
        []byte(`{"alert":"disk_space_low","severity":"critical"}`))

    eventService.Stop()

    fmt.Println("\n=== Testing Usage ===")

    // Testing: use mocks defined by consumer
    TestAutomationEventService_PublishTaskStarted()
    TestAutomationEventService_HandleTaskCompleted()

    fmt.Println("\n=== Key Insights ===")
    fmt.Println("1. Library provides concrete types, not interfaces")
    fmt.Println("2. Consumer defines interfaces for ONLY methods they use")
    fmt.Println("3. Consumer creates their own test doubles")
    fmt.Println("4. Library author doesn't need to worry about consumer testing")
    fmt.Println("5. Convention over configuration reduces coupling")
    fmt.Println("6. Each consumer gets exactly the interface they need")
    fmt.Println("7. Library stays clean and focused on real functionality")
}
```

**Prerequisites**: Module 22
