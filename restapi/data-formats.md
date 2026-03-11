# Data Formats in C#

## What is Data Formats?

**Data Formats** (also known as data serialization formats or interchange formats) are standardized structures for representing and transmitting data between systems. They solve the fundamental problem of enabling disparate applications, services, and platforms to exchange information reliably by providing a common language for data representation. The core purpose is to convert complex object structures into standardized text or binary representations that can be stored, transmitted, and reconstructed accurately.

## How it works in C#

### JSON
JSON (JavaScript Object Notation) is a lightweight, human-readable data interchange format that uses key-value pairs and arrays. It's the most common format for web APIs due to its simplicity and compatibility with JavaScript.

```csharp
using System.Text.Json;

public class Person
{
    public string Name { get; set; }
    public int Age { get; set; }
    public List<string> Hobbies { get; set; }
}

// Serialization
var person = new Person { Name = "Alice", Age = 30, Hobbies = new List<string> { "reading", "hiking" } };
string jsonString = JsonSerializer.Serialize(person, new JsonSerializerOptions { WriteIndented = true });
// Output: {"Name": "Alice", "Age": 30, "Hobbies": ["reading", "hiking"]}

// Deserialization
string json = "{\"Name\":\"Bob\",\"Age\":25,\"Hobbies\":[\"gaming\"]}";
Person deserializedPerson = JsonSerializer.Deserialize<Person>(json);
```

### XML
XML (eXtensible Markup Language) is a markup language that defines a set of rules for encoding documents in a human-readable format. It's more verbose than JSON but offers stronger schema validation capabilities.

```csharp
using System.Xml.Serialization;
using System.IO;

[XmlRoot("Person")]
public class Person
{
    [XmlElement("Name")]
    public string Name { get; set; }
    
    [XmlElement("Age")]
    public int Age { get; set; }
}

// Serialization
var person = new Person { Name = "Charlie", Age = 35 };
var serializer = new XmlSerializer(typeof(Person));
using var writer = new StringWriter();
serializer.Serialize(writer, person);
string xmlString = writer.ToString();
// Output: <?xml version="1.0"?><Person><Name>Charlie</Name><Age>35</Age></Person>

// Deserialization
string xml = "<Person><Name>Diana</Name><Age>28</Age></Person>";
using var reader = new StringReader(xml);
Person deserializedPerson = (Person)serializer.Deserialize(reader);
```

### YAML
YAML (YAML Ain't Markup Language) is a human-friendly data serialization standard that emphasizes readability. It's commonly used for configuration files and less frequently for API communication.

```csharp
using YamlDotNet.Serialization;

public class Config
{
    public string Database { get; set; }
    public int Port { get; set; }
    public List<string> Servers { get; set; }
}

// Serialization
var config = new Config { Database = "test", Port = 5432, Servers = new List<string> { "primary", "secondary" } };
var serializer = new SerializerBuilder().Build();
string yaml = serializer.Serialize(config);
// Output:
// Database: test
// Port: 5432
// Servers:
// - primary
// - secondary

// Deserialization
string yamlContent = "Database: production\nPort: 3306";
var deserializer = new DeserializerBuilder().Build();
Config deserializedConfig = deserializer.Deserialize<Config>(yamlContent);
```

### Serialization
Serialization is the process of converting an object's state into a format that can be stored or transmitted. It transforms complex object graphs into a flat, sequential format.

```csharp
using System.Text.Json;

public class Order
{
    public int Id { get; set; }
    public DateTime OrderDate { get; set; }
    public Customer Customer { get; set; }
}

// Custom serializer with specific options
var order = new Order { Id = 1, OrderDate = DateTime.UtcNow };
var options = new JsonSerializerOptions
{
    PropertyNamingPolicy = JsonNamingPolicy.CamelCase,
    WriteIndented = true
};
string serialized = JsonSerializer.Serialize(order, options);
```

### Deserialization
Deserialization is the reverse process - reconstructing objects from their serialized form. It's crucial for consuming data from external sources.

```csharp
// Handling polymorphic deserialization
[JsonDerivedType(typeof(Employee), "employee")]
[JsonDerivedType(typeof(Manager), "manager")]
public abstract class Person { }

public class Employee : Person { public string Department { get; set; } }
public class Manager : Person { public int TeamSize { get; set; } }

string json = "{\"$type\":\"manager\",\"TeamSize\":5}";
var options = new JsonSerializerOptions
{
    PropertyNameCaseInsensitive = true
};
Person person = JsonSerializer.Deserialize<Person>(json, options);
```

### Schema Validation
Schema validation ensures that data conforms to a predefined structure and rules. While JSON Schema is common, .NET offers various validation approaches.

```csharp
using System.ComponentModel.DataAnnotations;

public class User
{
    [Required, MinLength(3)]
    public string Username { get; set; }
    
    [EmailAddress]
    public string Email { get; set; }
    
    [Range(18, 100)]
    public int Age { get; set; }
}

// Validation using DataAnnotations
var user = new User { Username = "ab", Email = "invalid", Age = 15 };
var results = new List<ValidationResult>();
bool isValid = Validator.TryValidateObject(user, new ValidationContext(user), results, true);

foreach (var validationResult in results)
{
    Console.WriteLine(validationResult.ErrorMessage);
}
```

### Content Negotiation
Content negotiation allows clients and servers to agree on the data format for communication. ASP.NET Core handles this automatically through the `Accept` and `Content-Type` headers.

```csharp
// Controller action supporting multiple formats
[ApiController]
public class UsersController : ControllerBase
{
    [HttpGet("users/{id}")]
    [Produces("application/json", "application/xml")] // Supported response formats
    public IActionResult GetUser(int id)
    {
        var user = _userService.GetUser(id);
        
        // ASP.NET Core automatically formats response based on Accept header
        return Ok(user);
    }
    
    [HttpPost("users")]
    [Consumes("application/json", "application/xml")] // Accepted request formats
    public IActionResult CreateUser([FromBody] User user)
    {
        // Framework deserializes based on Content-Type header
        _userService.CreateUser(user);
        return CreatedAtAction(nameof(GetUser), new { id = user.Id }, user);
    }
}
```

## Why is Data Formats important?

1. **Interoperability (Standardization Principle)**: Enables communication between heterogeneous systems by providing common data representation standards that transcend programming language boundaries.

2. **Separation of Concerns (Single Responsibility Principle)**: Decouples data storage/transmission logic from business logic, allowing each component to focus on its specific responsibility.

3. **Scalability through Loose Coupling**: Facilitates microservices architecture by allowing independent services to evolve their internal data structures while maintaining stable external contracts.

## Advanced Nuances

### 1. Polymorphic Serialization with System.Text.Json
Handling inheritance hierarchies requires careful configuration:

```csharp
// Advanced polymorphic serialization with type discriminators
var options = new JsonSerializerOptions
{
    PropertyNamingPolicy = JsonNamingPolicy.CamelCase,
    WriteIndented = true,
    Converters = { new JsonStringEnumConverter() }
};

// Custom type resolver for complex hierarchies
options.TypeInfoResolver = new DefaultJsonTypeInfoResolver
{
    Modifiers = { AddPolymorphicTypeInfo }
};
```

### 2. Streaming Serialization for Large Datasets
For memory efficiency with large data:

```csharp
// Stream-based serialization to avoid loading entire dataset into memory
public async Task SerializeLargeDatasetAsync(IEnumerable<Data> largeDataset, Stream outputStream)
{
    await JsonSerializer.SerializeAsync(outputStream, largeDataset, new JsonSerializerOptions
    {
        WriteIndented = false, // Better performance without formatting
        DefaultBufferSize = 16384 // Larger buffer for better throughput
    });
}
```

### 3. Custom Schema Validation with JSON Schema
Beyond DataAnnotations for complex validation scenarios:

```csharp
using JsonSchema.Net;

// Validate against JSON Schema for complex business rules
var schema = JsonSchema.FromType<ComplexBusinessObject>();
var validationResults = schema.Validate(instance);

// Custom validation logic combining multiple rules
if (!validationResults.IsValid)
{
    foreach (var error in validationResults)
    {
        // Handle complex validation errors with detailed paths
        Console.WriteLine($"Error at {error.Path}: {error.Message}");
    }
}
```

## How this fits the Roadmap

Within the "HTTP Methods and Data" section, **Data Formats** serves as the foundational layer that enables all subsequent HTTP communication. It's the prerequisite for:

- **RESTful API Design**: Understanding data formats is essential for designing proper request/response payloads
- **API Versioning**: Data format evolution strategies are crucial for maintaining backward compatibility
- **Message Queuing and Event-Driven Architectures**: Serialization formats form the basis for message payloads in systems like Azure Service Bus or RabbitMQ

This knowledge unlocks more advanced topics including:
- API contract design with OpenAPI/Swagger
- Protocol Buffers and gRPC for high-performance communication
- GraphQL query handling and response formatting
- Advanced content negotiation strategies for multi-format APIs

Mastery of data formats provides the essential vocabulary and tooling needed to design robust, scalable APIs that can evolve gracefully over time.