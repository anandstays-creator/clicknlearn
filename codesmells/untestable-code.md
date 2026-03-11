# Untestable Code in C#: A Core Testability Concept

## What is Untestable Code?

**Untestable Code** (often called "Hard-to-Test Code" or "Tightly Coupled Code") refers to software design where dependencies are so tightly intertwined that unit testing becomes impractical or impossible without complex workarounds. The core problem it addresses is **dependency coupling** - when classes directly instantiate their dependencies instead of receiving them externally, making isolated testing unfeasible.

## How it works in C#

### Constructor Injection

**Explanation**: Constructor injection is the practice of requiring dependencies as parameters in a class's constructor rather than instantiating them internally. This enables dependency inversion and makes classes testable by allowing mock implementations to be passed during testing.

**Code Example**:
```csharp
// ✅ Testable version using constructor injection
public class OrderProcessor
{
    private readonly IPaymentGateway _paymentGateway;
    private readonly IInventoryService _inventoryService;
    
    // Dependencies are injected via constructor
    public OrderProcessor(IPaymentGateway paymentGateway, IInventoryService inventoryService)
    {
        _paymentGateway = paymentGateway ?? throw new ArgumentNullException(nameof(paymentGateway));
        _inventoryService = inventoryService ?? throw new ArgumentNullException(nameof(inventoryService));
    }
    
    public bool ProcessOrder(Order order)
    {
        // Business logic uses injected dependencies
        var paymentResult = _paymentGateway.ProcessPayment(order.TotalAmount);
        if (paymentResult.Success)
        {
            return _inventoryService.UpdateStock(order.ProductId, order.Quantity);
        }
        return false;
    }
}

// ❌ Untestable version - dependencies created internally
public class UntestableOrderProcessor
{
    private readonly PaymentGateway _paymentGateway;
    private readonly InventoryService _inventoryService;
    
    public UntestableOrderProcessor()
    {
        // Direct instantiation makes testing impossible
        _paymentGateway = new PaymentGateway(); // Cannot mock this!
        _inventoryService = new InventoryService(); // Real implementation always used
    }
}
```

### Static Method Removal

**Explanation**: Static methods create implicit dependencies that are difficult to mock or replace. Removing static calls in favor of instance methods with interfaces enables proper testing by allowing dependency substitution.

**Code Example**:
```csharp
// ✅ Testable version - static dependency abstracted behind interface
public interface ILogger
{
    void Log(string message);
}

public class FileLogger : ILogger
{
    public void Log(string message)
    {
        File.WriteAllText("log.txt", message); // Still uses static, but abstracted
    }
}

public class OrderService
{
    private readonly ILogger _logger;
    
    public OrderService(ILogger logger)
    {
        _logger = logger;
    }
    
    public void CreateOrder(Order order)
    {
        // Business logic with testable dependency
        _logger.Log($"Creating order {order.Id}");
        // ... order creation logic
    }
}

// ❌ Untestable version - direct static method usage
public class UntestableOrderService
{
    public void CreateOrder(Order order)
    {
        // Cannot mock or control this static call during tests
        Logger.LogToFile($"Creating order {order.Id}"); // Static method - untestable!
        // ... order creation logic
    }
}

public static class Logger
{
    public static void LogToFile(string message)
    {
        File.WriteAllText("log.txt", message); // Hard dependency
    }
}
```

### Global State Avoidance

**Explanation**: Global state (static fields, singletons, service locators) creates hidden dependencies that make tests unpredictable and non-isolated. Avoiding global state ensures each test runs in isolation without side effects from previous tests.

**Code Example**:
```csharp
// ✅ Testable version - state is managed locally or injected
public class UserSession : IUserSession
{
    public string CurrentUserId { get; set; }
}

public class OrderAuthorizationService
{
    private readonly IUserSession _userSession;
    
    public OrderAuthorizationService(IUserSession userSession)
    {
        _userSession = userSession;
    }
    
    public bool CanViewOrder(Order order)
    {
        // Uses injected state - easily mockable
        return order.UserId == _userSession.CurrentUserId;
    }
}

// ❌ Untestable version - relies on global state
public static class GlobalSession
{
    public static string CurrentUserId { get; set; } // Global state - test pollution!
}

public class UntestableAuthorizationService
{
    public bool CanViewOrder(Order order)
    {
        // Depends on global state - tests affect each other
        return order.UserId == GlobalSession.CurrentUserId; // Unreliable in tests!
    }
}

// Test demonstrating the problem
[Test]
public void TestOrderAuthorization()
{
    // This test might fail if another test modified GlobalSession!
    GlobalSession.CurrentUserId = "user123";
    var service = new UntestableAuthorizationService();
    var order = new Order { UserId = "user123" };
    
    Assert.IsTrue(service.CanViewOrder(order)); // Might fail due to test pollution
}
```

## Why is Untestable Code important?

1. **SOLID Principles Compliance** - Enables Dependency Inversion Principle (D) by ensuring high-level modules don't depend on low-level modules directly.
2. **Test Isolation** - Supports reliable unit testing by eliminating shared state and allowing mock dependencies, ensuring tests don't affect each other.
3. **Maintainability (Open/Closed Principle)** - Makes code more flexible to change by reducing coupling, allowing new implementations to be swapped without modifying existing code.

## Advanced Nuances

### 1. Legacy Code Integration
When working with legacy systems that heavily use static methods or singletons, you can create **interception layers** without full rewrites:

```csharp
// Adapter pattern for legacy static dependencies
public class LegacyStaticAdapter : IEmailService
{
    public void SendEmail(string to, string subject)
    {
        // Wrapper around untestable legacy code
        LegacyEmailService.Send(to, subject); // Can't change this, but can abstract it
    }
}

// Now testable through the interface
public class OrderNotifier
{
    private readonly IEmailService _emailService;
    public OrderNotifier(IEmailService emailService) => _emailService = emailService;
    
    public void NotifyOrderShipped(Order order)
    {
        _emailService.SendEmail(order.CustomerEmail, "Your order shipped!");
    }
}
```

### 2. Partial Mocking Strategy
For cases where you must test code that mixes testable and untestable elements:

```csharp
public class HybridService
{
    private readonly IConfigService _configService;
    private readonly string _environment;
    
    public HybridService(IConfigService configService)
    {
        _configService = configService;
        _environment = Environment.MachineName; // Static call - hard to test
    }
    
    public string GetServiceUrl()
    {
        return _configService.GetBaseUrl() + _environment;
    }
}

// Testing strategy: test what you can, isolate what you can't
[Test]
public void TestGetServiceUrl_ConfigurablePart()
{
    var mockConfig = new Mock<IConfigService>();
    mockConfig.Setup(x => x.GetBaseUrl()).Returns("https://api.");
    
    var service = new HybridService(mockConfig.Object);
    
    // Only assert the testable part, acknowledge the static dependency
    StringAssert.StartsWith("https://api.", service.GetServiceUrl());
}
```

### 3. Composition Root Pattern
Advanced dependency management where object graph construction is centralized:

```csharp
// Composition root - only place where dependencies are wired up
public static class CompositionRoot
{
    public static OrderProcessor BuildOrderProcessor()
    {
        return new OrderProcessor(
            new PaymentGateway(),      // Real implementation
            new InventoryService(),    // Real implementation  
            new FileLogger()          // Real implementation
        );
    }
    
    public static OrderProcessor BuildTestOrderProcessor(
        IPaymentGateway paymentGateway = null,
        IInventoryService inventoryService = null)
    {
        return new OrderProcessor(
            paymentGateway ?? new MockPaymentGateway(),
            inventoryService ?? new MockInventoryService()
        );
    }
}
```

## How this fits the Roadmap

Within the "Testability Smells" section, **Untestable Code** serves as the **foundational concept** that precedes more specific anti-patterns. It's the prerequisite for understanding:

- **What comes before**: Basic dependency management and inversion of control principles
- **What it enables**: More advanced testability patterns like:
  - **Mocking Frameworks Integration** - Building on constructor injection
  - **Test-Driven Development (TDD)** - Requires testable code design from the start
  - **Architectural Patterns** - Clean Architecture, Onion Architecture that rely on dependency inversion

This concept unlocks the ability to identify and refactor common testability issues like "new" keywords in business logic, static cling, and temporal coupling - making it the gateway to mastering enterprise-scale testable C# applications.