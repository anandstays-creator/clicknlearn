# Data Types in C#

## What is Data Types?

In C#, data types are fundamental constructs that define the nature of data that variables can hold — specifying the range of possible values, the operations that can be performed on them, and how they're stored in memory. Common aliases exist for many types (e.g., `int` for `Int32`, `string` for `String`). The core purpose is to enforce type safety, enabling the compiler to catch errors early, optimize memory allocation, and provide IntelliSense support. This solves the problem of runtime type errors and ensures predictable program behavior.

## How it works in C#

### Value Types
Value types store data directly in their own memory allocation. They're typically stored on the stack (though can be boxed) and include primitive types, structs, and enums. When assigned or passed, a copy of the value is created.

```csharp
// Primitive value types
int number = 42;           // 32-bit integer
double pi = 3.14159;       // Double-precision floating point
bool isValid = true;       // Boolean value
char letter = 'A';         // Single Unicode character

// Assignment creates a copy
int original = 10;
int copy = original;       // copy gets value 10
original = 20;            // copy remains 10
Console.WriteLine(copy);   // Output: 10
```

### Reference Types
Reference types store a reference to the actual data location (heap memory). The variable contains an address pointer rather than the data itself. Classes, arrays, delegates, and strings are reference types.

```csharp
// Class example
public class Person
{
    public string Name { get; set; }
}

Person person1 = new Person { Name = "Alice" };
Person person2 = person1;          // Both reference same object
person2.Name = "Bob";              // Modifies shared object
Console.WriteLine(person1.Name);   // Output: Bob (both affected)

// String special case (immutable reference type)
string str1 = "Hello";
string str2 = str1;                // References same string
str2 = "World";                    // Creates new string object
Console.WriteLine(str1);           // Output: Hello (unchanged)
```

### Generics
Generics enable type-safe, reusable code by allowing types to be parameters. They provide compile-time type safety while avoiding boxing/unboxing overhead.

```csharp
// Generic class
public class Repository<T> where T : class
{
    private List<T> items = new List<T>();
    
    public void Add(T item) => items.Add(item);
    public T Get(int index) => items[index];
}

// Usage with different types
Repository<string> stringRepo = new Repository<string>();
stringRepo.Add("Generic item");

Repository<Person> personRepo = new Repository<Person>();
personRepo.Add(new Person { Name = "John" });

// Generic method
public T Process<T>(T input) where T : IComparable
{
    // Type-safe operations on T
    return input;
}
```

### Nullable Types
Nullable types allow value types to represent null values using the `Nullable<T>` struct or the `?` suffix syntax. This is essential for database interactions and optional value scenarios.

```csharp
// Nullable value types
int? nullableInt = null;           // Equivalent to Nullable<int>
DateTime? nullableDate = null;     // Nullable DateTime

// Checking and accessing values
if (nullableInt.HasValue)
{
    int actualValue = nullableInt.Value;  // Access when not null
}

// Null-coalescing operator for defaults
int safeValue = nullableInt ?? 42;        // Use 42 if null

// Null-conditional operator with nullable
int? length = nullableInt?.ToString().Length;  // Returns null if nullableInt is null
```

### Enumerations
Enums are value types that define a set of named constants, making code more readable and maintainable. Underlying types can be specified (default is int).

```csharp
// Basic enum
public enum OrderStatus
{
    Pending,        // 0
    Processing,     // 1  
    Shipped,        // 2
    Delivered       // 3
}

// Enum with explicit values
public enum HttpStatusCode : short
{
    OK = 200,
    NotFound = 404,
    ServerError = 500
}

// Usage
OrderStatus status = OrderStatus.Processing;
if (status == OrderStatus.Shipped)
{
    Console.WriteLine("Order is on its way");
}

// Flags enum for bitwise combinations
[Flags]
public enum Permissions
{
    None = 0,
    Read = 1,
    Write = 2,
    Execute = 4,
    ReadWrite = Read | Write
}
```

### Structures
Structs are lightweight value types typically used for small data structures. They're stack-allocated and don't support inheritance.

```csharp
// Struct definition
public struct Point
{
    public int X { get; set; }
    public int Y { get; set; }
    
    public Point(int x, int y)
    {
        X = x;
        Y = y;
    }
    
    public double DistanceToOrigin() => Math.Sqrt(X * X + Y * Y);
}

// Usage
Point p1 = new Point(10, 20);
Point p2 = p1;              // Copy of value
p2.X = 30;                  // p1 unaffected

// Readonly struct for immutability
public readonly struct ImmutablePoint
{
    public int X { get; }
    public int Y { get; }
    
    public ImmutablePoint(int x, int y) => (X, Y) = (x, y);
}
```

## Why is Data Types important?

1. **Type Safety (Fail-Fast Principle)**: Compile-time type checking prevents runtime errors, catching issues early in the development cycle when they're cheapest to fix.

2. **Memory Optimization (Resource Management)**: Proper type selection enables efficient memory allocation — value types avoid heap overhead while reference types enable shared data.

3. **Code Clarity (Self-Documenting Code)**: Explicit typing makes code intentions clear, improving maintainability and reducing cognitive load for developers.

## Advanced Nuances

### Boxing/Unboxing Performance Implications
When value types are treated as objects (`object`), they undergo boxing — being wrapped in a reference type wrapper. This creates heap allocation and performance overhead.

```csharp
int number = 42;
object boxed = number;           // Boxing: heap allocation
int unboxed = (int)boxed;        // Unboxing: type check + extraction

// Generic alternative avoids boxing
public void Process<T>(T value) where T : struct
{
    // No boxing occurs with generics
}
```

### Nullable Value Type Underlying Representation
Nullable types are actually structs with a boolean flag tracking null state, demonstrating how language features build on fundamental types.

```csharp
// Nullable<int> is equivalent to:
public struct NullableInt
{
    private int value;
    private bool hasValue;
    
    public NullableInt(int v) 
    { 
        value = v; 
        hasValue = true; 
    }
    
    public bool HasValue => hasValue;
    public int Value => hasValue ? value : throw new InvalidOperationException();
}
```

### Ref Structs and Stack-Only Types
Advanced struct usage with `ref struct` ensures the type never escapes to the heap, enabling high-performance scenarios while imposing usage restrictions.

```csharp
public ref struct StackOnlyBuffer
{
    private Span<byte> buffer;
    
    public StackOnlyBuffer(Span<byte> data) => buffer = data;
    
    // Cannot be boxed or used in async methods
    // Ensures lifetime stays on stack
}
```

## How this fits the Roadmap

Within the "Core Concepts" section, Data Types serves as the foundational layer upon which all other C# concepts are built. It's a prerequisite for understanding **Memory Management** (stack vs. heap, garbage collection), **Type Systems** (inheritance, interfaces), and **Performance Optimization** (avoiding boxing, struct usage). 

Mastery of data types unlocks advanced topics like **Generic Constraints**, **Pattern Matching**, **Span<T> and Memory Management**, and **High-Performance Scenarios**. It's the essential vocabulary that enables meaningful discussion about more complex language features and architectural patterns.