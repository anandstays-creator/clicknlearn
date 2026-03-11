# Switch Statement Smells in C#

## What is Switch Statement Smells?

**Switch Statement Smell** (also known as "Switch Statements" or "Conditional Complexity") refers to the anti-pattern where extensive `switch` statements or long `if/else if` chains are used to handle type-based or state-based logic instead of leveraging object-oriented principles. This smell violates the fundamental OOP concept that behavior should live with data, not in external conditional logic that must constantly be updated when new types or states are added.

The core problem it solves is **maintainability**: when you need to add a new case, you're forced to modify existing code rather than extending it through polymorphism. This violates the Open/Closed Principle and creates fragile, tightly-coupled systems.

## How it works in C#

### Polymorphism

Polymorphism allows objects of different types to be treated uniformly through a common interface, moving behavior decisions from conditional logic to method dispatch.

```csharp
// Smelly approach
public class OrderProcessor
{
    public decimal CalculateDiscount(string customerType)
    {
        switch (customerType)
        {
            case "Regular": return 0.05m;
            case "Premium": return 0.15m;
            case "VIP": return 0.25m;
            default: return 0m;
        }
    }
}

// Polymorphic solution
public interface ICustomer
{
    decimal CalculateDiscount();
}

public class RegularCustomer : ICustomer
{
    public decimal CalculateDiscount() => 0.05m;
}

public class PremiumCustomer : ICustomer
{
    public decimal CalculateDiscount() => 0.15m;
}

public class VIPCustomer : ICustomer
{
    public decimal CalculateDiscount() => 0.25m;
}

// Usage - no switch needed
public class OrderProcessor
{
    public decimal CalculateDiscount(ICustomer customer)
    {
        return customer.CalculateDiscount(); // Polymorphic dispatch
    }
}
```

### Pattern Matching

C# 7.0+ introduced enhanced pattern matching that can be a better alternative to traditional switches, especially when you can't modify the class hierarchy.

```csharp
// Traditional smelly switch
public string ProcessShape(object shape)
{
    if (shape is Circle c)
        return $"Circle with radius {c.Radius}";
    else if (shape is Rectangle r)
        return $"Rectangle {r.Width}x{r.Height}";
    else
        return "Unknown shape";
}

// Modern pattern matching approach
public string ProcessShape(object shape)
{
    return shape switch
    {
        Circle { Radius: var r } => $"Circle with radius {r}",
        Rectangle { Width: var w, Height: var h } => $"Rectangle {w}x{h}",
        { } => "Unknown shape",  // Pattern for any non-null object
        null => "Null shape"
    };
}

// More advanced with property patterns
public decimal CalculateArea(object shape) => shape switch
{
    Circle { Radius: > 0 } c => (decimal)(Math.PI * c.Radius * c.Radius),
    Rectangle { Width: var w, Height: var h } when w > 0 && h > 0 => w * h,
    _ => throw new ArgumentException("Invalid shape")
};
```

### State/Factory Patterns

The State pattern encapsulates state-specific behavior, while the Factory pattern centralizes object creation, eliminating switch statements from client code.

```csharp
// Smelly state management
public class Order
{
    public string Status { get; set; }
    
    public void Process()
    {
        switch (Status)
        {
            case "Pending": /* process pending */ break;
            case "Approved": /* process approved */ break;
            case "Shipped": /* process shipped */ break;
        }
    }
}

// State pattern implementation
public interface IOrderState
{
    void Process(Order order);
    bool CanCancel(Order order);
}

public class PendingState : IOrderState
{
    public void Process(Order order) 
    {
        // Pending-specific logic
        order.State = OrderStateFactory.CreateApprovedState();
    }
    
    public bool CanCancel(Order order) => true;
}

public class OrderStateFactory
{
    public static IOrderState CreatePendingState() => new PendingState();
    public static IOrderState CreateApprovedState() => new ApprovedState();
    
    // Factory eliminates switch statements when creating states
    public static IOrderState CreateFromStatus(string status) => status switch
    {
        "Pending" => new PendingState(),
        "Approved" => new ApprovedState(),
        _ => throw new ArgumentException($"Unknown status: {status}")
    };
}

public class Order
{
    public IOrderState State { get; set; }
    
    public Order() => State = OrderStateFactory.CreatePendingState();
    
    public void Process() => State.Process(this);
}
```

## Why is Switch Statement Smells important?

1. **Open/Closed Principle Compliance** - The system becomes open for extension (new types/states) but closed for modification, as new behavior requires adding new classes rather than modifying existing conditional logic.

2. **Single Responsibility Principle** - Each class handles its own behavior, rather than having a "god class" that knows how to handle all possible cases through switch statements.

3. **Reduced Coupling** - Client code depends on abstractions (interfaces) rather than concrete implementations, making the system more flexible and testable.

## Advanced Nuances

### 1. Strategic Pattern Matching Bridge
When working with external types you can't modify, combine pattern matching with the Strategy pattern:

```csharp
public interface IExternalTypeHandler<T>
{
    bool CanHandle(T item);
    void Handle(T item);
}

// Register handlers and eliminate switch statements
public class ExternalTypeProcessor
{
    private readonly List<IExternalTypeHandler<object>> _handlers;
    
    public void Process(object item)
    {
        var handler = _handlers.FirstOrDefault(h => h.CanHandle(item));
        handler?.Handle(item); // No switch needed
    }
}
```

### 2. Visitor Pattern for Double Dispatch
When you need to perform operations based on two polymorphic types:

```csharp
public interface IShapeVisitor<T>
{
    T Visit(Circle circle);
    T Visit(Rectangle rectangle);
}

public interface IShape
{
    T Accept<T>(IShapeVisitor<T> visitor);
}

// Eliminates the need for type checking switches when processing shapes
public class AreaCalculator : IShapeVisitor<decimal>
{
    public decimal Visit(Circle circle) => (decimal)(Math.PI * circle.Radius * circle.Radius);
    public decimal Visit(Rectangle rectangle) => rectangle.Width * rectangle.Height;
}
```

### 3. When to Keep Switch Statements
Sometimes, switch statements are appropriate - particularly for:
- **Configuration mapping** where the mapping is inherently static
- **Performance-critical code** where virtual calls are too expensive
- **Simple value mappings** that don't represent behavioral differences

## How this fits the Roadmap

Within the "Object Orientation Smells" section, Switch Statement Smells serves as a **prerequisite** for understanding more complex refactoring techniques. It's foundational for recognizing when object-oriented principles are being violated through procedural coding patterns.

This concept **unlocks** more advanced topics including:
- **Refactoring to Patterns** - Learning how to systematically replace procedural code with design patterns
- **Domain-Driven Design** - Understanding how to model business logic using rich domain models rather than procedural services
- **Architectural Principles** - Applying the same "behavior with data" thinking at the architectural level

Mastering this smell enables developers to spot abstraction opportunities throughout the codebase, making it a crucial stepping stone toward advanced C# mastery and clean architecture design.