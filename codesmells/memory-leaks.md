# Memory Leaks in C# - Advanced Resource Management Guide

## What is Memory Leaks?

A memory leak occurs when objects remain allocated in memory even though they are no longer needed or referenced by active application code. In C#, despite having automatic garbage collection, memory leaks persist when unwanted references prevent the Garbage Collector (GC) from reclaiming memory. This leads to increased memory consumption over time, degraded performance, and potential out-of-memory exceptions. The core problem is **unintentional object retention** - objects that should be collected remain alive due to hidden references.

## How Memory Leaks Work in C#

### Event Handler Removal

**Explanation:** Event handlers create strong references from event publishers to subscribers. If subscribers don't unsubscribe before disposal, publishers keep subscriber objects alive, preventing garbage collection. This is particularly problematic with long-lived publishers (like static events) and short-lived subscribers.

```csharp
public class EventPublisher
{
    public event EventHandler<string> MessageReceived;
    
    public void SendMessage(string message)
    {
        MessageReceived?.Invoke(this, message);
    }
}

public class EventSubscriber : IDisposable
{
    private readonly EventPublisher _publisher;
    private readonly string _name;
    
    public EventSubscriber(EventPublisher publisher, string name)
    {
        _publisher = publisher;
        _name = name;
        
        // SUBTLE LEAK: Subscriber registers but never unregisters
        _publisher.MessageReceived += OnMessageReceived;
    }
    
    private void OnMessageReceived(object sender, string message)
    {
        Console.WriteLine($"{_name} received: {message}");
    }
    
    public void Dispose()
    {
        // CRITICAL: Must unsubscribe to avoid leak
        _publisher.MessageReceived -= OnMessageReceived;
    }
}

// Usage that causes memory leak:
var publisher = new EventPublisher();
while (true)
{
    var subscriber = new EventSubscriber(publisher, "TempSubscriber");
    // subscriber falls out of scope but publisher keeps reference
    // GC cannot collect subscriber until publisher is collected
}
```

### Static Fields

**Explanation:** Static fields have application-domain lifetime - they're rooted references that GC never collects. When static fields reference instances, those instances also become effectively immortal, creating permanent memory leaks if misused.

```csharp
public class CacheService
{
    // DANGEROUS: Static dictionary lives for application lifetime
    private static readonly Dictionary<string, object> _cache = new();
    
    public void AddToCache(string key, object data)
    {
        _cache[key] = data; // Object remains until app domain unloads
    }
    
    // Even if we remove, if keys accumulate without cleanup...
    public bool RemoveFromCache(string key)
    {
        return _cache.Remove(key);
    }
}

public class LargeData
{
    public byte[] Data = new byte[1000000]; // 1MB object
}

// Leak scenario:
var cache = new CacheService();
for (int i = 0; i < 1000; i++)
{
    cache.AddToCache($"item_{i}", new LargeData());
    // Each LargeData remains in memory forever
    // Even if we remove some, unused keys or references may persist
}
```

### Weak References

**Explanation:** Weak references allow objects to be accessible while permitting garbage collection when memory pressure occurs. They're ideal for caching scenarios where recreating objects is acceptable but keeping them in memory is optional.

```csharp
public class WeakReferenceCache
{
    private readonly Dictionary<string, WeakReference<LargeData>> _cache = new();
    
    public void Add(string key, LargeData data)
    {
        _cache[key] = new WeakReference<LargeData>(data);
    }
    
    public LargeData Get(string key)
    {
        if (_cache.TryGetValue(key, out var weakRef) && 
            weakRef.TryGetTarget(out var data))
        {
            return data; // Object still in memory
        }
        
        // Object was collected - recreate or return null
        return null;
    }
    
    public void Cleanup()
    {
        var deadKeys = _cache.Where(kvp => !kvp.Value.TryGetTarget(out _))
                            .Select(kvp => kvp.Key).ToList();
        foreach (var key in deadKeys)
        {
            _cache.Remove(key);
        }
    }
}

// Proper usage - allows GC to reclaim memory when needed
var weakCache = new WeakReferenceCache();
var largeData = new LargeData();
weakCache.Add("temp", largeData);

// Later, if memory pressure occurs, GC can collect LargeData
// weakCache.Get("temp") may return null, which is acceptable for cache misses
```

### Profiling

**Explanation:** Memory profiling involves using tools to detect, diagnose, and fix memory leaks. Key techniques include analyzing object lifetimes, reference chains, and GC generation promotions.

```csharp
// Code instrumented for memory analysis
public class LeakyService : IDisposable
{
    private static List<LeakyService> _instances = new(); // Intentional leak for demo
    
    private byte[] _data = new byte[1000000]; // 1MB payload
    
    public LeakyService()
    {
        _instances.Add(this); // LEAK: Static reference to all instances
    }
    
    public void Dispose()
    {
        // Should remove from static list but often forgotten
        // _instances.Remove(this); // FIX: Uncomment to prevent leak
    }
}

// Profiling demonstration
public class MemoryProfilerDemo
{
    public static void DemonstrateLeak()
    {
        var weakRefs = new List<WeakReference>();
        
        for (int i = 0; i < 100; i++)
        {
            var service = new LeakyService();
            weakRefs.Add(new WeakReference(service));
            
            // Proper disposal missing - service should be disposed
            // service.Dispose(); // Required to prevent leak
        }
        
        GC.Collect(); // Force GC to see what remains
        
        var aliveCount = weakRefs.Count(ref => ref.IsAlive);
        Console.WriteLine($"Objects still alive: {aliveCount}"); // Will show 100 - all leaked
    }
}
```

## Why is Memory Leaks Important?

1. **Scalability Principle**: Proper memory management enables applications to handle increasing workloads without degradation, as uncontrolled memory growth limits horizontal and vertical scaling capabilities.

2. **Single Responsibility Principle**: Each component should manage its own resource lifecycle; understanding leaks ensures classes properly handle their dependencies and references.

3. **Predictable Performance**: Following deterministic resource cleanup patterns (like Dispose pattern) ensures consistent application behavior under varying load conditions.

## Advanced Nuances

### 1. Conditional Weak References for Dependency Tracking

ConditionalWeakTable<TKey, TValue> creates references that don't prevent garbage collection but maintain key-value associations as long as the key is alive:

```csharp
public class AttachmentService
{
    // Advanced: Associates data with objects without affecting their lifetime
    private readonly ConditionalWeakTable<object, AttachmentMetadata> _attachments = new();
    
    public void AttachMetadata(object target, AttachmentMetadata metadata)
    {
        _attachments.Add(target, metadata);
        // When 'target' is collected, the entry is automatically removed
    }
    
    public AttachmentMetadata GetMetadata(object target)
    {
        _attachments.TryGetValue(target, out var metadata);
        return metadata;
    }
}
```

### 2. Event Handler Leaks in Asynchronous Contexts

Async event handlers can create subtle leaks when exceptions occur during unsubscribe:

```csharp
public class AsyncEventService
{
    public event Func<string, Task> MessageProcessed;
    
    public async Task<bool> UnsubscribeSafely(string handlerId)
    {
        var handler = MessageProcessed;
        if (handler == null) return true;
        
        // Safe unsubscribe: handle each delegate individually
        var delegates = handler.GetInvocationList();
        bool allRemoved = true;
        
        foreach (Func<string, Task> del in delegates)
        {
            try
            {
                MessageProcessed -= del;
            }
            catch (Exception ex)
            {
                // Log but continue - don't let one failure prevent others
                allRemoved = false;
            }
        }
        
        return allRemoved;
    }
}
```

### 3. Finalizer-Induced Leaks

Misimplemented finalizers can resurrect objects or delay collection:

```csharp
public class ResourceHolder : IDisposable
{
    private byte[] _resource = new byte[1000000];
    private static List<ResourceHolder> _finalizerQueue = new();
    
    ~ResourceHolder() // Finalizer
    {
        // DANGEROUS: Adding to static collection in finalizer
        _finalizerQueue.Add(this);
        
        // This resurrects the object and prevents collection
        // GC.ReRegisterForFinalize(this); // Even worse - infinite resurrection
    }
    
    public void Dispose()
    {
        GC.SuppressFinalize(this); // Proper disposal prevents finalizer leak
    }
}
```

## How This Fits the Roadmap

Within the "Resource Management" section, Memory Leaks serves as a **foundational diagnostic skill** that underpins more advanced topics. It's the prerequisite for:

- **Memory Optimization Patterns**: Understanding leaks enables learning object pooling, flyweight patterns, and allocation reduction techniques
- **Advanced GC Tuning**: Knowledge of leak patterns informs GC configuration decisions and generation-specific optimizations
- **Performance Critical Applications**: Essential for real-time systems, high-load servers, and memory-constrained environments

This concept unlocks **proactive memory management** - transitioning from reactive leak-fixing to designing systems with leak prevention built-in, which is crucial for the subsequent "High-Performance C#" and "System Architecture" sections of the mastery roadmap.