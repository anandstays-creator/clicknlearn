# Primitive Obsession in C# - Advanced Mastery Guide

## What is Primitive Obsession?

Primitive Obsession (also known as "Stringly Typed" code) is an anti-pattern where primitive data types like `string`, `int`, or `bool` are overused to represent domain concepts, rather than creating dedicated types. The core purpose of addressing this obsession is to create more expressive, type-safe, and maintainable code by modeling domain concepts with rich objects instead of primitive wrappers.

The problem it solves: when developers use primitives for everything, they lose compile-time safety, domain expressiveness, and encapsulation of business rules—leading to validation logic scattered throughout the codebase and subtle bugs that could be caught by the type system.

## How it works in C#

### Value Objects

**Explanation**: Value objects are immutable types that represent a concept in your domain and are defined by their attributes rather than identity. They encapsulate validation rules and operations related to that concept.

```csharp
public record EmailAddress
{
    public string Value { get; }

    public EmailAddress(string value)
    {
        if (string.IsNullOrWhiteSpace(value) || !value.Contains('@'))
            throw new ArgumentException("Invalid email format");

        Value = value.Trim().ToLowerInvariant();
    }

    // Implicit conversion for convenience
    public static implicit operator string(EmailAddress email) => email.Value;
    
    // Explicit conversion with validation
    public static explicit operator EmailAddress(string value) => new(value);

    public override string ToString() => Value;
}

// Usage
public class User
{
    public EmailAddress Email { get; }  // Instead of string Email
    
    public User(EmailAddress email)
    {
        Email = email;
    }
}

// Compile-time safety - no invalid emails can be created
var user = new User(new EmailAddress("valid@email.com"));
```

### Enumerations

**Explanation**: Enumerations provide type-safe alternatives to magic strings or numbers by defining a closed set of possible values. Smart enums take this further by adding behavior.

```csharp
// Basic enum - better than string literals
public enum OrderStatus
{
    Pending = 1,
    Confirmed = 2,
    Shipped = 3,
    Delivered = 4
}

// Smart enum with behavior - advanced pattern
public abstract class OrderStatusSmart
{
    public static readonly OrderStatusSmart Pending = new PendingStatus();
    public static readonly OrderStatusSmart Confirmed = new ConfirmedStatus();
    
    public int Id { get; }
    public string Name { get; }
    
    protected OrderStatusSmart(int id, string name)
    {
        Id = id;
        Name = name;
    }
    
    public abstract bool CanTransitionTo(OrderStatusSmart newStatus);
    
    private class PendingStatus : OrderStatusSmart
    {
        public PendingStatus() : base(1, "Pending") { }
        
        public override bool CanTransitionTo(OrderStatusSmart newStatus)
            => newStatus == Confirmed;
    }
    
    private class ConfirmedStatus : OrderStatusSmart
    {
        public ConfirmedStatus() : base(2, "Confirmed") { }
        
        public override bool CanTransitionTo(OrderStatusSmart newStatus)
            => newStatus == OrderStatusSmart.Pending; // Example rule
    }
}

// Usage - no magic strings, compile-time safety
var status = OrderStatusSmart.Pending;
if (status.CanTransitionTo(OrderStatusSmart.Confirmed))
{
    // Valid transition
}
```

### Domain Models

**Explanation**: Domain models use rich objects instead of primitives to encapsulate business logic and maintain invariants. They represent real-world concepts with behavior, not just data.

```csharp
public class BankAccount
{
    public AccountNumber Number { get; }        // Not string
    public Money Balance { get; private set; }  // Not decimal
    public AccountStatus Status { get; private set; }

    public BankAccount(AccountNumber number, Money initialBalance)
    {
        Number = number;
        Balance = initialBalance;
        Status = AccountStatus.Active;
    }

    public void Withdraw(Money amount)
    {
        if (Status != AccountStatus.Active)
            throw new InvalidOperationException("Account is not active");
            
        if (amount > Balance)
            throw new InvalidOperationException("Insufficient funds");
            
        Balance -= amount;
    }
}

// Supporting value objects
public record AccountNumber(string Value)
{
    public static implicit operator string(AccountNumber number) => number.Value;
}

public record Money(decimal Amount, string Currency)
{
    public static Money operator -(Money a, Money b)
    {
        if (a.Currency != b.Currency)
            throw new InvalidOperationException("Currency mismatch");
            
        return new Money(a.Amount - b.Amount, a.Currency);
    }
}
```

### DTOs

**Explanation**: While DTOs are naturally primitive-heavy for data transfer, we can still apply Primitive Obsession principles by using value objects in DTO properties and implementing proper mapping.

```csharp
// Domain model with rich types
public class Product
{
    public ProductId Id { get; }           // Value object
    public ProductName Name { get; }       // Value object
    public Money Price { get; }            // Value object
}

// DTO with primitive obsession remediation
public class ProductDto
{
    public string Id { get; set; }         // Still string for serialization
    public string Name { get; set; }
    public decimal Price { get; set; }
    public string Currency { get; set; }
}

// Mapping layer with validation
public static class ProductMapping
{
    public static Product ToDomain(this ProductDto dto)
    {
        return new Product(
            id: new ProductId(dto.Id),
            name: new ProductName(dto.Name),
            price: new Money(dto.Price, dto.Currency)
        );
    }

    public static ProductDto ToDto(this Product product)
    {
        return new ProductDto
        {
            Id = product.Id.Value,
            Name = product.Name.Value,
            Price = product.Price.Amount,
            Currency = product.Price.Currency
        };
    }
}

// Usage - clean separation with validation
var dto = new ProductDto { Id = "prod-123", Name = "Widget", Price = 29.99m, Currency = "USD" };
var product = dto.ToDomain(); // Validation happens here
```

## Why is Primitive Obsession important?

**1. Enhanced Type Safety (SOLID - Liskov Substitution)**: By creating specific types for domain concepts, the compiler catches invalid assignments and operations at compile time rather than runtime, adhering to the type safety principles in SOLID.

**2. Domain Logic Encapsulation (Domain-Driven Design)**: Value objects and rich domain models encapsulate business rules and validation logic in one place, following the Single Responsibility Principle and eliminating duplication.

**3. Improved Code Expressiveness (Ubiquitous Language)**: Code reads like the domain language, making it more maintainable and reducing cognitive load for developers working with the codebase.

## Advanced Nuances

**1. Performance Considerations with Many Small Objects**: While value objects improve design, creating thousands of small objects can impact performance in high-throughput scenarios. Solutions include:
- Using `readonly struct` for high-frequency value objects
- Implementing object pooling for frequently created types
- Using source generators for boilerplate code

```csharp
public readonly struct HighFrequencyValueObject
{
    private readonly string _value;
    
    public HighFrequencyValueObject(string value)
    {
        // Validation logic
        _value = value;
    }
}
```

**2. Serialization Challenges**: Value objects can complicate serialization/deserialization. Advanced patterns include:
- Custom JSON converters for System.Text.Json
- Type converters for XML serialization
- Factory methods with error handling instead of constructors

```csharp
[JsonConverter(typeof(EmailAddressConverter))]
public record EmailAddress(string Value)
{
    public static EmailAddress Create(string value) => 
        new(value); // Centralized creation with validation
}

public class EmailAddressConverter : JsonConverter<EmailAddress>
{
    public override EmailAddress Read(ref Utf8JsonReader reader, 
        Type typeToConvert, JsonSerializerOptions options) =>
        EmailAddress.Create(reader.GetString());
}
```

**3. Entity Framework Integration**: Mapping value objects to database primitives requires advanced EF configurations:
- Value conversions for simple mappings
- Owned entity types for complex value objects
- Custom value comparers for proper change tracking

```csharp
modelBuilder.Entity<BankAccount>()
    .OwnsOne(ba => ba.Balance, owned =>
    {
        owned.Property(m => m.Amount).HasColumnName("BalanceAmount");
        owned.Property(m => m.Currency).HasColumnName("BalanceCurrency");
    });
```

## How this fits the Roadmap

Within the "Logic Complexity" section of the Advanced C# Mastery roadmap, Primitive Obsession serves as a foundational concept that precedes more advanced topics. It's the prerequisite for understanding:

**What it builds upon**: Basic C# type system knowledge, object-oriented principles, and simple domain modeling.

**What it unlocks**: 
- **Domain-Driven Design patterns**: Entities, aggregates, and repositories all rely on proper domain modeling
- **Architectural patterns**: Clean Architecture and Vertical Slice Architecture benefit from rich domain models
- **Advanced testing strategies**: Value objects enable more precise unit testing and property-based testing
- **Event sourcing**: Strong typing is crucial for event schema evolution and versioning

Mastering Primitive Obsession equips you to tackle complex business domains with confidence, creating systems that are both technically robust and aligned with business requirements.