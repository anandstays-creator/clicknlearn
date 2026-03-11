# Class Structure in C#

## What is Class Structure?

Class Structure refers to the organization and design of classes in C#, including their members, relationships, and construction/destruction mechanisms. It encompasses how classes are built, initialized, and cleaned up, providing the blueprint for creating objects in object-oriented programming.

## How it Works in C#

### Constructor Chaining

Constructor chaining allows one constructor to call another within the same class, promoting code reuse and reducing duplication.

```csharp
public class Person
{
    public string Name { get; set; }
    public int Age { get; set; }
    public string Email { get; set; }

    // Default constructor chains to parameterized constructor
    public Person() : this("Unknown", 0, "unknown@example.com")
    {
    }

    // Primary constructor with all parameters
    public Person(string name, int age, string email)
    {
        Name = name;
        Age = age;
        Email = email;
    }

    // Partial parameter constructor chains to primary constructor
    public Person(string name, int age) : this(name, age, $"{name.ToLower()}@example.com")
    {
    }
}

// Usage example
var person1 = new Person(); // Uses default constructor chain
var person2 = new Person("Alice", 25); // Chains to primary constructor
```

### Destructors

Destructors (finalizers) are special methods that automatically clean up resources when an object is garbage collected. They're called automatically by the CLR.

```csharp
public class ResourceHandler
{
    private FileStream _fileStream;
    private bool _disposed = false;

    public ResourceHandler(string filePath)
    {
        _fileStream = File.OpenRead(filePath);
        Console.WriteLine("Resource acquired");
    }

    // Destructor (finalizer)
    ~ResourceHandler()
    {
        Dispose(false);
        Console.WriteLine("Destructor called");
    }

    // Proper disposal pattern
    public void Dispose()
    {
        Dispose(true);
        GC.SuppressFinalize(this); // Prevents finalizer call
    }

    protected virtual void Dispose(bool disposing)
    {
        if (!_disposed)
        {
            if (disposing)
            {
                // Managed resources
                _fileStream?.Dispose();
            }
            // Unmanaged resources cleanup
            _disposed = true;
        }
    }
}
```

### Static Constructors

Static constructors initialize static members of a class and are called automatically before any static members are accessed or any instances are created.

```csharp
public class ConfigurationManager
{
    private static readonly Dictionary<string, string> _settings;
    private static readonly DateTime _initializationTime;

    // Static constructor
    static ConfigurationManager()
    {
        _settings = new Dictionary<string, string>();
        _initializationTime = DateTime.Now;
        
        // Load configuration from file
        LoadSettings();
        Console.WriteLine("Static constructor executed");
    }

    private static void LoadSettings()
    {
        _settings["Database"] = "Server=localhost;Database=MyApp";
        _settings["ApiKey"] = "secret-key-123";
    }

    public static string GetSetting(string key)
    {
        return _settings.TryGetValue(key, out var value) ? value : null;
    }

    public static DateTime InitializationTime => _initializationTime;
}

// Usage - static constructor is called automatically
var dbSetting = ConfigurationManager.GetSetting("Database");
```

## Why is Class Structure Important?

1. **Maintainability** - Constructor chaining follows the DRY principle, reducing code duplication and making maintenance easier
2. **Resource Management** - Proper destructor implementation ensures efficient resource cleanup, following the Dispose pattern
3. **Reliability** - Static constructors provide controlled initialization, adhering to the Single Responsibility Principle

## Advanced Nuances

### Lazy Initialization with Static Constructors

Static constructors combined with `Lazy<T>` can provide thread-safe lazy initialization:

```csharp
public class LazySingleton
{
    private static readonly Lazy<LazySingleton> _instance = 
        new Lazy<LazySingleton>(() => new LazySingleton());
    
    private LazySingleton() { }
    
    public static LazySingleton Instance => _instance.Value;
}
```

### Exception Handling in Constructors

Constructors should handle exceptions carefully to avoid partially constructed objects:

```csharp
public class DatabaseConnection
{
    private readonly SqlConnection _connection;
    
    public DatabaseConnection(string connectionString)
    {
        try
        {
            _connection = new SqlConnection(connectionString);
            _connection.Open(); // Could throw exception
        }
        catch
        {
            _connection?.Dispose(); // Cleanup if initialization fails
            throw; // Re-throw to prevent partial construction
        }
    }
}
```

## How This Fits the Roadmap

Class Structure serves as the foundation for the "Object-Oriented Programming" section. It's a prerequisite for understanding more advanced concepts like inheritance, polymorphism, and design patterns. Mastering class structure unlocks:

- **Inheritance patterns** - Understanding base class construction
- **Disposable pattern** - Advanced resource management techniques
- **Singleton pattern** - Static constructor applications
- **Factory patterns** - Constructor design strategies

This knowledge enables progression toward more complex architectural patterns and prepares developers for enterprise-level application design.