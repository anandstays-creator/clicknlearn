# Endpoint Design in C# - Advanced C# Mastery Guide

## What is Endpoint Design?

**Endpoint Design** (also known as *Route Design* or *API Endpoint Architecture*) is the practice of structuring and organizing the entry points of your web API. It defines how clients interact with your application through HTTP endpoints, encompassing URL patterns, parameter handling, and resource organization. 

The core purpose is to create a **consistent, intuitive, and maintainable** interface that follows RESTful principles while solving the problem of **unstructured API sprawl** where endpoints become difficult to discover, document, and maintain over time.

## How it works in C#

### URL Structures
**Explanation**: URL Structures define the hierarchical organization of your API endpoints. They should represent your domain model clearly, using nouns (resources) rather than verbs (actions), and follow a logical hierarchy from general to specific.

```csharp
[ApiController]
[Route("api/[controller]")]
public class ProductsController : ControllerBase
{
    // GET api/products
    [HttpGet]
    public IActionResult GetAllProducts() 
    {
        // Returns collection of products
        return Ok(new[] { new { Id = 1, Name = "Laptop" } });
    }

    // GET api/products/5
    [HttpGet("{id}")]
    public IActionResult GetProduct(int id)
    {
        // Returns specific product by ID
        return Ok(new { Id = id, Name = "Laptop" });
    }

    // POST api/products
    [HttpPost]
    public IActionResult CreateProduct([FromBody] ProductDto product)
    {
        // Creates new product
        return CreatedAtAction(nameof(GetProduct), new { id = product.Id }, product);
    }
}
```

### URI Templating
**Explanation**: URI Templating uses route parameters to create dynamic endpoints. In C#, this is achieved through route templates with placeholders that map to method parameters, enabling flexible and reusable endpoint patterns.

```csharp
[ApiController]
public class OrdersController : ControllerBase
{
    // Advanced URI template with multiple parameters and constraints
    [HttpGet("customers/{customerId:int}/orders/{orderId:guid}")]
    public IActionResult GetOrder(int customerId, Guid orderId)
    {
        // {customerId:int} constrains parameter to integers only
        // {orderId:guid} constrains parameter to GUID format
        return Ok(new { CustomerId = customerId, OrderId = orderId });
    }

    // Optional parameters with default values
    [HttpGet("search/{category?}/{minPrice?}")]
    public IActionResult SearchProducts(string category = "all", decimal? minPrice = null)
    {
        // Both parameters are optional with default values
        return Ok(new { Category = category, MinPrice = minPrice });
    }
}
```

### Query/Matrix Parameters
**Explanation**: Query parameters (after `?`) are used for filtering, pagination, and optional data. Matrix parameters (semicolon-separated) are less common but can represent resource-specific metadata. C# automatically binds these to method parameters.

```csharp
[ApiController]
[Route("api/[controller]")]
public class ProductsController : ControllerBase
{
    // Using query parameters for filtering and pagination
    [HttpGet]
    public IActionResult GetProducts(
        [FromQuery] string category = null,      // Optional filtering
        [FromQuery] decimal? minPrice = null,    // Optional range filter
        [FromQuery] int page = 1,               // Pagination with default
        [FromQuery] int pageSize = 20)           // Page size with default
    {
        // Implementation using the query parameters
        var filter = new ProductFilter 
        { 
            Category = category, 
            MinPrice = minPrice,
            Page = page,
            PageSize = pageSize
        };
        
        return Ok(await _productService.SearchAsync(filter));
    }

    // Matrix parameter example (less common but supported)
    [HttpGet("categories/{category}/products")]
    public IActionResult GetProductsByCategory(
        string category,
        [FromQuery] string color,           // Query param: ?color=red
        [FromRoute] string categoryDetails)  // Matrix: /categories/electronics;sort=price
    {
        // Matrix parameters are part of the route but less commonly used
        return Ok(new { Category = category, Color = color });
    }
}
```

### Custom Actions
**Explanation**: For operations that don't fit CRUD patterns, custom actions extend RESTful principles using: 1) Custom HTTP methods on existing resources, or 2) Specialized sub-resource endpoints that represent actions rather than entities.

```csharp
[ApiController]
[Route("api/[controller]")]
public class OrdersController : ControllerBase
{
    // Custom action on existing resource - prefer this approach
    [HttpPost("{id}/cancel")]
    public IActionResult CancelOrder(int id)
    {
        // Represents an action on a specific order
        _orderService.CancelOrder(id);
        return Ok(new { Message = $"Order {id} cancelled" });
    }

    // Specialized endpoint for non-CRUD operations
    [HttpPost("bulk-ship")]
    public IActionResult BulkShipOrders([FromBody] int[] orderIds)
    {
        // Represents a business operation that doesn't map to CRUD
        _shippingService.BulkShip(orderIds);
        return Ok(new { ShippedOrders = orderIds.Length });
    }

    // Custom action with HTTP PATCH for partial updates
    [HttpPatch("{id}/status")]
    public IActionResult UpdateOrderStatus(int id, [FromBody] OrderStatusUpdate statusUpdate)
    {
        _orderService.UpdateStatus(id, statusUpdate.NewStatus);
        return Ok(new { OrderId = id, Status = statusUpdate.NewStatus });
    }
}
```

### Nested Resources
**Explanation**: Nested resources represent hierarchical relationships where child resources only exist in the context of their parent. This creates intuitive URLs that reflect real-world relationships.

```csharp
[ApiController]
public class CustomerOrdersController : ControllerBase
{
    // Nested resource: Orders belong to a specific customer
    [HttpGet("api/customers/{customerId}/orders")]
    public IActionResult GetCustomerOrders(int customerId)
    {
        // All orders for a specific customer
        var orders = _orderService.GetOrdersByCustomer(customerId);
        return Ok(orders);
    }

    // Specific order within customer context
    [HttpGet("api/customers/{customerId}/orders/{orderId}")]
    public IActionResult GetCustomerOrder(int customerId, int orderId)
    {
        // Verify the order belongs to the customer (authorization check)
        var order = _orderService.GetCustomerOrder(customerId, orderId);
        if (order == null)
            return NotFound();
            
        return Ok(order);
    }

    // Creating a new order for a specific customer
    [HttpPost("api/customers/{customerId}/orders")]
    public IActionResult CreateCustomerOrder(int customerId, [FromBody] OrderDto order)
    {
        order.CustomerId = customerId; // Ensure relationship integrity
        var createdOrder = _orderService.CreateOrder(order);
        
        return CreatedAtAction(
            nameof(GetCustomerOrder), 
            new { customerId, orderId = createdOrder.Id }, 
            createdOrder);
    }
}
```

## Why is Endpoint Design Important?

### 1. **Scalability through Resource-Oriented Architecture**
Well-designed endpoints isolate concerns and prevent dependency chains, allowing independent scaling of different resource handlers following the **Single Responsibility Principle**.

### 2. **Maintainability via DRY (Don't Repeat Yourself)**
Consistent URL patterns and parameter handling reduce code duplication, making APIs easier to extend and refactor as requirements evolve.

### 3. **Discoverability through Intuitive Hierarchy**
Following RESTful conventions creates **self-documenting APIs** where clients can predict endpoint structures, reducing the learning curve and documentation burden.

## Advanced Nuances

### 1. **Overlapping Route Ambiguity**
When multiple routes match the same URL, ASP.NET Core uses complex routing precedence rules. Senior developers should understand route ordering and constraint usage to prevent ambiguity.

```csharp
// Problem: Both routes could match "api/products/search/electronics"
[HttpGet("search/{category}")]
public IActionResult SearchByCategory(string category) { }

[HttpGet("search/{id:int}")]
public IActionResult SearchById(int id) { }

// Solution: Use Order attribute or more specific constraints
[HttpGet("search/category/{category}", Order = 1)]
[HttpGet("search/id/{id:int}", Order = 2)]
```

### 2. **PATCH vs PUT Idempotency**
Advanced APIs distinguish between partial updates (PATCH) and full replacements (PUT). PATCH endpoints should be designed to handle sparse updates idempotently when possible.

```csharp
[HttpPatch("{id}")]
public IActionResult PatchProduct(int id, [FromBody] JsonPatchDocument<ProductDto> patchDoc)
{
    // JSON Patch format allows specific, idempotent partial updates
    var product = _productService.GetProduct(id);
    patchDoc.ApplyTo(product);
    _productService.UpdateProduct(product);
    return Ok(product);
}
```

### 3. **HATEOAS for Self-Descriptive APIs**
Senior developers implement Hypermedia As The Engine Of Application State to make APIs discoverable by including links to related resources in responses.

```csharp
[HttpGet("{id}")]
public IActionResult GetProduct(int id)
{
    var product = _productService.GetProduct(id);
    var response = new 
    {
        Product = product,
        Links = new[]
        {
            new { Rel = "self", Href = Url.Action(nameof(GetProduct), new { id }) },
            new { Rel = "orders", Href = Url.Action("GetProductOrders", new { productId = id }) }
        }
    };
    return Ok(response);
}
```

## How this Fits the Roadmap

Within the "API Design and Development" section, **Endpoint Design** serves as the **foundational layer** that precedes more advanced topics. It's the prerequisite for:

- **API Versioning**: Well-structured endpoints make versioning strategies (URL, header, query-based) easier to implement
- **Hypermedia & HATEOAS**: Consistent resource naming enables meaningful link generation
- **Advanced Authentication/Authorization**: Clear resource hierarchies support granular security policies
- **API Documentation (Swagger/OpenAPI**: Logical endpoint structures generate cleaner, more useful documentation

This concept unlocks **advanced API patterns** like CQRS endpoints, microservice API gateways, and GraphQL resolvers, establishing the architectural basis for enterprise-scale API development in the C# ecosystem.