# Feature Envy in C#: Mastering Method Responsibility

## What is Feature Envy?

**Feature Envy** (also known as "Inappropriate Intimacy" or "Data-Driven Method") occurs when a method in one class is more interested in the data of another class than its own. This code smell violates encapsulation by having methods that manipulate external data more frequently than their own class's data. The core purpose of identifying Feature Envy is to correctly assign responsibilities and maintain proper data locality within object-oriented design.

## How it works in C#

### Method Movement

**Explanation**: Method movement involves relocating a method from a class where it primarily uses another class's data to the class that actually owns that data. This refactoring technique ensures that methods operate on the data they're naturally associated with.

**C# Example**:
```csharp
// BEFORE: Feature Envy - OrderProcessor is too interested in Order's data
public class Order
{
    public decimal Subtotal { get; set; }
    public decimal TaxRate { get; set; }
    public List<OrderItem> Items { get; set; }
}

public class OrderProcessor
{
    public decimal CalculateTotal(Order order)
    {
        // This method is envious of Order's data - it should live in Order
        decimal tax = order.Subtotal * order.TaxRate;
        return order.Subtotal + tax;
    }
    
    public bool IsEligibleForDiscount(Order order)
    {
        return order.Items.Count > 5; // Again, using Order's data extensively
    }
}

// AFTER: Method moved to proper class
public class Order
{
    public decimal Subtotal { get; set; }
    public decimal TaxRate { get; set; }
    public List<OrderItem> Items { get; set; }
    
    // Methods moved to where the data lives
    public decimal CalculateTotal()
    {
        decimal tax = Subtotal * TaxRate;
        return Subtotal + tax;
    }
    
    public bool IsEligibleForDiscount()
    {
        return Items.Count > 5;
    }
}

public class OrderProcessor
{
    // Now focuses on processing logic, not data calculation
    public void ProcessOrder(Order order)
    {
        decimal total = order.CalculateTotal(); // Delegate to Order
        // Processing logic here
    }
}
```

### Data Localization

**Explanation**: Data localization ensures that data and the operations on that data are co-located within the same class. This principle prevents classes from becoming "data bags" while other classes handle their business logic.

**C# Example**:
```csharp
// BEFORE: Data separated from behavior
public class CustomerData
{
    public string Name { get; set; }
    public DateTime JoinDate { get; set; }
    public decimal Balance { get; set; }
}

public class CustomerService
{
    public bool IsPremiumCustomer(CustomerData data)
    {
        // Feature envy - should be with the data
        return DateTime.Now.Year - data.JoinDate.Year >= 2 && data.Balance > 1000;
    }
    
    public string GetCustomerStatus(CustomerData data)
    {
        if (data.Balance < 0) return "Delinquent";
        return data.Balance > 500 ? "Preferred" : "Standard";
    }
}

// AFTER: Data and behavior localized together
public class Customer
{
    public string Name { get; set; }
    public DateTime JoinDate { get; set; }
    public decimal Balance { get; set; }
    
    // Behavior localized with data
    public bool IsPremiumCustomer()
    {
        return DateTime.Now.Year - JoinDate.Year >= 2 && Balance > 1000;
    }
    
    public string GetCustomerStatus()
    {
        if (Balance < 0) return "Delinquent";
        return Balance > 500 ? "Preferred" : "Standard";
    }
}

public class CustomerService
{
    // Now focuses on service-level operations
    public void UpdateCustomerStatus(Customer customer)
    {
        // Service logic that uses customer's own methods
        string status = customer.GetCustomerStatus();
        // Additional service operations
    }
}
```

### Domain Logic Placement

**Explanation**: Domain logic placement involves ensuring that business rules and domain-specific logic reside in the appropriate domain entities rather than service classes or infrastructure layers. This maintains the domain model's richness and prevents anemic domain models.

**C# Example**:
```csharp
// BEFORE: Domain logic in service layer (Anemic Domain Model)
public class Product
{
    public string Name { get; set; }
    public decimal Price { get; set; }
    public int StockQuantity { get; set; }
    public bool IsActive { get; set; }
}

public class ProductService
{
    public bool CanSellProduct(Product product, int quantity)
    {
        // Feature envy - domain logic leaked into service
        return product.IsActive && product.StockQuantity >= quantity && quantity > 0;
    }
    
    public decimal CalculateDiscountedPrice(Product product, decimal discountRate)
    {
        return product.Price * (1 - discountRate);
    }
}

// AFTER: Domain logic properly placed in domain entity
public class Product
{
    public string Name { get; set; }
    public decimal Price { get; set; }
    public int StockQuantity { get; set; }
    public bool IsActive { get; set; }
    
    // Domain logic belongs to the product entity
    public bool CanSell(int quantity)
    {
        return IsActive && StockQuantity >= quantity && quantity > 0;
    }
    
    public decimal CalculateDiscountedPrice(decimal discountRate)
    {
        if (discountRate < 0 || discountRate > 1)
            throw new ArgumentException("Invalid discount rate");
            
        return Price * (1 - discountRate);
    }
    
    public void Sell(int quantity)
    {
        if (!CanSell(quantity))
            throw new InvalidOperationException("Cannot sell product");
            
        StockQuantity -= quantity;
    }
}

public class ProductService
{
    // Service now handles coordination, not domain logic
    public SalesResult SellProduct(Product product, int quantity)
    {
        product.Sell(quantity); // Domain logic is in Product
        // Handle transaction, logging, etc.
        return new SalesResult();
    }
}
```

## Why is Feature Envy Important?

1. **Single Responsibility Principle (SOLID)**: Ensures each class has one reason to change by keeping data and behavior together, making systems more maintainable.

2. **Enhanced Encapsulation**: Protects internal data representation by co-locating data with operations, reducing coupling between classes.

3. **Improved Testability**: Isolates behavior with its data, enabling focused unit testing without complex mock setups.

## Advanced Nuances

### 1. Strategic Feature Envy in Visitor Pattern
Sometimes, Feature Envy is intentionally used in patterns like Visitor, where behavior is externalized to handle cross-cutting concerns:

```csharp
// Intentional Feature Envy for double dispatch
public interface IProductVisitor
{
    void Visit(BookProduct book);  // Envious of BookProduct's data
    void Visit(ElectronicsProduct electronics);
}

public class DiscountCalculator : IProductVisitor
{
    public decimal Discount { get; private set; }
    
    public void Visit(BookProduct book)
    {
        // This envy is intentional for the pattern
        Discount = book.Price * 0.1m; // Books get 10% discount
    }
    
    public void Visit(ElectronicsProduct electronics)
    {
        Discount = electronics.Price * 0.05m; // Electronics get 5%
    }
}
```

### 2. Data Transfer Objects (DTOs) Exception
DTOs are designed to be "data bags" without behavior, creating legitimate cases where other classes will exhibit Feature Envy:

```csharp
public class ProductDto // Pure data transfer
{
    public string Name { get; set; }
    public decimal Price { get; set; }
}

public class ProductViewModelBuilder
{
    // Legitimate Feature Envy - DTOs shouldn't have behavior
    public ProductViewModel Build(ProductDto dto)
    {
        return new ProductViewModel 
        { 
            DisplayName = dto.Name.ToUpper(),
            FormattedPrice = $"{dto.Price:C}"
        };
    }
}
```

### 3. Boundaries between Architectural Layers
Feature Envy can indicate improper layer boundaries, helping identify when logic should move between presentation, business, and data layers.

## How this fits the Roadmap

Within the "Encapsulation Issues" section, Feature Envy serves as a **fundamental diagnostic tool** for identifying improper responsibility assignment. It's a prerequisite for understanding more advanced concepts like:

- **Law of Demeter**: Feature Envy often violates this principle by reaching deeply into foreign objects
- **Anemic Domain Model**: The antithesis of proper Feature Envy resolution
- **Domain-Driven Design**: Proper domain logic placement is essential for rich domain models

Mastering Feature Envy unlocks deeper understanding of **design patterns** and **architectural principles**, particularly those related to maintaining clean boundaries between components and ensuring sustainable, scalable codebases.