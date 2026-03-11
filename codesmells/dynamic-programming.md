# Dynamic Programming in C# - Advanced Mastery Guide

## What is Dynamic Programming?

**Dynamic Programming** in C# (commonly called *dynamic typing* or *late binding*) is a programming paradigm that defers type resolution from compile-time to runtime. Its core purpose is to enable more flexible, adaptive code that can handle scenarios where static type information is unavailable or impractical. This solves problems like interoperability with dynamic languages, working with loosely-structured data, and building highly adaptable systems.

## How it works in C#

### Dynamic keyword

The `dynamic` keyword tells the compiler to bypass static type checking and resolve method calls, property access, and operations at runtime. The Dynamic Language Runtime (DLR) handles the dispatch.

```csharp
using System;

class Program
{
    static void Main()
    {
        // Compiler bypasses type checking - resolves at runtime
        dynamic dynamicValue = GetDynamicData();
        
        // These calls are resolved at runtime
        Console.WriteLine(dynamicValue.Name);       // Accesses property dynamically
        Console.WriteLine(dynamicValue.Calculate(5)); // Calls method dynamically
        
        // Runtime binding allows flexibility
        dynamicValue.NewProperty = "Dynamically added"; // Adding properties dynamically
        Console.WriteLine(dynamicValue.NewProperty);
    }
    
    static dynamic GetDynamicData()
    {
        return new { Name = "John", Calculate = new Func<int, int>(x => x * 2) };
    }
}
```

### ExpandoObject

`ExpandoObject` is a dynamic object that allows you to add and remove members (properties, methods, events) at runtime. It implements `IDynamicMetaObjectProvider` and is particularly useful for working with dynamic data structures.

```csharp
using System;
using System.Dynamic;

class Program
{
    static void Main()
    {
        // Create a fully dynamic object
        dynamic person = new ExpandoObject();
        
        // Add properties dynamically
        person.Name = "Alice";
        person.Age = 30;
        
        // Add methods dynamically using lambda expressions
        person.Introduce = new Action(() => 
        {
            Console.WriteLine($"Hi, I'm {person.Name}, {person.Age} years old");
        });
        
        person.Introduce(); // Calls the dynamically added method
        
        // Implement interfaces dynamically through events
        person.PropertyChanged += (sender, e) => 
        {
            Console.WriteLine($"Property {e} changed");
        };
        
        // Cast to IDictionary<string, object> for enumeration
        var dict = (System.Collections.Generic.IDictionary<string, object>)person;
        foreach (var property in dict)
        {
            Console.WriteLine($"{property.Key}: {property.Value}");
        }
    }
}
```

### Expression Trees

Expression trees represent code as data structures that can be examined, modified, and compiled into executable code at runtime. They enable meta-programming and are fundamental to LINQ providers.

```csharp
using System;
using System.Linq.Expressions;

class Program
{
    static void Main()
    {
        // Create expression tree: (x, y) => x + y * 2
        ParameterExpression paramX = Expression.Parameter(typeof(int), "x");
        ParameterExpression paramY = Expression.Parameter(typeof(int), "y");
        
        // Build the expression tree: y * 2
        ConstantExpression constant = Expression.Constant(2, typeof(int));
        BinaryExpression multiply = Expression.Multiply(paramY, constant);
        
        // Build final expression: x + (y * 2)
        BinaryExpression body = Expression.Add(paramX, multiply);
        
        // Create lambda expression
        Expression<Func<int, int, int>> lambda = 
            Expression.Lambda<Func<int, int, int>>(body, paramX, paramY);
        
        // Compile to executable delegate
        Func<int, int, int> compiled = lambda.Compile();
        
        // Execute
        int result = compiled(5, 3); // 5 + (3 * 2) = 11
        Console.WriteLine($"Result: {result}");
        
        // Advanced: Modify expression tree
        // Change multiplication to addition
        BinaryExpression newBody = Expression.Add(paramX, paramY); // x + y
        Expression<Func<int, int, int>> modifiedLambda = 
            Expression.Lambda<Func<int, int, int>>(newBody, paramX, paramY);
        
        Func<int, int, int> modifiedCompiled = modifiedLambda.Compile();
        Console.WriteLine($"Modified result: {modifiedCompiled(5, 3)}"); // 8
    }
}
```

## Why is Dynamic Programming important?

1. **Duck Typing Principle** - Enables "if it walks like a duck" behavior, allowing code to work with any object that has the required members, promoting interface flexibility over strict inheritance hierarchies.

2. **Open/Closed Principle (SOLID)** - Allows systems to be extended with new functionality at runtime without modifying existing code, supporting dynamic composition and plugin architectures.

3. **Runtime Adaptability** - Facilitates building systems that can adapt to changing requirements and data structures without recompilation, enhancing scalability in dynamic environments.

## Advanced Nuances

### 1. Custom Dynamic Object Implementation

Beyond `ExpandoObject`, you can create custom dynamic objects by implementing `IDynamicMetaObjectProvider`, giving you full control over dynamic dispatch:

```csharp
public class CustomDynamicObject : IDynamicMetaObjectProvider
{
    private readonly Dictionary<string, object> _properties = new();
    
    public DynamicMetaObject GetMetaObject(Expression parameter)
    {
        return new CustomMetaObject(parameter, BindingRestrictions.Empty, this);
    }
    
    // Custom meta object implementation would handle method calls and property access
}

// This enables highly specialized dynamic behavior beyond ExpandoObject's capabilities
```

### 2. Expression Tree Manipulation for Query Providers

Advanced usage involves building LINQ providers by intercepting and translating expression trees:

```csharp
// This is the foundation for Entity Framework, LINQ to SQL, etc.
public class CustomQueryProvider : IQueryProvider
{
    public IQueryable CreateQuery(Expression expression)
    {
        // Analyze expression tree and translate to target language (SQL, etc.)
        return new CustomQueryable(this, expression);
    }
    
    // Expression tree analysis enables complex query translation
}
```

### 3. Performance vs. Flexibility Trade-offs

Dynamic dispatch incurs runtime overhead compared to static binding. The DLR uses caching strategies (call site caching) to optimize repeated dynamic calls, but understanding when to use dynamic vs. static approaches is crucial for performance-critical applications.

## How this fits the Roadmap

Within the "Reflection and Metadata" section, Dynamic Programming serves as the bridge between compile-time and runtime type manipulation. It's a prerequisite for understanding:

- **Advanced Reflection Patterns** - Dynamic programming builds upon reflection capabilities but provides a more elegant API for runtime type manipulation
- **Aspect-Oriented Programming** - Expression trees enable runtime code generation for cross-cutting concerns
- **Dynamic Code Generation** - The foundation for runtime compilation and execution of code
- **Advanced Interop Scenarios** - Essential for working with COM, dynamic languages, and REST APIs

This knowledge unlocks more advanced topics like building custom DSLs (Domain Specific Languages), creating sophisticated ORM layers, and implementing complex runtime composition patterns that are hallmarks of senior C# expertise.