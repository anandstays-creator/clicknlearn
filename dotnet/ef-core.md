# Entity Framework Core Explained

## What is Entity Framework Core?

Entity Framework Core (EF Core) is an open-source, lightweight, extensible Object-Relational Mapper (ORM) for .NET. Common aliases include EF Core, EF7+, and previously Entity Framework. It serves as a bridge between your C# object model and your database, eliminating the need for most data-access code developers typically write. EF Core solves the **impedance mismatch** between object-oriented programming and relational databases by allowing developers to work with data using C# objects rather than direct database interactions.

## How it works in C#

### Model First

**Explanation:** In Model First approach, you design your entity model visually using EF Core's designer tools, then generate both the database schema and C# entity classes from this model. While more common in older Entity Framework versions, EF Core supports similar workflows through reverse engineering or scaffolding.

```csharp
// After designing model visually, EF Core generates entities like:
public class Blog
{
    public int BlogId { get; set; }
    public string Url { get; set; }
    
    // Navigation property - generated from model relationships
    public List<Post> Posts { get; set; } = new List<Post>();
}

public class Post
{
    public int PostId { get; set; }
    public string Title { get; set; }
    public string Content { get; set; }
    
    // Foreign key relationship
    public int BlogId { get; set; }
    public Blog Blog { get; set; }
}

// DbContext is also generated
public class BloggingContext : DbContext
{
    public DbSet<Blog> Blogs { get; set; }
    public DbSet<Post> Posts { get; set; }
}
```

### Database First

**Explanation:** Database First starts with an existing database schema. EF Core scaffolding tools reverse-engineer the database to generate corresponding C# entity classes and DbContext. This is ideal when working with legacy databases.

```csharp
// Command to scaffold from existing database:
// dotnet ef dbcontext scaffold "ConnectionString" Microsoft.EntityFrameworkCore.SqlServer

// Generated entity from existing Products table
public partial class Product
{
    public Product()
    {
        OrderDetails = new HashSet<OrderDetail>();
    }

    public int ProductId { get; set; }
    public string ProductName { get; set; }
    public decimal? UnitPrice { get; set; }
    public int? SupplierId { get; set; }

    public virtual Supplier Supplier { get; set; }
    public virtual ICollection<OrderDetail> OrderDetails { get; set; }
}

// Partial class to add customizations without losing scaffolded code
public partial class Product
{
    public bool IsExpensive => UnitPrice > 100;
}
```

### Code First Migrations

**Explanation:** Code First is EF Core's predominant approach. You define entities and relationships in C# code, then use migrations to create and update the database schema. Migrations track schema changes over time.

```csharp
// Define entities in code
public class Order
{
    public int OrderId { get; set; }
    public DateTime OrderDate { get; set; }
    public List<OrderItem> Items { get; set; } = new List<OrderItem>();
}

// Create migration
// dotnet ef migrations add InitialCreate
// dotnet ef database update

// Migration file generated:
public partial class InitialCreate : Migration
{
    protected override void Up(MigrationBuilder migrationBuilder)
    {
        migrationBuilder.CreateTable(
            name: "Orders",
            columns: table => new
            {
                OrderId = table.Column<int>(type: "int", nullable: false)
                    .Annotation("SqlServer:Identity", "1, 1"),
                OrderDate = table.Column<DateTime>(type: "datetime2", nullable: false)
            },
            constraints: table =>
            {
                table.PrimaryKey("PK_Orders", x => x.OrderId);
            });
    }

    protected override void Down(MigrationBuilder migrationBuilder)
    {
        migrationBuilder.DropTable(name: "Orders");
    }
}
```

### Querying Data

**Explanation:** EF Core uses LINQ (Language Integrated Query) to translate C# queries into SQL. It supports eager loading, explicit loading, and lazy loading for related data.

```csharp
using var context = new BloggingContext();

// Basic LINQ query
var blogs = context.Blogs
    .Where(b => b.Rating > 3)
    .OrderBy(b => b.Url)
    .ToList();

// Eager loading with Include
var blogsWithPosts = context.Blogs
    .Include(b => b.Posts)  // LEFT JOIN in SQL
    .ThenInclude(p => p.Tags) // Multiple levels
    .ToList();

// Projection (select specific columns)
var blogTitles = context.Blogs
    .Select(b => new { b.BlogId, b.Url })
    .ToList();

// Raw SQL queries when needed
var highRatedBlogs = context.Blogs
    .FromSqlRaw("SELECT * FROM Blogs WHERE Rating > 5")
    .ToList();
```

### Saving Data

**Explanation:** EF Core tracks changes to entities and batches database commands for efficient saving. It supports adding, updating, and deleting entities through the DbContext.

```csharp
using var context = new BloggingContext();

// Adding new entity
var blog = new Blog { Url = "http://example.com" };
context.Blogs.Add(blog);

// Updating existing entity
var existingBlog = context.Blogs.Find(1);
existingBlog.Url = "http://updated.com";

// Deleting entity
var blogToDelete = context.Blogs.Find(2);
context.Blogs.Remove(blogToDelete);

// Batch operations with transaction
using var transaction = context.Database.BeginTransaction();
try
{
    context.SaveChanges(); // Executes all changes in single round-trip
    transaction.Commit();
}
catch
{
    transaction.Rollback();
    throw;
}
```

### Relationships

**Explanation:** EF Core supports various relationship types (one-to-one, one-to-many, many-to-many) through navigation properties and foreign keys. Relationships can be configured using Fluent API or data annotations.

```csharp
// Entity definitions with relationships
public class Author
{
    public int AuthorId { get; set; }
    public string Name { get; set; }
    public List<Book> Books { get; set; } // One-to-many
}

public class Book
{
    public int BookId { get; set; }
    public string Title { get; set; }
    public int AuthorId { get; set; } // Foreign key
    public Author Author { get; set; } // Navigation property
}

// Configuring relationships with Fluent API
protected override void OnModelCreating(ModelBuilder modelBuilder)
{
    // One-to-many relationship
    modelBuilder.Entity<Book>()
        .HasOne(b => b.Author)
        .WithMany(a => a.Books)
        .HasForeignKey(b => b.AuthorId);

    // Many-to-many (EF Core 5+)
    modelBuilder.Entity<Book>()
        .HasMany(b => b.Tags)
        .WithMany(t => t.Books);
}

// Using relationships in queries
var authorsWithBooks = context.Authors
    .Include(a => a.Books)
    .ThenInclude(b => b.Tags)
    .ToList();
```

## Why is Entity Framework Core important?

1. **DRY Principle**: Eliminates repetitive data access code by automatically handling CRUD operations, reducing boilerplate and maintenance overhead.

2. **Repository Pattern Implementation**: Provides built-in implementation of repository and unit of work patterns, promoting clean separation of concerns in application architecture.

3. **Database Abstraction**: Enables database portability and scalability by abstracting specific SQL dialects, allowing easier switching between database providers.

## Advanced Nuances

1. **Change Tracking Strategies**: EF Core offers different tracking approaches - snapshot (default), changed tracking with notification entities (INotifyPropertyChanged), and no-tracking for read-only scenarios. Understanding when to use `AsNoTracking()` can significantly improve performance.

```csharp
// No-tracking for read-heavy scenarios
var readOnlyBlogs = context.Blogs
    .AsNoTracking()
    .Where(b => b.Rating > 3)
    .ToList();
```

2. **Owned Types and Table Splitting**: Advanced relationship modeling where owned entities are stored in the same table as the owner, useful for complex value objects.

```csharp
modelBuilder.Entity<Order>().OwnsOne(o => o.ShippingAddress);
```

3. **Global Query Filters**: Apply automatic filters to all queries, ideal for multi-tenancy or soft delete scenarios.

```csharp
modelBuilder.Entity<Blog>().HasQueryFilter(b => !b.IsDeleted);
```

## How this fits the Roadmap

Within the "Data Access" section of the Advanced C# Mastery roadmap, EF Core serves as the **foundational ORM technology**. It's a prerequisite for more advanced topics like:

- **Performance Optimization**: Learning query optimization, batching strategies, and when to bypass EF Core for raw SQL
- **Repository and Unit of Work Patterns**: Understanding how to properly abstract EF Core for testability and maintainability
- **Domain-Driven Design**: Implementing aggregates, value objects, and domain events using EF Core
- **Microservices Data Patterns**: Understanding database-per-service vs. shared database strategies
- **Advanced CQRS**: Implementing command and query separation with EF Core

EF Core mastery unlocks the ability to design sophisticated data access layers that scale efficiently while maintaining clean architecture principles.