# Authorization in C#: A Comprehensive Guide

## What is Authorization?

Authorization (often called "access control") is the process of determining what an authenticated user is allowed to do within an application. While authentication answers "Who are you?", authorization answers "What are you allowed to do?". It solves the problem of ensuring that users can only access resources and perform actions that they have explicit permission for, preventing unauthorized data exposure and system manipulation.

## How it works in C#

### Role-Based Authorization
**Explanation:** Role-Based Access Control (RBAC) assigns permissions to roles, and then assigns roles to users. Users inherit permissions through their role membership, making management scalable for large user bases.

```csharp
// Controller-level role requirement
[Authorize(Roles = "Admin,Manager")]
public class AdminController : Controller
{
    // Action-level role requirement (more specific)
    [Authorize(Roles = "Admin")]
    public IActionResult DeleteUser(int userId)
    {
        // Only users with Admin role can access this
        return View();
    }
}

// Program.cs configuration
builder.Services.AddAuthorization(options =>
{
    // Define policy based on role
    options.AddPolicy("HRManager", policy => 
        policy.RequireRole("HR", "Manager"));
});

// Policy usage in controller
[Authorize(Policy = "HRManager")]
public IActionResult ViewSalaries() => View();
```

### Scope-Based Authorization
**Explanation:** Scope-based authorization uses OAuth2 scopes to limit what an API client can do. Commonly used in API scenarios where different clients need different levels of access to the same API.

```csharp
// Custom requirement for scope validation
public class ScopeRequirement : IAuthorizationRequirement
{
    public string Scope { get; }
    public ScopeRequirement(string scope) => Scope = scope;
}

// Handler for scope requirement
public class ScopeRequirementHandler : AuthorizationHandler<ScopeRequirement>
{
    protected override Task HandleRequirementAsync(
        AuthorizationHandlerContext context, 
        ScopeRequirement requirement)
    {
        // Check if user has the required scope
        var scopeClaim = context.User.FindFirst("scope");
        if (scopeClaim?.Value.Split(' ').Contains(requirement.Scope) == true)
        {
            context.Succeed(requirement);
        }
        return Task.CompletedTask;
    }
}

// Policy registration
builder.Services.AddAuthorization(options =>
{
    options.AddPolicy("ReadUsers", policy =>
        policy.Requirements.Add(new ScopeRequirement("users.read")));
});

// Usage in API controller
[Authorize(Policy = "ReadUsers")]
[HttpGet("users")]
public IActionResult GetUsers() => Ok(users);
```

### Resource-Based Authorization
**Explanation:** Resource-based authorization evaluates access rights based on the specific resource being accessed. This allows for fine-grained control where permissions depend on both the user and the resource's properties.

```csharp
// Custom requirement for resource-based auth
public class DocumentOwnerRequirement : IAuthorizationRequirement { }

// Resource-based authorization handler
public class DocumentAuthorizationHandler : 
    AuthorizationHandler<DocumentOwnerRequirement, Document>
{
    protected override Task HandleRequirementAsync(
        AuthorizationHandlerContext context,
        DocumentOwnerRequirement requirement,
        Document resource)
    {
        // Only allow access if user owns the document
        if (context.User.Identity?.Name == resource.OwnerId)
        {
            context.Succeed(requirement);
        }
        return Task.CompletedTask;
    }
}

// Registration
builder.Services.AddSingleton<IAuthorizationHandler, DocumentAuthorizationHandler>();
builder.Services.AddAuthorization(options =>
{
    options.AddPolicy("DocumentOwner", policy =>
        policy.Requirements.Add(new DocumentOwnerRequirement()));
});

// Usage in controller - resource passed to Authorize attribute
[Authorize(Policy = "DocumentOwner")]
public IActionResult EditDocument(int id)
{
    var document = _documentService.GetDocument(id);
    // The handler will automatically receive the document resource
    return View(document);
}
```

### Policy Enforcement
**Explanation:** Policy enforcement uses a flexible, claims-based system where requirements can be combined using complex logic. This is ASP.NET Core's primary authorization mechanism.

```csharp
// Complex policy with multiple requirements
builder.Services.AddAuthorization(options =>
{
    options.AddPolicy("SeniorDeveloper", policy =>
        policy
            .RequireAuthenticatedUser()
            .RequireClaim("Department", "Engineering")
            .RequireClaim("YearsOfExperience", "3", "5", "10")
            .RequireAssertion(context =>
            {
                // Custom logic - must not be on probation
                return !context.User.HasClaim("EmploymentStatus", "Probation");
            }));
});

// Policy with role and custom requirement combination
options.AddPolicy("ProjectLead", policy =>
    policy
        .RequireRole("SeniorDeveloper")
        .Requirements.Add(new ProjectAccessRequirement()));

// Manual policy evaluation in code
public async Task<IActionResult> SomeAction()
{
    var resource = await GetResourceAsync();
    var authResult = await _authorizationService
        .AuthorizeAsync(User, resource, "CustomPolicy");
    
    if (!authResult.Succeeded)
    {
        return Forbid();
    }
    return View(resource);
}
```

### ACLs (Access Control Lists)
**Explanation:** ACLs store permissions as a list attached to each resource, specifying which users/roles can perform which actions. Useful when each resource has unique access requirements.

```csharp
// ACL entity
public class ResourceAcl
{
    public int Id { get; set; }
    public int ResourceId { get; set; }
    public string UserId { get; set; }
    public string Permissions { get; set; } // "Read,Write,Delete"
}

// ACL-based authorization service
public class AclAuthorizationService
{
    private readonly ApplicationDbContext _context;
    
    public async Task<bool> CheckAccessAsync(string userId, int resourceId, string permission)
    {
        var acl = await _context.ResourceAcls
            .FirstOrDefaultAsync(a => a.ResourceId == resourceId && a.UserId == userId);
        
        return acl?.Permissions?.Split(',').Contains(permission) == true;
    }
}

// Custom authorization attribute using ACLs
public class AclAuthorizeAttribute : Attribute, IAsyncAuthorizationFilter
{
    private readonly string _permission;
    
    public AclAuthorizeAttribute(string permission) => _permission = permission;
    
    public async Task OnAuthorizationAsync(AuthorizationFilterContext context)
    {
        var aclService = context.HttpContext.RequestServices.GetService<AclAuthorizationService>();
        var resourceId = GetResourceIdFromRoute(context);
        var userId = context.HttpContext.User.Identity.Name;
        
        if (!await aclService.CheckAccessAsync(userId, resourceId, _permission))
        {
            context.Result = new ForbidResult();
        }
    }
    
    private int GetResourceIdFromRoute(AuthorizationFilterContext context)
    {
        // Extract resource ID from route parameters
        return int.Parse(context.RouteData.Values["id"].ToString());
    }
}

// Usage
[AclAuthorize("Write")]
public IActionResult EditDocument(int id) => View();
```

### Delegation
**Explanation:** Delegation allows users to grant their permissions to other users temporarily. Common in enterprise scenarios where users need to share access without involving administrators.

```csharp
// Delegation entity
public class PermissionDelegation
{
    public int Id { get; set; }
    public string FromUserId { get; set; }
    public string ToUserId { get; set; }
    public string ResourceType { get; set; }
    public int ResourceId { get; set; }
    public string Permissions { get; set; }
    public DateTime ExpiresAt { get; set; }
}

// Delegation-aware authorization handler
public class DelegationAwareAuthorizationHandler : 
    AuthorizationHandler<DocumentOwnerRequirement, Document>
{
    private readonly ApplicationDbContext _context;
    
    protected override async Task HandleRequirementAsync(
        AuthorizationHandlerContext context,
        DocumentOwnerRequirement requirement,
        Document resource)
    {
        var userId = context.User.Identity.Name;
        
        // Check direct ownership
        if (userId == resource.OwnerId)
        {
            context.Succeed(requirement);
            return;
        }
        
        // Check for valid delegation
        var delegation = await _context.PermissionDelegations
            .FirstOrDefaultAsync(d => 
                d.ToUserId == userId && 
                d.ResourceId == resource.Id &&
                d.ExpiresAt > DateTime.UtcNow);
        
        if (delegation != null)
        {
            context.Succeed(requirement);
        }
    }
}

// Delegation service for managing delegations
public class DelegationService
{
    public async Task<bool> DelegatePermissionsAsync(string fromUserId, DelegationRequest request)
    {
        // Validate that fromUserId has permissions to delegate
        // Create delegation record
        // Notify users, log activity, etc.
    }
}
```

## Why is Authorization important?

1. **Principle of Least Privilege** - Authorization ensures users have only the minimum permissions needed, reducing the attack surface and potential damage from compromised accounts.

2. **Separation of Concerns (from SOLID)** - By separating authorization logic from business logic, the system becomes more maintainable and follows the Single Responsibility Principle.

3. **Audit Trail Compliance** - Proper authorization enables comprehensive logging and monitoring, essential for regulatory compliance and security investigations.

## Advanced Nuances

### 1. Performance Optimization with Caching
```csharp
// Cached authorization service to reduce database hits
public class CachedAuthorizationService : IAuthorizationService
{
    private readonly IAuthorizationService _innerService;
    private readonly IMemoryCache _cache;
    
    public async Task<AuthorizationResult> AuthorizeAsync(
        ClaimsPrincipal user, object resource, string policyName)
    {
        var cacheKey = $"auth_{user.Identity.Name}_{resource?.GetHashCode()}_{policyName}";
        return await _cache.GetOrCreateAsync(cacheKey, async entry =>
        {
            entry.AbsoluteExpirationRelativeToNow = TimeSpan.FromMinutes(5);
            return await _innerService.AuthorizeAsync(user, resource, policyName);
        });
    }
}
```

### 2. Dynamic Policy Generation
```csharp
// Generate policies at runtime based on configuration
public class DynamicAuthorizationPolicyProvider : DefaultAuthorizationPolicyProvider
{
    private readonly IConfiguration _config;
    
    public override async Task<AuthorizationPolicy> GetPolicyAsync(string policyName)
    {
        // Check if policy is defined in configuration
        var policyConfig = _config.GetSection($"Policies:{policyName}");
        if (policyConfig.Exists())
        {
            var policyBuilder = new AuthorizationPolicyBuilder();
            
            // Build policy dynamically from configuration
            foreach (var requirement in policyConfig.GetChildren())
            {
                // Add requirements based on configuration
            }
            
            return policyBuilder.Build();
        }
        
        return await base.GetPolicyAsync(policyName);
    }
}
```

### 3. Cross-Cutting Authorization with Resource Transformers
```csharp
// Transform resources before authorization evaluation
public class ResourceTransformingAuthorizationMiddleware
{
    private readonly RequestDelegate _next;
    
    public async Task InvokeAsync(HttpContext context)
    {
        // Transform resource based on user context
        // Apply tenant isolation, data filtering, etc.
        await _next(context);
    }
}
```

## How this fits the Roadmap

Within the "Security and Best Practices" section, Authorization serves as a **foundational pillar** that enables more advanced security concepts. It's a prerequisite for understanding:

- **Advanced Authentication Flows** (OAuth2, OpenID Connect)
- **Secure API Design** (proper access control in RESTful services)
- **Data Protection** (encryption key management tied to authorization)
- **Compliance Frameworks** (GDPR, HIPAA implementation)

Authorization unlocks enterprise-grade patterns like **multi-tenant applications**, **delegated administration**, and **fine-grained audit systems**. Mastery of authorization concepts is essential for building scalable, secure applications that can handle complex business requirements while maintaining robust security postures.