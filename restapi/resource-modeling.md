# Resource Modeling in C# API Design

## What is Resource Modeling?

**Resource Modeling** (often called **Resource-Oriented Design** or **Resource-Centric Architecture**) is an API design approach where you model your API around resources—distinct entities that clients can interact with through standardized HTTP operations. The core purpose is to create consistent, discoverable, and intuitive APIs by mapping business entities to RESTful resources. It solves the problem of inconsistent API design patterns that make APIs difficult to learn, maintain, and consume.

## How it works in C#

### Noun-Based Resources
**Explanation**: Resources should be modeled as nouns (things) rather than verbs (actions). This aligns with REST principles where HTTP methods (GET, POST, PUT, DELETE) already represent the actions. Instead of endpoints like `/createProduct`, you model resources like `/products`.

```csharp
// GOOD: Noun-based resources
[ApiController]
[Route("api/products")] // Noun, plural
public class ProductsController : ControllerBase
{
    [HttpGet]
    public IActionResult GetProducts() { /* returns list */ }
    
    [HttpPost]
    public IActionResult CreateProduct(ProductDto product) { /* creates resource */ }
}

// AVOID: Verb-based endpoints
[Route("api/productOperations")]
public class ProductOperationsController : ControllerBase
{
    [HttpPost("create")] // Verb in URL - not RESTful
    public IActionResult Create() { /* ... */ }
}
```

### Relationships
**Explanation**: Resources often relate to each other. Model these relationships hierarchically in your URL structure and through resource references. Parent-child relationships are expressed as nested routes.

```csharp
[ApiController]
public class OrdersController : ControllerBase
{
    // GET api/customers/5/orders - Get orders for customer 5
    [HttpGet("api/customers/{customerId}/orders")]
    public async Task<ActionResult<List<OrderDto>>> GetCustomerOrders(int customerId)
    {
        var orders = await _orderService.GetByCustomerIdAsync(customerId);
        return Ok(orders.Select(o => o.ToDto()));
    }

    // POST api/customers/5/orders - Create order for customer 5
    [HttpPost("api/customers/{customerId}/orders")]
    public async Task<ActionResult<OrderDto>> CreateOrder(int customerId, CreateOrderDto request)
    {
        // request.CustomerId should match customerId from route, or be omitted from body
        var order = await _orderService.CreateAsync(customerId, request);
        return CreatedAtAction(nameof(GetOrder), new { id = order.Id }, order.ToDto());
    }
}

// Resource DTO with relationship references
public class OrderDto
{
    public int Id { get; set; }
    public decimal Total { get; set; }
    public string Status { get; set; }
    
    // Reference to related resource (HATEOAS-style)
    public string CustomerUrl { get; set; } // "/api/customers/5"
    
    // Or embedded ID (simpler approach)
    public int CustomerId { get; set; }
}
```

### Plural Naming
**Explanation**: Use plural nouns for resource collections to indicate you're working with sets of resources. This consistency makes APIs predictable and follows common REST conventions.

```csharp
// CORRECT: Plural resource names
[Route("api/products")]        // Collection of products
[Route("api/users")]           // Collection of users  
[Route("api/orders")]          // Collection of orders

// AVOID: Mixed singular/plural
[Route("api/product")]         // Inconsistent - should be plural
[Route("api/users")]           // Good
[Route("api/order")]          // Inconsistent

// Example with plural naming throughout
[ApiController]
[Route("api/products")]
public class ProductsController : ControllerBase
{
    [HttpGet] // GET api/products - get collection
    public IActionResult GetProducts() => Ok(_products);
    
    [HttpGet("{id}")] // GET api/products/5 - get specific resource
    public IActionResult GetProduct(int id) => Ok(_products.Find(p => p.Id == id));
    
    [HttpPost] // POST api/products - create new resource in collection
    public IActionResult CreateProduct(ProductDto product) { /* ... */ }
}
```

### Versioning
**Explanation**: API versioning is crucial for maintaining backward compatibility. Common approaches include URL-based versioning, query string versioning, or header-based versioning.

```csharp
// URL-based versioning (most common)
[ApiController]
[Route("api/v{version:apiVersion}/products")]
[ApiVersion("1.0")]
public class ProductsControllerV1 : ControllerBase
{
    [HttpGet]
    public IActionResult GetProducts()
    {
        // V1 response - simple structure
        var products = _productService.GetAll();
        return Ok(products.Select(p => new ProductDtoV1 
        { 
            Id = p.Id, 
            Name = p.Name, 
            Price = p.Price 
        }));
    }
}

[Route("api/v{version:apiVersion}/products")]
[ApiVersion("2.0")]
public class ProductsControllerV2 : ControllerBase
{
    [HttpGet]
    public IActionResult GetProducts()
    {
        // V2 response - enhanced structure
        var products = _productService.GetAll();
        return Ok(products.Select(p => new ProductDtoV2 
        { 
            Id = p.Id,
            Name = p.Name,
            Price = p.Price,
            Category = p.Category, // New field
            CreatedDate = p.CreatedDate // New field
        }));
    }
}

// Configure versioning in Startup.cs or Program.cs
builder.Services.AddApiVersioning(options =>
{
    options.DefaultApiVersion = new ApiVersion(1, 0);
    options.AssumeDefaultVersionWhenUnspecified = true;
    options.ReportApiVersions = true; // Include version info in headers
});
```

### Pagination
**Explanation**: For large collections, pagination prevents overwhelming clients with huge responses and improves performance. Standard patterns include page-based or cursor-based pagination.

```csharp
[ApiController]
[Route("api/products")]
public class ProductsController : ControllerBase
{
    [HttpGet]
    public async Task<ActionResult<PagedResponse<ProductDto>>> GetProducts(
        [FromQuery] int page = 1, 
        [FromQuery] int pageSize = 20,
        [FromQuery] string sortBy = "name")
    {
        // Validate pagination parameters
        page = Math.Max(1, page);
        pageSize = Math.Clamp(pageSize, 1, 100); // Prevent excessive page sizes
        
        var (products, totalCount) = await _productService.GetPagedAsync(page, pageSize, sortBy);
        
        // Build pagination response with metadata
        var response = new PagedResponse<ProductDto>
        {
            Data = products.Select(p => p.ToDto()),
            Pagination = new PaginationMetadata
            {
                Page = page,
                PageSize = pageSize,
                TotalCount = totalCount,
                TotalPages = (int)Math.Ceiling(totalCount / (double)pageSize)
            }
        };
        
        return Ok(response);
    }
}

// Pagination response model
public class PagedResponse<T>
{
    public IEnumerable<T> Data { get; set; }
    public PaginationMetadata Pagination { get; set; }
}

public class PaginationMetadata
{
    public int Page { get; set; }
    public int PageSize { get; set; }
    public int TotalCount { get; set; }
    public int TotalPages { get; set; }
    
    // Optional: Add URLs for next/previous pages (HATEOAS)
    public string NextPageUrl { get; set; }
    public string PreviousPageUrl { get; set; }
}
```

### Filtering/Sorting
**Explanation**: Allow clients to filter and sort results based on resource properties. Use query parameters for flexibility while maintaining a consistent interface.

```csharp
[ApiController]
[Route("api/products")]
public class ProductsController : ControllerBase
{
    [HttpGet]
    public async Task<ActionResult<PagedResponse<ProductDto>>> GetProducts(
        [FromQuery] ProductQueryParameters queryParams)
    {
        // Apply filtering and sorting
        var filteredQuery = _dbContext.Products.AsQueryable();
        
        // Filtering
        if (!string.IsNullOrEmpty(queryParams.Category))
            filteredQuery = filteredQuery.Where(p => p.Category == queryParams.Category);
        
        if (queryParams.MinPrice.HasValue)
            filteredQuery = filteredQuery.Where(p => p.Price >= queryParams.MinPrice);
            
        if (queryParams.MaxPrice.HasValue)
            filteredQuery = filteredQuery.Where(p => p.Price <= queryParams.MaxPrice);
        
        // Sorting
        filteredQuery = queryParams.SortBy?.ToLower() switch
        {
            "price" => queryParams.SortDescending ? 
                      filteredQuery.OrderByDescending(p => p.Price) : 
                      filteredQuery.OrderBy(p => p.Price),
            "name" => queryParams.SortDescending ?
                     filteredQuery.OrderByDescending(p => p.Name) :
                     filteredQuery.OrderBy(p => p.Name),
            _ => filteredQuery.OrderBy(p => p.Id) // Default sort
        };
        
        // Pagination
        var totalCount = await filteredQuery.CountAsync();
        var products = await filteredQuery
            .Skip((queryParams.Page - 1) * queryParams.PageSize)
            .Take(queryParams.PageSize)
            .ToListAsync();
            
        return Ok(new PagedResponse<ProductDto> 
        { 
            Data = products.Select(p => p.ToDto()),
            Pagination = new PaginationMetadata 
            { 
                Page = queryParams.Page, 
                PageSize = queryParams.PageSize, 
                TotalCount = totalCount 
            }
        });
    }
}

// Centralized query parameters model
public class ProductQueryParameters
{
    public int Page { get; set; } = 1;
    public int PageSize { get; set; } = 20;
    
    // Filtering parameters
    public string Category { get; set; }
    public decimal? MinPrice { get; set; }
    public decimal? MaxPrice { get; set; }
    
    // Sorting parameters
    public string SortBy { get; set; } = "name";
    public bool SortDescending { get; set; } = false;
}
```

## Why is Resource Modeling important?

1. **Consistency (Uniform Interface Principle)**: Provides a predictable API structure that follows REST constraints, making APIs self-descriptive and easier for consumers to understand and use correctly.

2. **Scalability (Separation of Concerns)**: Separates resource representation from business logic, allowing independent evolution of API contracts and backend implementations while maintaining backward compatibility.

3. **Maintainability (DRY Principle)**: Centralizes common patterns like pagination and filtering into reusable components, reducing code duplication and making the codebase easier to maintain and extend.

## Advanced Nuances

### 1. Hypermedia (HATEOAS) for Discoverability
Advanced resource modeling includes hypermedia controls that make APIs self-discoverable. This moves beyond simple CRUD to fully navigable APIs.

```csharp
public class ProductResource
{
    public int Id { get; set; }
    public string Name { get; set; }
    public decimal Price { get; set; }
    public List<HypermediaLink> Links { get; set; } = new();
}

// Usage in controller
var productResource = product.ToResource();
productResource.Links.Add(new HypermediaLink 
{ 
    Href = $"/api/products/{product.Id}", 
    Rel = "self", 
    Method = "GET" 
});
productResource.Links.Add(new HypermediaLink 
{ 
    Href = $"/api/products/{product.Id}", 
    Rel = "delete", 
    Method = "DELETE" 
});
```

### 2. CQRS Pattern Integration
For complex scenarios, resource modeling can integrate with Command Query Responsibility Segregation, separating read and write models while maintaining a consistent resource interface.

```csharp
// Command (write) - separate from query model
public class CreateProductCommand : IRequest<int>
{
    public string Name { get; set; }
    public decimal Price { get; set; }
}

// Query (read) - optimized for reading
public class GetProductsQuery : IRequest<PagedResponse<ProductDto>>
{
    public ProductQueryParameters Parameters { get; set; }
}

// Controller uses Mediator pattern
[HttpPost]
public async Task<ActionResult<int>> CreateProduct(CreateProductCommand command)
{
    var productId = await _mediator.Send(command);
    return CreatedAtAction(nameof(GetProduct), new { id = productId }, productId);
}
```

### 3. Resource Versioning with Content Negotiation
Advanced versioning strategies include content negotiation where the same endpoint serves different representations based on Accept headers.

```csharp
[ApiController]
[Route("api/products")]
public class ProductsController : ControllerBase
{
    [HttpGet("{id}")]
    public IActionResult GetProduct(int id)
    {
        var product = _productService.GetById(id);
        
        // Version based on Accept header
        if (Request.Headers["Accept"].ToString().Contains("application/vnd.company.v2+json"))
            return Ok(product.ToV2Dto());
        else
            return Ok(product.ToV1Dto()); // Default
    }
}
```

## How this fits the Roadmap

Resource Modeling sits as a **foundational pillar** within the "API Design and Development" section. It's a prerequisite for more advanced topics because it establishes the fundamental patterns that all subsequent API concepts build upon.

**This concept unlocks:**
- **API Security**: Proper resource modeling defines clear boundaries for authorization policies
- **Performance Optimization**: Understanding resource relationships enables efficient loading strategies (GraphQL, DTO projection)
- **Microservices Architecture**: Resource modeling principles directly translate to service boundaries and inter-service communication
- **Documentation Generation**: Consistent resource patterns enable automated OpenAPI/Swagger documentation

**Builds upon**: Basic C# fundamentals, HTTP protocol knowledge, REST principles. Masters resource modeling before moving to advanced API security, performance optimization, and architectural patterns.