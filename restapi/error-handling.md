# Error Handling in C#

## What is Error Handling?

Error handling, also known as exception handling, is a systematic approach to anticipating, detecting, and responding to errors that occur during program execution. Its core purpose is to prevent application crashes, maintain data integrity, and provide meaningful feedback when unexpected situations arise. Error handling solves the fundamental problem of unpredictability in software systems—it ensures that applications can gracefully recover from failures rather than terminating abruptly.

## How it works in C#

### Consistent Responses

**Explanation:** Consistent responses ensure that your application handles similar errors in the same way across the entire codebase. This creates predictable behavior and makes the API easier to consume. In service-based architectures, this typically involves returning standardized response objects regardless of where the error originates.

```csharp
public class ApiResponse<T>
{
    public bool Success { get; set; }
    public string Message { get; set; }
    public string ErrorCode { get; set; }
    public T Data { get; set; }
    
    public static ApiResponse<T> CreateSuccess(T data) => new ApiResponse<T> 
    { 
        Success = true, 
        Data = data 
    };
    
    public static ApiResponse<T> CreateError(string message, string errorCode) => new ApiResponse<T> 
    { 
        Success = false, 
        Message = message, 
        ErrorCode = errorCode 
    };
}

public class UserService
{
    public ApiResponse<User> GetUser(int userId)
    {
        try
        {
            var user = _repository.GetUser(userId);
            if (user == null)
            {
                // Consistent error response for "not found" scenario
                return ApiResponse<User>.CreateError("User not found", "USER_NOT_FOUND");
            }
            return ApiResponse<User>.CreateSuccess(user);
        }
        catch (Exception ex)
        {
            // Consistent response for unexpected errors
            return ApiResponse<User>.CreateError("Internal server error", "INTERNAL_ERROR");
        }
    }
}
```

### Custom Codes

**Explanation:** Custom error codes provide machine-readable identifiers for specific error conditions, allowing clients to programmatically handle different types of failures. Unlike exception messages which can change, error codes remain stable across application versions.

```csharp
public static class ErrorCodes
{
    // Domain-specific error codes
    public const string UserNotFound = "USER_NOT_FOUND";
    public const string InvalidEmail = "INVALID_EMAIL_FORMAT";
    public const string InsufficientPermissions = "INSUFFICIENT_PERMISSIONS";
    public const string PaymentFailed = "PAYMENT_PROCESSING_FAILED";
}

public class CustomException : Exception
{
    public string ErrorCode { get; }
    public object AdditionalData { get; }
    
    public CustomException(string errorCode, string message, object additionalData = null) 
        : base(message)
    {
        ErrorCode = errorCode;
        AdditionalData = additionalData;
    }
}

public class PaymentService
{
    public void ProcessPayment(PaymentRequest request)
    {
        if (!IsValidCardNumber(request.CardNumber))
        {
            throw new CustomException(
                ErrorCodes.PaymentFailed, 
                "Invalid credit card number",
                new { CardLastFour = request.CardNumber?.Substring(Math.Max(0, request.CardNumber.Length - 4)) }
            );
        }
        
        // Payment processing logic...
    }
}
```

### Detailed Messages

**Explanation:** Detailed error messages provide context about what went wrong, why it happened, and potentially how to fix it. They should be informative for developers while avoiding exposure of sensitive information.

```csharp
public class DetailedException : Exception
{
    public string Context { get; }
    public string SuggestedAction { get; }
    public DateTime Timestamp { get; }
    
    public DetailedException(string message, string context, string suggestedAction = null) 
        : base(message)
    {
        Context = context;
        SuggestedAction = suggestedAction ?? "Please contact support if the issue persists";
        Timestamp = DateTime.UtcNow;
    }
    
    public override string ToString()
    {
        return $"[{Timestamp:yyyy-MM-dd HH:mm:ss}] Error: {Message}. Context: {Context}. Action: {SuggestedAction}";
    }
}

public class FileProcessor
{
    public void ProcessFile(string filePath)
    {
        if (!File.Exists(filePath))
        {
            throw new DetailedException(
                "File not accessible",
                $"Attempted to access file at: {filePath}",
                "Verify the file path exists and the application has read permissions"
            );
        }
        
        try
        {
            var content = File.ReadAllText(filePath);
            // Process file content
        }
        catch (IOException ioEx)
        {
            throw new DetailedException(
                "File read operation failed",
                $"File: {filePath}, Size: {new FileInfo(filePath).Length} bytes",
                "Check if file is locked by another process or disk space is available"
            );
        }
    }
}
```

### Logging

**Explanation:** Logging captures error information for debugging, monitoring, and auditing purposes. It should include sufficient context to diagnose issues without overwhelming the log storage with redundant information.

```csharp
public interface ILogger
{
    void LogError(string message, Exception ex, object context = null);
}

public class StructuredLogger : ILogger
{
    private readonly ILogger _underlyingLogger;
    
    public void LogError(string message, Exception ex, object context = null)
    {
        var logEntry = new
        {
            Timestamp = DateTime.UtcNow,
            Message = message,
            ExceptionMessage = ex.Message,
            ExceptionType = ex.GetType().Name,
            StackTrace = ex.StackTrace,
            Context = context,
            MachineName = Environment.MachineName,
            ProcessId = Environment.ProcessId
        };
        
        _underlyingLogger.LogError("Structured Error: {@LogEntry}", logEntry);
    }
}

public class OrderService
{
    private readonly ILogger _logger;
    
    public void CreateOrder(Order order)
    {
        try
        {
            _validator.Validate(order);
            _repository.Save(order);
        }
        catch (ValidationException ex)
        {
            // Log validation errors with context
            _logger.LogError("Order validation failed", ex, new 
            { 
                OrderId = order.Id, 
                CustomerId = order.CustomerId,
                ValidationErrors = ex.Errors 
            });
            throw;
        }
        catch (DbException ex)
        {
            // Log database errors with connection info
            _logger.LogError("Database operation failed", ex, new 
            { 
                OrderId = order.Id,
                Database = "OrdersDB",
                Operation = "Insert" 
            });
            throw new CustomException("DATABASE_ERROR", "Failed to save order");
        }
    }
}
```

### Resilience

**Explanation:** Resilience refers to the ability of a system to continue operating despite failures in individual components. This involves designing error handling that contains failures and prevents cascading crashes.

```csharp
public class ResilientService
{
    private readonly ICircuitBreaker _circuitBreaker;
    private readonly ILogger _logger;
    
    public async Task<Data> GetDataWithResilienceAsync()
    {
        // Check circuit breaker state before attempting operation
        if (_circuitBreaker.State == CircuitBreakerState.Open)
        {
            throw new ServiceUnavailableException("Service temporarily unavailable");
        }
        
        try
        {
            var result = await _circuitBreaker.ExecuteAsync(async () => 
            {
                return await _externalService.GetDataAsync();
            });
            
            return result;
        }
        catch (Exception ex)
        {
            _logger.LogError("Resilience strategy failed", ex);
            // Fallback to cached data or default values
            return await GetCachedDataAsync();
        }
    }
}

public class CircuitBreaker
{
    public CircuitBreakerState State { get; private set; }
    private int _failureCount = 0;
    private readonly int _failureThreshold;
    private DateTime _lastFailureTime;
    
    public async Task<T> ExecuteAsync<T>(Func<Task<T>> action)
    {
        if (State == CircuitBreakerState.Open)
        {
            if (DateTime.UtcNow - _lastFailureTime > TimeSpan.FromMinutes(1))
            {
                State = CircuitBreakerState.HalfOpen; // Attempt recovery
            }
            else
            {
                throw new CircuitBreakerOpenException();
            }
        }
        
        try
        {
            var result = await action();
            _failureCount = 0; // Reset on success
            State = CircuitBreakerState.Closed;
            return result;
        }
        catch (Exception ex)
        {
            _failureCount++;
            _lastFailureTime = DateTime.UtcNow;
            
            if (_failureCount >= _failureThreshold)
            {
                State = CircuitBreakerState.Open;
            }
            throw;
        }
    }
}

public enum CircuitBreakerState { Closed, Open, HalfOpen }
```

### Retry Logic

**Explanation:** Retry logic automatically reattempts failed operations, particularly useful for transient errors like network timeouts or database deadlocks. It should include strategies like exponential backoff to avoid overwhelming the target system.

```csharp
public class RetryPolicy
{
    private readonly int _maxRetries;
    private readonly TimeSpan _initialDelay;
    private readonly double _backoffMultiplier;
    
    public async Task<T> ExecuteWithRetryAsync<T>(Func<Task<T>> operation, 
        Func<Exception, bool> shouldRetry = null)
    {
        var exceptions = new List<Exception>();
        var currentDelay = _initialDelay;
        
        for (int attempt = 0; attempt <= _maxRetries; attempt++)
        {
            try
            {
                return await operation();
            }
            catch (Exception ex) when (ShouldRetryException(ex, shouldRetry))
            {
                exceptions.Add(ex);
                
                if (attempt == _maxRetries) // Last attempt failed
                {
                    throw new AggregateException(
                        $"Operation failed after {_maxRetries + 1} attempts", 
                        exceptions
                    );
                }
                
                // Exponential backoff with jitter to avoid thundering herd
                var jitter = TimeSpan.FromMilliseconds(new Random().Next(0, 100));
                await Task.Delay(currentDelay + jitter);
                currentDelay = TimeSpan.FromTicks((long)(currentDelay.Ticks * _backoffMultiplier));
            }
        }
        
        throw new InvalidOperationException("Retry logic error");
    }
    
    private bool ShouldRetryException(Exception ex, Func<Exception, bool> customPredicate)
    {
        // Retry on transient errors
        if (ex is TimeoutException || ex is HttpRequestException) return true;
        if (ex is SqlException sqlEx && IsTransientSqlError(sqlEx)) return true;
        
        return customPredicate?.Invoke(ex) ?? false;
    }
}

public class DatabaseService
{
    private readonly RetryPolicy _retryPolicy = new RetryPolicy(
        maxRetries: 3, 
        initialDelay: TimeSpan.FromSeconds(1), 
        backoffMultiplier: 2.0
    );
    
    public async Task<User> GetUserAsync(int userId)
    {
        return await _retryPolicy.ExecuteWithRetryAsync(async () =>
        {
            using var connection = new SqlConnection(_connectionString);
            return await connection.QuerySingleAsync<User>(
                "SELECT * FROM Users WHERE Id = @UserId", 
                new { UserId = userId }
            );
        }, 
        shouldRetry: ex => ex is SqlException); // Custom retry condition
    }
}
```

## Why is Error Handling Important?

1. **Fault Tolerance (Resilience Pattern):** Proper error handling enables systems to continue operating despite component failures, implementing the resilience engineering principle that systems should degrade gracefully rather than fail completely.

2. **Maintainability (Single Responsibility Principle):** Centralized error handling separates business logic from error recovery code, adhering to SOLID principles by ensuring each component has a single reason to change.

3. **Operational Excellence (Observability Principle):** Comprehensive error handling with logging and monitoring provides the telemetry needed for rapid incident response and continuous improvement, supporting scalable system operations.

## Advanced Nuances

### 1. Exception Filters for Conditional Catch Blocks

```csharp
public class AdvancedErrorHandling
{
    public void ProcessWithFilters()
    {
        try
        {
            RiskyOperation();
        }
        // Exception filters allow conditional catching without wrapping
        catch (HttpRequestException ex) when (ex.StatusCode == System.Net.HttpStatusCode.TooManyRequests)
        {
            // Handle rate limiting specifically
            _rateLimiter.BackOff();
            throw;
        }
        catch (HttpRequestException ex) when (ex.StatusCode == System.Net.HttpStatusCode.NotFound)
        {
            // Handle not found differently
            _cache.Invalidate(ex.RequestUri);
            throw;
        }
        catch (Exception ex) when (IsTransientError(ex))
        {
            // Filter-based retry logic
            _retryPolicy.Retry(() => RiskyOperation());
        }
    }
    
    private bool IsTransientError(Exception ex) => 
        ex is TimeoutException || 
        (ex is SqlException sqlEx && sqlEx.Number == -2); // Timeout error code
}
```

### 2. Functional Error Handling with Result Pattern

```csharp
public class Result<T>
{
    public bool IsSuccess { get; }
    public T Value { get; }
    public Error Error { get; }
    
    private Result(T value) { Value = value; IsSuccess = true; }
    private Result(Error error) { Error = error; IsSuccess = false; }
    
    public static Result<T> Success(T value) => new Result<T>(value);
    public static Result<T> Failure(Error error) => new Result<T>(error);
    
    public Result<TResult> Bind<TResult>(Func<T, Result<TResult>> function) =>
        IsSuccess ? function(Value) : Result<TResult>.Failure(Error);
}

public class FunctionalErrorHandling
{
    public Result<Order> ProcessOrder(OrderRequest request)
    {
        return ValidateRequest(request)
            .Bind(ValidateInventory)
            .Bind(ProcessPayment)
            .Bind(SaveOrder);
    }
    
    // No try-catch blocks - errors flow through the Result chain
    private Result<OrderRequest> ValidateRequest(OrderRequest request)
    {
        if (request.Quantity <= 0)
            return Result<OrderRequest>.Failure(Error.InvalidQuantity);
        
        return Result<OrderRequest>.Success(request);
    }
}
```

### 3. Strategic Exception Wrapping and Unwrapping

```csharp
public class ExceptionWrapping
{
    public void OuterMethod()
    {
        try
        {
            MiddlewareMethod();
        }
        catch (Exception ex)
        {
            // Unwrap aggregated exceptions to find root cause
            var rootCause = ex is AggregateException aggEx ? 
                aggEx.GetBaseException() : ex;
                
            // Preserve stack trace when re-throwing
            ExceptionDispatchInfo.Capture(rootCause).Throw();
        }
    }
    
    private void MiddlewareMethod()
    {
        try
        {
            CoreBusinessMethod();
        }
        catch (SpecificBusinessException ex)
        {
            // Wrap with additional context but preserve original
            throw new OperationFailedException(
                "Business operation failed during processing", 
                ex // Inner exception preserved
            );
        }
    }
}
```

## How this fits the Roadmap

Within the "Security and Best Practices" section of the Advanced C# Mastery roadmap, Error Handling serves as a foundational pillar that enables more advanced security concepts. It's a prerequisite for implementing robust **Security Frameworks** (as proper error handling prevents information leakage that could aid attackers) and enables **Defensive Programming** techniques.

This concept unlocks more advanced topics like:
- **Distributed Tracing**: Building on structured logging to track errors across microservices
- **Circuit Breaker Patterns**: Advanced resilience building upon basic retry logic
- **Aspect-Oriented Error Handling**: Using interceptors for cross-cutting concern management
- **Security Incident Response**: Leveraging detailed error telemetry for security monitoring

Error handling forms the bedrock upon which secure, observable, and maintainable enterprise applications are built, making it essential for any developer progressing toward C# mastery.