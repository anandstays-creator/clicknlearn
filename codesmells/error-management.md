# Error Management in C#

## What is Error Management?

Error Management, commonly referred to as **Exception Handling**, is the systematic approach to detecting, responding to, and recovering from runtime errors in C# applications. Its core purpose is to gracefully handle exceptional conditions that disrupt normal program flow, preventing application crashes and maintaining system stability. Error management solves the problem of unpredictable runtime failures by providing a structured way to separate error-handling logic from business logic, ensuring robust and maintainable code.

## How it works in C#

### Try/Catch/Finally

The `try-catch-finally` block forms the foundation of C# exception handling. The `try` block contains code that might throw exceptions, `catch` blocks handle specific exceptions, and the `finally` block contains cleanup code that always executes.

```csharp
public class FileProcessor
{
    public void ProcessFile(string filePath)
    {
        StreamReader reader = null;
        try
        {
            // Code that might throw exceptions
            reader = new StreamReader(filePath);
            string content = reader.ReadToEnd();
            Console.WriteLine($"File content: {content}");
        }
        catch (FileNotFoundException ex)
        {
            // Handle specific exception type
            Console.WriteLine($"File not found: {ex.FileName}");
        }
        catch (IOException ex)
        {
            // Handle IO-related exceptions
            Console.WriteLine($"IO error: {ex.Message}");
        }
        catch (Exception ex)
        {
            // Catch-all for any other exceptions
            Console.WriteLine($"Unexpected error: {ex.Message}");
        }
        finally
        {
            // Always execute cleanup code
            reader?.Dispose();
            Console.WriteLine("Cleanup completed");
        }
    }
}
```

### Custom Exceptions

Custom exceptions allow you to create domain-specific error types that convey meaningful information about application-specific failure conditions. They should inherit from `Exception` or its derived classes.

```csharp
// Custom exception for business rule violations
public class InvalidOrderException : Exception
{
    public string OrderId { get; }
    public string ViolatedRule { get; }
    
    public InvalidOrderException(string orderId, string violatedRule) 
        : base($"Order {orderId} violated rule: {violatedRule}")
    {
        OrderId = orderId;
        ViolatedRule = violatedRule;
    }
    
    // Advanced: Support for serialization
    protected InvalidOrderException(
        System.Runtime.Serialization.SerializationInfo info,
        System.Runtime.Serialization.StreamingContext context) : base(info, context)
    {
        OrderId = info.GetString(nameof(OrderId));
        ViolatedRule = info.GetString(nameof(ViolatedRule));
    }
}

public class OrderValidator
{
    public void ValidateOrder(Order order)
    {
        if (order.TotalAmount < 0)
        {
            // Throw custom exception with domain-specific context
            throw new InvalidOrderException(order.Id, "Total amount cannot be negative");
        }
        
        if (order.Items.Count == 0)
        {
            throw new InvalidOrderException(order.Id, "Order must contain at least one item");
        }
    }
}
```

### Filtering

Exception filtering allows you to catch exceptions based on conditions beyond just the exception type, using the `when` keyword. This enables more precise exception handling without unwinding the stack.

```csharp
public class DatabaseService
{
    private int retryCount = 0;
    private const int MaxRetries = 3;
    
    public void ExecuteQuery(string query)
    {
        try
        {
            // Simulate database operation
            if (DateTime.Now.Second % 3 == 0) // Simulate intermittent failure
                throw new SqlException { Number = 1205 }; // Deadlock
            else if (DateTime.Now.Second % 5 == 0)
                throw new SqlException { Number = -2 }; // Timeout
            
            Console.WriteLine("Query executed successfully");
            retryCount = 0; // Reset on success
        }
        catch (SqlException ex) when (ex.Number == 1205 && retryCount < MaxRetries)
        {
            // Filter for deadlock errors with retry logic
            retryCount++;
            Console.WriteLine($"Deadlock detected, retrying... ({retryCount}/{MaxRetries})");
            System.Threading.Thread.Sleep(100); // Brief pause
            ExecuteQuery(query); // Retry
        }
        catch (SqlException ex) when (ex.Number == -2)
        {
            // Filter for timeout errors - no retry
            Console.WriteLine("Query timeout - requires manual intervention");
            throw; // Re-throw to caller
        }
        catch (SqlException ex) when (ShouldRetry(ex))
        {
            // Complex filtering using helper method
            HandleRetryableError(ex);
        }
    }
    
    private bool ShouldRetry(SqlException ex)
    {
        // Define retryable error codes
        int[] retryableErrors = { 1205, 1222, 8651 };
        return retryableErrors.Contains(ex.Number) && retryCount < MaxRetries;
    }
    
    private void HandleRetryableError(SqlException ex)
    {
        retryCount++;
        Console.WriteLine($"Retryable error {ex.Number}, attempt {retryCount}");
        System.Threading.Thread.Sleep(100);
        ExecuteQuery("SELECT * FROM Users");
    }
}
```

## Why is Error Management important?

1. **Resilience Principle**: Enables applications to gracefully handle failures and continue operating, ensuring higher system availability and reliability.

2. **Separation of Concerns (SOLID)**: Isolates error-handling logic from business logic, making code more maintainable and adhering to the Single Responsibility Principle.

3. **Debugging Efficiency**: Provides structured error information and context, significantly reducing troubleshooting time through meaningful exception messages and stack traces.

## Advanced Nuances

### Exception Filter Performance

Exception filters execute **before** stack unwinding, which has important implications:

```csharp
public void DemonstranteFilterTiming()
{
    try
    {
        ThrowException();
    }
    catch (Exception ex) when (LogAndFilter(ex)) // Filter runs BEFORE stack unwind
    {
        // This block only executes if filter returns true
    }
}

private bool LogAndFilter(Exception ex)
{
    Console.WriteLine("Filter executing - stack still intact");
    // Stack trace includes ThrowException method at this point
    return ex is InvalidOperationException;
}
```

### AggregateException for Parallel Processing

When working with parallel operations, multiple exceptions can be wrapped in `AggregateException`:

```csharp
public void ProcessMultipleFiles(string[] files)
{
    try
    {
        Parallel.ForEach(files, file =>
        {
            if (file.Contains("invalid"))
                throw new ArgumentException($"Invalid file: {file}");
            // Process file...
        });
    }
    catch (AggregateException ae)
    {
        // Handle multiple exceptions from parallel operations
        ae.Handle(ex =>
        {
            if (ex is ArgumentException)
            {
                Console.WriteLine($"Skipping invalid file: {ex.Message}");
                return true; // Exception handled
            }
            return false; // Exception not handled - will be rethrown
        });
    }
}
```

### Exception DispatchInfo for Re-throwing

Preserving stack traces when re-throwing exceptions across boundaries:

```csharp
public void ExceptionPreservationExample()
{
    ExceptionDispatchInfo capturedException = null;
    
    try
    {
        MethodThatThrows();
    }
    catch (Exception ex)
    {
        // Capture exception without losing stack trace
        capturedException = ExceptionDispatchInfo.Capture(ex);
    }
    
    // Later, re-throw with original stack trace
    capturedException?.Throw();
}
```

## How this fits the Roadmap

Error Management serves as the **foundation** of the "Exception Handling" section in the Advanced C# Mastery roadmap. It's a prerequisite for more advanced topics like:

- **Exception Handling Patterns**: Building on basic try-catch to implement patterns like Circuit Breaker, Retry, and Fallback
- **Global Exception Handling**: Applying error management principles at application level with global handlers
- **Performance Considerations**: Understanding exception overhead and when to use alternative error handling strategies
- **Async Exception Handling**: Extending these concepts to asynchronous programming paradigms

Mastering these fundamentals unlocks the ability to design robust, enterprise-level error handling strategies that can gracefully handle complex failure scenarios in distributed systems.