# REST Principles in C#

## What is REST Principles?

REST (Representational State Transfer), often called RESTful principles, is an architectural style for designing networked applications. It provides a set of constraints for creating scalable web services. REST solves the problem of building maintainable, scalable, and standardized web APIs by defining clear boundaries and communication patterns between clients and servers.

## How it works in C#

### Client-Server Separation

**Explanation**: This constraint mandates a clear separation between the client (UI/consumer) and server (data/processing). The client is responsible for the user interface and user experience, while the server handles data storage, business logic, and scalability concerns.

```csharp
// Server-side Controller (Web API)
[ApiController]
[Route("api/[controller]")]
public class ProductsController : ControllerBase
{
    private readonly IProductService _productService;

    // Server focuses on business logic and data
    public ProductsController(IProductService productService)
    {
        _productService = productService;
    }

    [HttpGet("{id}")]
    public async Task<ActionResult<ProductDto>> GetProduct(int id)
    {
        // Server handles data retrieval and validation
        var product = await _productService.GetProductByIdAsync(id);
        return Ok(product);
    }
}

// Client-side Consumer (Separate application)
public class ProductClient
{
    private readonly HttpClient _httpClient;

    public ProductClient(HttpClient httpClient)
    {
        _httpClient = httpClient;
    }

    public async Task<ProductDto> GetProductAsync(int id)
    {
        // Client only knows how to make HTTP requests and handle responses
        var response = await _httpClient.GetAsync($"api/products/{id}");
        response.EnsureSuccessStatusCode();
        
        var product = await response.Content.ReadFromJsonAsync<ProductDto>();
        return product;
    }
}
```

### Statelessness Constraint

**Explanation**: Each request from client to server must contain all the information needed to understand and process the request. The server should not store any client context between requests.

```csharp
[ApiController]
public class ShoppingCartController : ControllerBase
{
    [HttpPost("add-item")]
    public async Task<ActionResult> AddToCart([FromBody] AddToCartRequest request)
    {
        // Server doesn't rely on session state - all necessary info comes in the request
        var userId = request.UserId; // Included in every request
        var productId = request.ProductId;
        var quantity = request.Quantity;

        // Process the request independently
        await _cartService.AddItemAsync(userId, productId, quantity);
        
        return Ok(new { success = true });
    }

    [HttpPost("checkout")]
    public async Task<ActionResult> Checkout([FromBody] CheckoutRequest request)
    {
        // Each request is self-contained - no server-side session tracking
        var userId = request.UserId; // Must be provided again
        var paymentInfo = request.PaymentInfo;
        
        await _orderService.ProcessCheckoutAsync(userId, paymentInfo);
        return Ok(new { orderId = Guid.NewGuid() });
    }
}

public class AddToCartRequest
{
    public int UserId { get; set; }        // Required for stateless operation
    public int ProductId { get; set; }     // All context in request
    public int Quantity { get; set; }      // No server-side state
}
```

### Cacheability Overview

**Explanation**: Responses should be explicitly labeled as cacheable or non-cacheable to improve performance by reducing client-server interactions.

```csharp
[ApiController]
public class ProductCatalogController : ControllerBase
{
    [HttpGet("products")]
    [ResponseCache(Duration = 300)] // Cache for 5 minutes
    public async Task<ActionResult<List<ProductDto>>> GetProducts()
    {
        var products = await _productService.GetAllProductsAsync();
        
        // Client and intermediaries can cache this response
        return Ok(products);
    }

    [HttpGet("products/{id}")]
    [ResponseCache(Duration = 600)] // Cache for 10 minutes
    public async Task<ActionResult<ProductDto>> GetProduct(int id)
    {
        var product = await _productService.GetProductByIdAsync(id);
        
        // Add cache headers explicitly
        Response.Headers.CacheControl = "public, max-age=600";
        return Ok(product);
    }

    [HttpPost("products")]
    [ResponseCache(NoStore = true)] // Prevent caching of sensitive operations
    public async Task<ActionResult> CreateProduct([FromBody] CreateProductDto product)
    {
        await _productService.CreateProductAsync(product);
        
        // POST responses typically shouldn't be cached
        Response.Headers.CacheControl = "no-store";
        return CreatedAtAction(nameof(GetProduct), new { id = product.Id }, product);
    }
}
```

### Uniform Interface Basics

**Explanation**: A consistent way of interacting with resources through standardized HTTP methods, resource identification, and self-descriptive messages.

```csharp
[ApiController]
[Route("api/[controller]")]
public class OrdersController : ControllerBase
{
    // GET - Retrieve resource(s)
    [HttpGet]
    public async Task<ActionResult<List<OrderDto>>> GetOrders()
    {
        return Ok(await _orderService.GetAllOrdersAsync());
    }

    // GET with ID - Retrieve specific resource
    [HttpGet("{id}")]
    public async Task<ActionResult<OrderDto>> GetOrder(int id)
    {
        var order = await _orderService.GetOrderByIdAsync(id);
        return order != null ? Ok(order) : NotFound();
    }

    // POST - Create new resource
    [HttpPost]
    public async Task<ActionResult<OrderDto>> CreateOrder([FromBody] CreateOrderDto orderDto)
    {
        var createdOrder = await _orderService.CreateOrderAsync(orderDto);
        return CreatedAtAction(nameof(GetOrder), new { id = createdOrder.Id }, createdOrder);
    }

    // PUT - Replace entire resource
    [HttpPut("{id}")]
    public async Task<ActionResult> UpdateOrder(int id, [FromBody] UpdateOrderDto orderDto)
    {
        await _orderService.UpdateOrderAsync(id, orderDto);
        return NoContent();
    }

    // DELETE - Remove resource
    [HttpDelete("{id}")]
    public async Task<ActionResult> DeleteOrder(int id)
    {
        await _orderService.DeleteOrderAsync(id);
        return NoContent();
    }
}
```

### Layered System Concept

**Explanation**: A client cannot tell whether it's connected directly to the end server or to an intermediary, allowing for load balancers, proxies, and security layers.

```csharp
// Client code - unaware of server architecture
public class ApiClient
{
    private readonly HttpClient _httpClient;

    public async Task<string> GetDataAsync()
    {
        // Client doesn't know about load balancers, caching layers, or security gateways
        var response = await _httpClient.GetAsync("api/data");
        return await response.Content.ReadAsStringAsync();
    }
}

// Server with multiple layers
public class Program
{
    public static void Main(string[] args)
    {
        var builder = WebApplication.CreateBuilder(args);
        
        // Layer 1: Infrastructure (IoC, Configuration)
        builder.Services.AddControllers();
        builder.Services.AddScoped<IDataService, DataService>();
        
        // Layer 2: Security Middleware
        builder.Services.AddAuthentication(JwtBearerDefaults.AuthenticationScheme)
            .AddJwtBearer(options => { /* JWT config */ });
        
        // Layer 3: Caching Layer
        builder.Services.AddResponseCaching();
        
        // Layer 4: Routing and Endpoints
        var app = builder.Build();
        app.UseRouting();
        app.UseAuthentication();
        app.UseAuthorization();
        app.UseResponseCaching();
        app.MapControllers();
        
        app.Run();
    }
}
```

### Code on Demand

**Explanation**: Servers can temporarily extend client functionality by transferring executable code (like JavaScript). This is optional and least commonly used.

```csharp
[ApiController]
public class DynamicFormsController : ControllerBase
{
    [HttpGet("form-validation/{formType}")]
    public ActionResult GetFormValidationLogic(string formType)
    {
        // Return JavaScript code that the client can execute
        var validationScript = formType switch
        {
            "registration" => """
                function validateForm(data) {
                    if (!data.email.includes('@')) return 'Invalid email';
                    if (data.password.length < 8) return 'Password too short';
                    return null;
                }
                """,
            "payment" => """
                function validateForm(data) {
                    if (!data.cardNumber || data.cardNumber.length !== 16) 
                        return 'Invalid card number';
                    return null;
                }
                """,
            _ => "function validateForm(data) { return null; }"
        };

        return Content(validationScript, "application/javascript");
    }
}

// Client-side usage
public class DynamicFormValidator
{
    private readonly HttpClient _httpClient;
    
    public async Task<string> LoadValidationLogicAsync(string formType)
    {
        var response = await _httpClient.GetAsync($"api/form-validation/{formType}");
        return await response.Content.ReadAsStringAsync();
    }
    
    // The returned JavaScript would be executed in a JavaScript engine
}
```

### Resources Identification

**Explanation**: Everything is a resource identified by a unique URI. Resources are separate from their representations.

```csharp
[ApiController]
[Route("api/[controller]")]
public class ResourceBasedController : ControllerBase
{
    [HttpGet("customers/{customerId}/orders/{orderId}")]
    public async Task<ActionResult<OrderDto>> GetCustomerOrder(
        [FromRoute] int customerId, 
        [FromRoute] int orderId)
    {
        // URI: /api/resourcebased/customers/123/orders/456
        // Clearly identifies the specific resource hierarchy
        var order = await _orderService.GetCustomerOrderAsync(customerId, orderId);
        return Ok(order);
    }

    [HttpGet("products/{id}")]
    public async Task<ActionResult> GetProduct(int id)
    {
        var product = await _productService.GetProductByIdAsync(id);
        
        if (product == null)
            return NotFound();
            
        // Resource representation can vary based on Accept header
        if (Request.Headers.Accept.Contains("application/xml"))
        {
            return Content(SerializeToXml(product), "application/xml");
        }
        
        return Ok(product); // Default JSON representation
    }

    [HttpPost("products/{id}/reviews")]
    public async Task<ActionResult> AddReview(int id, [FromBody] ReviewDto review)
    {
        // URI identifies the resource being acted upon
        var createdReview = await _reviewService.AddReviewAsync(id, review);
        
        // Location header points to the new resource
        return Created($"api/resourcebased/products/{id}/reviews/{createdReview.Id}", createdReview);
    }
}
```

## Why is REST Principles important?

1. **Scalability (Statelessness)**: Stateless servers can handle more concurrent requests since they don't maintain client state, allowing horizontal scaling through load balancers.

2. **Separation of Concerns (Single Responsibility Principle)**: Client-server separation enforces clear boundaries, making both components independently maintainable and evolvable.

3. **Interoperability (Uniform Interface)**: Standard HTTP methods and status codes enable different clients (web, mobile, third-party) to consistently interact with the API without custom protocols.

## Advanced Nuances

### 1. HATEOAS (Hypermedia as the Engine of Application State)

```csharp
[HttpGet("orders/{id}")]
public async Task<ActionResult<OrderDto>> GetOrder(int id)
{
    var order = await _orderService.GetOrderByIdAsync(id);
    
    // Include hypermedia links for discoverability
    order.Links = new List<LinkDto>
    {
        new() { Href = $"api/orders/{id}", Rel = "self", Method = "GET" },
        new() { Href = $"api/orders/{id}", Rel = "update", Method = "PUT" },
        new() { Href = $"api/orders/{id}", Rel = "delete", Method = "DELETE" },
        new() { Href = "api/orders", Rel = "collection", Method = "GET" }
    };
    
    return Ok(order);
}

public class OrderDto
{
    public int Id { get; set; }
    public decimal Total { get; set; }
    public string Status { get; set; }
    public List<LinkDto> Links { get; set; } // Hypermedia controls
}
```

### 2. Conditional Requests and ETags for Optimistic Concurrency

```csharp
[HttpPut("products/{id}")]
public async Task<ActionResult> UpdateProduct(int id, [FromBody] ProductDto product)
{
    // Check ETag for concurrency control
    if (Request.Headers.TryGetValue("If-Match", out var etag))
    {
        var currentEtag = await _productService.GetCurrentETagAsync(id);
        if (etag != currentEtag)
            return StatusCode(412, "Resource has been modified"); // Precondition Failed
    }
    
    await _productService.UpdateProductAsync(id, product);
    
    // Return new ETag
    var newEtag = await _productService.GetCurrentETagAsync(id);
    Response.Headers.ETag = newEtag;
    
    return NoContent();
}
```

### 3. API Versioning with URI or Content Negotiation

```csharp
// URI Versioning
[ApiController]
[Route("api/v{version:apiVersion}/[controller]")]
[ApiVersion("1.0")]
public class ProductsControllerV1 : ControllerBase
{
    [HttpGet]
    public ActionResult<List<ProductDto>> GetProducts() => Ok(_productsV1);
}

[ApiController]
[Route("api/v{version:apiVersion}/[controller]")]
[ApiVersion("2.0")]
public class ProductsControllerV2 : ControllerBase
{
    [HttpGet]
    public ActionResult<List<ProductDtoV2>> GetProducts() => Ok(_productsV2);
}

// Content Negotiation Versioning
[ApiController]
[Route("api/[controller]")]
public class ProductsController : ControllerBase
{
    [HttpGet]
    public ActionResult GetProducts()
    {
        var version = Request.Headers["Accept"].ToString().Contains("v2") ? 2 : 1;
        return version == 2 ? Ok(_productsV2) : Ok(_productsV1);
    }
}
```

## How this fits the Roadmap

Within the "Core Concepts" section of the Advanced C# Mastery roadmap, REST Principles serves as a **fundamental building block** for modern web API development. It's a prerequisite for:

- **Advanced API Design Patterns** (CQRS, Mediator, Repository patterns)
- **Microservices Architecture** (service-to-service communication)
- **API Security** (Authentication/Authorization implementations)
- **Performance Optimization** (Caching strategies, pagination)
- **API Versioning and Evolution** (maintaining backward compatibility)

Mastering REST principles unlocks more advanced topics like GraphQL integration, gRPC implementations, and distributed systems design, forming the foundation for enterprise-level application architecture in C#.