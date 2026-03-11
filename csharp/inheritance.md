# Inheritance and Polymorphism in C# - Advanced Mastery

## What is Inheritance and Polymorphism?

Inheritance is an object-oriented programming mechanism where a class (derived/child class) acquires properties and behaviors from another class (base/parent class). Polymorphism (Greek for "many forms") allows objects of different types to be treated as objects of a common base type, enabling method overriding and dynamic method dispatch.

**Common Aliases:** 
- Inheritance: "IS-A" relationship, class hierarchy, derivation
- Polymorphism: Method overriding, dynamic binding, runtime polymorphism

**Core Purpose:** Inheritance enables code reuse and establishes hierarchical relationships, while polymorphism provides flexibility by allowing derived classes to implement base class methods in their own way. Together, they solve the problem of code duplication and enable scalable, maintainable software design.

## How it works in C#

### Abstract Classes

**Explanation:** Abstract classes are classes that cannot be instantiated directly and serve as base classes for other classes. They can contain both implemented methods and abstract methods (without implementation) that must be implemented by derived classes.

```csharp
// Base abstract class defining common structure
public abstract class Shape
{
    protected string _name;
    
    // Abstract method - must be implemented by derived classes
    public abstract double CalculateArea();
    
    // Concrete method with implementation
    public virtual void Display()
    {
        Console.WriteLine($"Shape: {_name}");
    }
    
    // Constructor for base class
    protected Shape(string name)
    {
        _name = name;
    }
}

// Derived class implementing the abstract method
public class Circle : Shape
{
    public double Radius { get; set; }
    
    public Circle(string name, double radius) : base(name)
    {
        Radius = radius;
    }
    
    // Mandatory implementation of abstract method
    public override double CalculateArea()
    {
        return Math.PI * Radius * Radius;
    }
    
    // Optional override of virtual method
    public override void Display()
    {
        Console.WriteLine($"Circle '{_name}' with radius {Radius}");
    }
}
```

### Virtual Methods

**Explanation:** Virtual methods are methods in a base class that can be overridden in derived classes using the `override` keyword. They enable runtime polymorphism by allowing the most derived implementation to be called.

```csharp
public class Vehicle
{
    public string Model { get; set; }
    
    // Virtual method - can be overridden by derived classes
    public virtual void StartEngine()
    {
        Console.WriteLine($"Starting {Model} engine with key ignition");
    }
    
    // Non-virtual method - cannot be overridden (only hidden with 'new')
    public void StopEngine()
    {
        Console.WriteLine("Engine stopped");
    }
}

public class ElectricCar : Vehicle
{
    // Overriding virtual method with specialized implementation
    public override void StartEngine()
    {
        Console.WriteLine($"Starting {Model} with silent electric power");
    }
    
    // Hiding base method (not recommended - use virtual/override pattern)
    public new void StopEngine()
    {
        Console.WriteLine("Electric motor disengaged");
    }
}

// Usage demonstrating polymorphism
Vehicle myCar = new ElectricCar { Model = "Tesla Model S" };
myCar.StartEngine(); // Calls ElectricCar's implementation due to polymorphism
```

### Interface Implementation

**Explanation:** Interfaces define contracts that classes must implement. Unlike abstract classes, interfaces contain only method signatures without implementations. A class can implement multiple interfaces.

```csharp
// Interface defining a contract
public interface ILoggable
{
    void Log(string message);
    string GetLogHeader();
}

public interface IIdentifiable
{
    Guid Id { get; }
}

// Class implementing multiple interfaces
public class DatabaseService : ILoggable, IIdentifiable
{
    public Guid Id { get; } = Guid.NewGuid();
    private List<string> _logEntries = new List<string>();
    
    // Explicit interface implementation
    void ILoggable.Log(string message)
    {
        _logEntries.Add($"{DateTime.Now}: {message}");
    }
    
    string ILoggable.GetLogHeader()
    {
        return $"Database Service {Id} - Log entries: {_logEntries.Count}";
    }
    
    // Regular class method
    public void ClearLogs()
    {
        _logEntries.Clear();
    }
}

// Usage with interface-based polymorphism
ILoggable logger = new DatabaseService();
logger.Log("Database connection established");
Console.WriteLine(logger.GetLogHeader());
```

## Why is Inheritance and Polymorphism Important?

1. **DRY Principle (Don't Repeat Yourself):** Inheritance eliminates code duplication by allowing common functionality to be defined once in base classes and reused across multiple derived classes.

2. **Open/Closed Principle (SOLID):** Polymorphism enables systems to be open for extension but closed for modification, allowing new functionality through inheritance without changing existing code.

3. **Liskov Substitution Principle (SOLID):** Proper inheritance hierarchies ensure that derived classes can substitute base classes without breaking functionality, enabling robust and scalable architectures.

## Advanced Nuances

### 1. Covariant Return Types (C# 9.0+)
```csharp
public abstract class Animal
{
    public abstract Animal Clone();
}

public class Dog : Animal
{
    // Covariant return type - can return more derived type
    public override Dog Clone()
    {
        return new Dog();
    }
}
```

### 2. Explicit Interface Implementation for Diamond Problem Resolution
```csharp
interface IA { void Method(); }
interface IB { void Method(); }

class Diamond : IA, IB
{
    // Explicit implementations resolve method ambiguity
    void IA.Method() => Console.WriteLine("IA implementation");
    void IB.Method() => Console.WriteLine("IB implementation");
}
```

### 3. Protected Internal Access Modifier Combination
```csharp
public class BaseClass
{
    // Accessible within same assembly OR by derived classes in any assembly
    protected internal string HybridAccessField;
}
```

## How This Fits the Roadmap

In the "Object Oriented Programming" section of the Advanced C# Mastery roadmap, Inheritance and Polymorphism serve as the foundation for more advanced concepts. This topic is a prerequisite for understanding:

- **Design Patterns:** Factory, Strategy, Template Method, and other GoF patterns rely heavily on polymorphism
- **Dependency Injection:** Modern DI containers use polymorphism to resolve dependencies at runtime
- **Entity Framework & ORM:** Navigation properties and inheritance mapping strategies build upon these concepts
- **ASP.NET Core Middleware:** The pipeline pattern uses polymorphism extensively for request handling

Mastering inheritance and polymorphism unlocks advanced topics like generic constraints, reflection-based type handling, and sophisticated architectural patterns that form the core of enterprise-level C# development.