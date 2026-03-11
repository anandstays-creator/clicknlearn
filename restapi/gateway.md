# API Gateway in C#

## **What is API Gateway?**

An API Gateway, also known as **API Proxy** or **API Mediator**, is a reverse proxy that acts as a single entry point for all client requests to backend microservices. Its core purpose is to handle cross-cutting concerns like routing, security, caching, and monitoring, solving the problem of microservice complexity by providing a unified interface. Instead of clients communicating directly with individual services, they interact with the gateway, which manages requests and responses centrally.

## **How it works in C#**

### **Management**
API Gateway management involves configuration, service discovery, and health monitoring. In C#, this typically uses configuration files, service registries, and health check endpoints.

```csharp
public class ApiGatewayManagementService
{
    private readonly IConfiguration _configuration;
    private readonly IServiceRegistry _serviceRegistry;

    public ApiGatewayManagementService(IConfiguration configuration, 
                                      IServiceRegistry serviceRegistry)
    {
        _configuration = configuration;
        _serviceRegistry = serviceRegistry;
    }

    // Health monitoring for backend services
    public async Task<bool> CheckBackendHealthAsync(string serviceName)
    {
        var serviceEndpoint = await _serviceRegistry.GetServiceEndpointAsync(serviceName);
        using var client = new HttpClient();
        try
        {
            var response = await client.GetAsync($"{serviceEndpoint}/health");
            return response.IsSuccessStatusCode;
        }
        catch
        {
            return false;
        }
    }

    // Dynamic configuration update
    public void UpdateRoutingConfiguration(RouteConfig newConfig)
    {
        // Typically persisted to distributed cache or database
        _configuration.GetSection("Routing").Bind(newConfig);
    }
}

// Route configuration model
public class RouteConfig
{
    public string RoutePath { get; set; }
    public string BackendService { get; set; }
    public List<string> AllowedMethods { get; set; }
}
```

### **Routing**
Routing determines how incoming requests are mapped to backend services based on URL patterns, HTTP methods, or headers.

```csharp
public class RouteResolver
{
    private readonly Dictionary<string, RouteConfig> _routes;

    public RouteResolver()
    {
        // In practice, load from configuration or service discovery
        _routes = new Dictionary<string, RouteConfig>
        {
            {
                "/api/users/*", 
                new RouteConfig 
                { 
                    BackendService = "user-service", 
                    AllowedMethods = new List<string> { "GET", "POST", "PUT" }
                }
            },
            {
                "/api/orders/*", 
                new RouteConfig 
                { 
                    BackendService = "order-service", 
                    AllowedMethods = new List<string> { "GET", "POST", "DELETE" }
                }
            }
        };
    }

    public RouteConfig ResolveRoute(string path, string method)
    {
        var matchingRoute = _routes
            .FirstOrDefault(r => path.StartsWith(r.Key.Replace("/*", "")))
            .Value;

        if (matchingRoute != null && !matchingRoute.AllowedMethods.Contains(method))
        {
            throw new MethodNotAllowedException($"Method {method} not allowed for {path}");
        }

        return matchingRoute;
    }
}

// Middleware for routing
public class RoutingMiddleware
{
    private readonly RequestDelegate _next;
    private readonly RouteResolver _routeResolver;

    public RoutingMiddleware(RequestDelegate next, RouteResolver routeResolver)
    {
        _next = next;
        _routeResolver = routeResolver;
    }

    public async Task InvokeAsync(HttpContext context)
    {
        var route = _routeResolver.ResolveRoute(context.Request.Path, context.Request.Method);
        
        if (route != null)
        {
            context.Items["BackendService"] = route.BackendService;
        }
        
        await _next(context);
    }
}
```

### **Load Balancing**
Load balancing distributes requests across multiple backend service instances to improve performance and reliability.

```csharp
public interface ILoadBalancer
{
    Task<string> GetNextInstanceAsync(string serviceName);
}

public class RoundRobinLoadBalancer : ILoadBalancer
{
    private readonly IServiceDiscovery _serviceDiscovery;
    private readonly ConcurrentDictionary<string, int> _currentIndex = new();

    public RoundRobinLoadBalancer(IServiceDiscovery serviceDiscovery)
    {
        _serviceDiscovery = serviceDiscovery;
    }

    public async Task<string> GetNextInstanceAsync(string serviceName)
    {
        var instances = await _serviceDiscovery.GetHealthyInstancesAsync(serviceName);
        
        if (instances.Count == 0)
            throw new ServiceUnavailableException($"No healthy instances for {serviceName}");

        var index = _currentIndex.AddOrUpdate(serviceName, 0, (key, old) => (old + 1) % instances.Count);
        return instances[index].Endpoint;
    }
}

public class LoadAwareLoadBalancer : ILoadBalancer
{
    private readonly IServiceDiscovery _serviceDiscovery;

    public LoadAwareLoadBalancer(IServiceDiscovery serviceDiscovery)
    {
        _serviceDiscovery = serviceDiscovery;
    }

    public async Task<string> GetNextInstanceAsync(string serviceName)
    {
        var instances = await _serviceDiscovery.GetHealthyInstancesAsync(serviceName);
        
        return instances
            .OrderBy(instance => instance.CurrentLoad) // Select least loaded instance
            .First()
            .Endpoint;
    }
}
```

### **Caching**
Caching stores responses to reduce backend load and improve response times for frequently accessed data.

```csharp
public class ResponseCacheMiddleware
{
    private readonly RequestDelegate _next;
    private readonly IDistributedCache _cache;
    private readonly ICacheKeyGenerator _keyGenerator;

    public ResponseCacheMiddleware(RequestDelegate next, 
                                  IDistributedCache cache,
                                  ICacheKeyGenerator keyGenerator)
    {
        _next = next;
        _cache = cache;
        _keyGenerator = keyGenerator;
    }

    public async Task InvokeAsync(HttpContext context)
    {
        var cacheKey = _keyGenerator.GenerateKey(context.Request);
        
        // Check cache first
        var cachedResponse = await _cache.GetStringAsync(cacheKey);
        if (cachedResponse != null)
        {
            await ReturnCachedResponse(context, cachedResponse);
            return;
        }

        // Capture response for caching
        var originalBodyStream = context.Response.Body;
        using var responseBody = new MemoryStream();
        context.Response.Body = responseBody;

        await _next(context);

        // Cache successful GET responses
        if (context.Request.Method == "GET" && context.Response.StatusCode == 200)
        {
            responseBody.Position = 0;
            var responseText = await new StreamReader(responseBody).ReadToEndAsync();
            
            await _cache.SetStringAsync(cacheKey, responseText, new DistributedCacheEntryOptions
            {
                AbsoluteExpirationRelativeToNow = TimeSpan.FromMinutes(5)
            });

            responseBody.Position = 0;
            await responseBody.CopyToAsync(originalBodyStream);
        }
    }

    private async Task ReturnCachedResponse(HttpContext context, string cachedResponse)
    {
        context.Response.ContentType = "application/json";
        context.Response.StatusCode = 200;
        await context.Response.WriteAsync(cachedResponse);
    }
}

public interface ICacheKeyGenerator
{
    string GenerateKey(HttpRequest request);
}
```

### **Security**
Security encompasses authentication, authorization, rate limiting, and threat protection.

```csharp
public class SecurityMiddleware
{
    private readonly RequestDelegate _next;
    private readonly IRateLimiter _rateLimiter;
    private readonly IJwtValidator _jwtValidator;

    public SecurityMiddleware(RequestDelegate next, 
                            IRateLimiter rateLimiter,
                            IJwtValidator jwtValidator)
    {
        _next = next;
        _rateLimiter = rateLimiter;
        _jwtValidator = jwtValidator;
    }

    public async Task InvokeAsync(HttpContext context)
    {
        // Rate limiting
        var clientIp = context.Connection.RemoteIpAddress.ToString();
        if (!await _rateLimiter.IsAllowedAsync(clientIp))
        {
            context.Response.StatusCode = 429;
            await context.Response.WriteAsync("Rate limit exceeded");
            return;
        }

        // JWT validation for protected endpoints
        if (IsProtectedEndpoint(context.Request.Path))
        {
            var token = context.Request.Headers["Authorization"].FirstOrDefault()?.Replace("Bearer ", "");
            
            if (string.IsNullOrEmpty(token) || !await _jwtValidator.ValidateTokenAsync(token))
            {
                context.Response.StatusCode = 401;
                await context.Response.WriteAsync("Unauthorized");
                return;
            }
        }

        await _next(context);
    }

    private bool IsProtectedEndpoint(string path) => path.StartsWith("/api/protected/");
}

public interface IRateLimiter
{
    Task<bool> IsAllowedAsync(string clientIdentifier);
}

public class TokenBucketRateLimiter : IRateLimiter
{
    private readonly ConcurrentDictionary<string, TokenBucket> _buckets = new();

    public async Task<bool> IsAllowedAsync(string clientIdentifier)
    {
        var bucket = _buckets.GetOrAdd(clientIdentifier, 
            key => new TokenBucket(100, 10)); // 100 tokens, 10 refill per second

        return await bucket.TryConsumeAsync(1);
    }
}
```

### **Analytics**
Analytics involves collecting metrics, logging, and monitoring API usage patterns.

```csharp
public class AnalyticsMiddleware
{
    private readonly RequestDelegate _next;
    private readonly IMetricsCollector _metricsCollector;
    private readonly ILogger<AnalyticsMiddleware> _logger;

    public AnalyticsMiddleware(RequestDelegate next, 
                             IMetricsCollector metricsCollector,
                             ILogger<AnalyticsMiddleware> logger)
    {
        _next = next;
        _metricsCollector = metricsCollector;
        _logger = logger;
    }

    public async Task InvokeAsync(HttpContext context)
    {
        var stopwatch = Stopwatch.StartNew();
        
        try
        {
            await _next(context);
            stopwatch.Stop();

            // Collect metrics
            await _metricsCollector.RecordRequestAsync(new RequestMetric
            {
                Path = context.Request.Path,
                Method = context.Request.Method,
                StatusCode = context.Response.StatusCode,
                ResponseTimeMs = stopwatch.ElapsedMilliseconds,
                Timestamp = DateTime.UtcNow
            });

            _logger.LogInformation("Request {Method} {Path} completed in {ElapsedMs}ms", 
                context.Request.Method, context.Request.Path, stopwatch.ElapsedMilliseconds);
        }
        catch (Exception ex)
        {
            stopwatch.Stop();
            await _metricsCollector.RecordErrorAsync(context.Request.Path, ex);
            _logger.LogError(ex, "Request {Method} {Path} failed", 
                context.Request.Method, context.Request.Path);
            throw;
        }
    }
}

public interface IMetricsCollector
{
    Task RecordRequestAsync(RequestMetric metric);
    Task RecordErrorAsync(string path, Exception exception);
}

public class RequestMetric
{
    public string Path { get; set; }
    public string Method { get; set; }
    public int StatusCode { get; set; }
    public long ResponseTimeMs { get; set; }
    public DateTime Timestamp { get; set; }
}
```

## **Why is API Gateway important?**

1. **Single Responsibility Principle (SOLID)** - The gateway centralizes cross-cutting concerns, allowing backend services to focus solely on business logic.

2. **Facade Pattern** - It provides a simplified interface to complex microservice architectures, reducing client-side complexity.

3. **Scalability** - Enables horizontal scaling through load balancing and caching, following distributed systems best practices.

## **Advanced Nuances**

### **Circuit Breaker Pattern Integration**
Advanced API Gateways implement circuit breakers to prevent cascading failures:

```csharp
public class CircuitBreakerPolicy
{
    private int _failureCount = 0;
    private DateTime _lastFailureTime;
    private readonly TimeSpan _timeout;
    private readonly int _failureThreshold;

    public async Task<T> ExecuteAsync<T>(Func<Task<T>> action)
    {
        if (IsOpen)
        {
            if (DateTime.UtcNow - _lastFailureTime < _timeout)
                throw new CircuitBreakerOpenException();
            
            // Try to close circuit
            _failureCount = 0;
        }

        try
        {
            var result = await action();
            _failureCount = 0; // Reset on success
            return result;
        }
        catch
        {
            _failureCount++;
            _lastFailureTime = DateTime.UtcNow;
            throw;
        }
    }

    private bool IsOpen => _failureCount >= _failureThreshold;
}
```

### **Dynamic Routing with Feature Flags**
Senior developers implement feature-flag based routing for canary deployments:

```csharp
public class FeatureAwareRouter
{
    public async Task<RouteConfig> GetRouteAsync(string path, string userId)
    {
        var baseRoute = await _routeResolver.ResolveRoute(path);
        
        // Route 10% of users to new service version
        if (await _featureManager.IsEnabledAsync("new-service-version", userId))
        {
            return baseRoute with { BackendService = "new-service-v2" };
        }
        
        return baseRoute;
    }
}
```

## **How this fits the Roadmap**

Within the "Advanced Topics" section, API Gateway serves as a **fundamental building block** that unlocks several advanced concepts:

**Prerequisites**: This topic assumes knowledge of HTTP protocols, middleware patterns, dependency injection, and basic microservices.

**Unlocks**: 
- **Service Mesh Integration** (Istio, Linkerd) - API Gateways often evolve into service mesh control planes
- **Event-Driven Architectures** - Gateway patterns extend to message brokering and event routing
- **Advanced Monitoring** - Distributed tracing and observability platforms build upon gateway analytics

API Gateway sits at the intersection of infrastructure and application logic, making it essential for architects designing scalable cloud-native systems.