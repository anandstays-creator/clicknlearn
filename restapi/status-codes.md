# HTTP Status Codes in C#

## What is HTTP Status Codes?

HTTP Status Codes are standardized three-digit numbers returned by web servers to indicate the result of an HTTP request. Often called "response codes" or simply "status codes," they serve as a communication protocol between client and server. The core purpose is to provide immediate, machine-readable feedback about whether a request succeeded, failed, or requires additional action. This solves the problem of ambiguous request outcomes by establishing a universal language for HTTP transaction results.

## How it works in C#

### 2xx Success

**Explanation:** 2xx codes indicate that the client's request was successfully received, understood, and processed. These are positive responses signaling that everything worked as expected.

```csharp
[ApiController]
[Route("api/[controller]")]
public class ProductsController : ControllerBase
{
    [HttpGet("{id}")]
    public IActionResult GetProduct(int id)
    {
        var product = _productService.GetById(id);
        
        if (product == null)
            return NotFound(); // Returns 404
        
        // 200 OK - Standard success response with data
        return Ok(product); 
    }

    [HttpPost]
    public async Task<IActionResult> CreateProduct(ProductDto productDto)
    {
        var createdProduct = await _productService.CreateAsync(productDto);
        
        // 201 Created - Successfully created resource with location header
        return CreatedAtAction(nameof(GetProduct), 
            new { id = createdProduct.Id }, createdProduct);
    }

    [HttpDelete("{id}")]
    public IActionResult DeleteProduct(int id)
    {
        _productService.Delete(id);
        
        // 204 No Content - Success but no content to return
        return NoContent();
    }
}
```

### 3xx Redirection

**Explanation:** 3xx codes indicate that further action needs to be taken by the client to complete the request. This includes redirects, caching instructions, and resource relocation.

```csharp
public class RedirectController : ControllerBase
{
    [HttpGet("old-url")]
    public IActionResult OldEndpoint()
    {
        // 301 Moved Permanently - Resource has permanently moved
        return RedirectPermanent("/api/products/new-url");
    }

    [HttpGet("temp-redirect")]
    public IActionResult TemporaryRedirect()
    {
        // 302 Found (Temporary Redirect) - Resource temporarily at different location
        return Redirect("/api/temporary-location");
    }

    [HttpGet("cache-me")]
    public IActionResult CachedResponse()
    {
        // 304 Not Modified - Use cached version (client sends If-Modified-Since header)
        if (Request.Headers.ContainsKey("If-Modified-Since"))
        {
            return StatusCode(304);
        }
        
        Response.Headers.Add("Last-Modified", DateTime.UtcNow.ToString("R"));
        return Ok(new { data = "This can be cached" });
    }
}
```

### 4xx Client Error

**Explanation:** 4xx codes indicate errors caused by the client's request. This includes malformed requests, authentication issues, or attempting to access forbidden resources.

```csharp
[ApiController]
public class AuthController : ControllerBase
{
    [HttpPost("login")]
    public IActionResult Login(LoginRequest request)
    {
        if (!ModelState.IsValid)
        {
            // 400 Bad Request - Client sent invalid data
            return BadRequest(ModelState);
        }

        var user = _authService.Authenticate(request.Email, request.Password);
        
        if (user == null)
        {
            // 401 Unauthorized - Authentication required/failed
            return Unauthorized("Invalid credentials");
        }

        if (!user.IsActive)
        {
            // 403 Forbidden - Authenticated but not authorized
            return Forbid("Account is deactivated");
        }

        return Ok(new { token = GenerateToken(user) });
    }

    [HttpGet("secure-data/{id}")]
    public IActionResult GetSecureData(int id)
    {
        var data = _service.GetSecureData(id, User.Identity.Name);
        
        if (data == null)
        {
            // 404 Not Found - Resource doesn't exist
            return NotFound($"Data with id {id} not found");
        }

        if (data.Owner != User.Identity.Name)
        {
            // 403 Forbidden - No permission to access
            return Forbid("You don't own this resource");
        }

        return Ok(data);
    }
}
```

### 5xx Server Error

**Explanation:** 5xx codes indicate that the server failed to fulfill a valid request. These are server-side errors where the client's request was correct but the server couldn't process it.

```csharp
public class OrdersController : ControllerBase
{
    [HttpPost("process-order")]
    public async Task<IActionResult> ProcessOrder(Order order)
    {
        try
        {
            await _orderService.ProcessAsync(order);
            return Ok(new { message = "Order processed successfully" });
        }
        catch (DatabaseConnectionException ex)
        {
            // 503 Service Unavailable - Server cannot handle request
            return StatusCode(503, new { error = "Database unavailable, try again later" });
        }
        catch (ExternalApiException ex)
        {
            // 502 Bad Gateway - Upstream server error
            return StatusCode(502, new { error = "Payment gateway unavailable" });
        }
        catch (Exception ex)
        {
            // 500 Internal Server Error - Generic server error
            _logger.LogError(ex, "Unexpected error processing order");
            return StatusCode(500, new { error = "An unexpected error occurred" });
        }
    }

    [HttpGet("heavy-operation")]
    public async Task<IAResult> HeavyOperation()
    {
        try
        {
            // Simulate timeout scenario
            await Task.Delay(30000); // 30 seconds
            return Ok();
        }
        catch (TaskCanceledException)
        {
            // 504 Gateway Timeout - Server acting as gateway timed out
            return StatusCode(504, "Operation timed out");
        }
    }
}
```

### Common Usage

**Explanation:** Common usage patterns involve consistent application of status codes across your API to maintain predictability and proper RESTful semantics.

```csharp
public static class ApiResponseHelper
{
    public static IActionResult Success<T>(T data, string message = null)
    {
        return new OkObjectResult(new ApiResponse<T>
        {
            Success = true,
            Data = data,
            Message = message,
            StatusCode = 200
        });
    }

    public static IActionResult Created<T>(string location, T data)
    {
        var result = new ObjectResult(new ApiResponse<T>
        {
            Success = true,
            Data = data,
            StatusCode = 201
        })
        {
            StatusCode = 201
        };
        result.Headers.Location = location;
        return result;
    }

    public static IActionResult Error(int statusCode, string message, IEnumerable<string> errors = null)
    {
        return new ObjectResult(new ApiResponse<object>
        {
            Success = false,
            Message = message,
            Errors = errors,
            StatusCode = statusCode
        })
        {
            StatusCode = statusCode
        };
    }
}

// Usage in controller
[HttpPut("{id}")]
public IActionResult UpdateProduct(int id, ProductDto productDto)
{
    if (id != productDto.Id)
        return ApiResponseHelper.Error(400, "ID mismatch");

    try
    {
        var updatedProduct = _productService.Update(productDto);
        return ApiResponseHelper.Success(updatedProduct, "Product updated");
    }
    catch (NotFoundException)
    {
        return ApiResponseHelper.Error(404, "Product not found");
    }
}
```

### Custom Codes

**Explanation:** While standard codes cover most scenarios, sometimes you need custom codes for business-specific scenarios, typically using unassigned ranges.

```csharp
public static class CustomStatusCodes
{
    public const int ValidationFailed = 422; // Already standard but demonstrates pattern
    public const int PaymentRequired = 402;
    public const int BusinessRuleViolation = 450; // Custom code
}

public class CustomResults
{
    public static IActionResult BusinessRuleViolation(string message)
    {
        return new ObjectResult(new { error = message }) { StatusCode = 450 };
    }
    
    public static IActionResult InsufficientFunds(decimal amount, decimal balance)
    {
        return new ObjectResult(new 
        { 
            error = "Insufficient funds",
            required = amount,
            currentBalance = balance
        }) 
        { StatusCode = 402 }; // 402 Payment Required
    }
}

// Usage
[HttpPost("purchase")]
public IActionResult Purchase(PurchaseRequest request)
{
    var userBalance = _walletService.GetBalance(User.Identity.Name);
    
    if (userBalance < request.Amount)
    {
        return CustomResults.InsufficientFunds(request.Amount, userBalance);
    }
    
    if (!_businessRules.ValidatePurchase(request))
    {
        return CustomResults.BusinessRuleViolation("Purchase violates business rules");
    }
    
    return Ok();
}
```

### Semantics

**Explanation:** Proper semantic usage ensures your API communicates clearly and consistently, following REST principles and HTTP specifications.

```csharp
public class SemanticApiController : ControllerBase
{
    // Proper RESTful semantics
    [HttpPost] // CREATE → 201 Created
    public IActionResult CreateResource(ResourceDto resource)
    {
        var created = _service.Create(resource);
        return CreatedAtAction(nameof(GetResource), new { id = created.Id }, created);
    }

    [HttpGet] // READ → 200 OK
    public IActionResult GetResources()
    {
        var resources = _service.GetAll();
        return Ok(resources); // 200 with data
    }

    [HttpPut("{id}")] // UPDATE → 200 OK or 204 No Content
    public IActionResult UpdateResource(int id, ResourceDto resource)
    {
        _service.Update(id, resource);
        return NoContent(); // 204 - success but no content to return
    }

    [HttpDelete("{id}")] // DELETE → 204 No Content
    public IActionResult DeleteResource(int id)
    {
        _service.Delete(id);
        return NoContent(); // 204 - resource deleted, nothing to return
    }

    // Proper error semantics
    [HttpGet("{id}")]
    public IActionResult GetResource(int id)
    {
        var resource = _service.GetById(id);
        
        if (resource == null)
            return NotFound(); // 404 - resource doesn't exist
        
        if (!_authService.CanAccess(User, resource))
            return Forbid(); // 403 - exists but no permission
        
        return Ok(resource);
    }
}
```

## Why is HTTP Status Codes important?

1. **Standardized Communication (Protocol Design):** Status codes provide a universal language that enables interoperability between diverse clients and servers, following established web standards.

2. **Separation of Concerns (Single Responsibility Principle):** They separate success and error handling logic, allowing clients to process responses based on clear categories rather than parsing custom error messages.

3. **Predictable Error Handling (Robustness Principle):** Following "be conservative in what you send, liberal in what you accept" ensures APIs gracefully handle both expected and unexpected scenarios through standardized status code ranges.

## Advanced Nuances

### 1. Idempotency and Status Codes

```csharp
public class IdempotentController : ControllerBase
{
    private readonly IDistributedCache _cache;
    
    [HttpPost("idempotent-operation")]
    public async Task<IActionResult> IdempotentOperation([FromHeader] string idempotencyKey, Order order)
    {
        if (string.IsNullOrEmpty(idempotencyKey))
            return BadRequest("Idempotency key required");
            
        // Check if we've already processed this request
        var cachedResult = await _cache.GetStringAsync(idempotencyKey);
        if (cachedResult != null)
        {
            // 409 Conflict or 200 OK? Depends on business logic
            // Some APIs return 200 with cached response, others 409
            return Conflict("Operation already processed");
        }
        
        // Process and cache result
        var result = await _orderService.ProcessAsync(order);
        await _cache.SetStringAsync(idempotencyKey, JsonSerializer.Serialize(result), 
            new DistributedCacheEntryOptions { AbsoluteExpiration = TimeSpan.FromHours(24) });
            
        return Ok(result);
    }
}
```

### 2. Content Negotiation and Status Codes

```csharp
public class ContentNegotiationController : ControllerBase
{
    [HttpGet("complex-response")]
    public IActionResult GetComplexResponse()
    {
        var data = new { message = "Success", timestamp = DateTime.UtcNow };
        
        // Different status codes based on Accept header or query params
        if (Request.Headers.Accept.Contains("application/vnd.company.error+json"))
        {
            // Custom media type might expect different success codes
            return StatusCode(202, new { status = "processing", data });
        }
        
        // Standard JSON response
        return Ok(data);
    }
    
    [HttpPost("conditional-update")]
    public IActionResult ConditionalUpdate(Resource resource)
    {
        // ETag/If-Match for optimistic concurrency
        if (Request.Headers.TryGetValue("If-Match", out var etag))
        {
            if (!_service.ValidateETag(resource.Id, etag))
            {
                // 412 Precondition Failed - Resource was modified
                return StatusCode(412, "Resource modified by another request");
            }
        }
        
        var updated = _service.Update(resource);
        Response.Headers.ETag = updated.ETag;
        return Ok(updated);
    }
}
```

### 3. Progressive Enhancement with Status Codes

```csharp
public class ProgressiveController : ControllerBase
{
    [HttpPost("async-operation")]
    public async Task<IActionResult> AsyncOperation()
    {
        // Start long-running operation
        var operationId = await _longRunningService.StartAsync();
        
        // 202 Accepted - Operation started, check later for result
        return Accepted($"/api/operations/{operationId}/status", new { operationId });
    }
    
    [HttpGet("operations/{id}/status")]
    public IActionResult GetOperationStatus(string id)
    {
        var status = _longRunningService.GetStatus(id);
        
        switch (status.State)
        {
            case OperationState.Pending:
                return Accepted(); // 202 - Still processing
            case OperationState.Completed:
                return Ok(status.Result); // 200 - Done
            case OperationState.Failed:
                return StatusCode(500, status.Error); // 500 - Failed
            default:
                return StatusCode(418); // 418 - Custom logic (I'm a teapot)
        }
    }
}
```

## How this fits the Roadmap

Within the "HTTP Methods and Data" section of the Advanced C# Mastery roadmap, HTTP Status Codes serve as the **fundamental feedback mechanism** that enables proper client-server communication. 

**Prerequisites:** This concept builds upon understanding of HTTP protocols, REST principles, and basic C# web development with ASP.NET Core.

**Unlocks:** Mastery of status codes is essential for:
- **API Design Patterns** - Designing consistent, predictable RESTful APIs
- **Error Handling Strategies** - Implementing robust global exception handling and error responses
- **Advanced Middleware** - Creating custom middleware for status code rewriting and response normalization
- **Microservices Communication** - Proper inter-service communication with meaningful status codes
- **Caching Strategies** - Leveraging status codes for effective client-side and intermediary caching
- **HATEOAS** - Building hypermedia-driven APIs where status codes guide navigation

This foundation enables you to progress to advanced topics like API versioning strategies, content negotiation, and building resilient distributed systems.