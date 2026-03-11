# Concurrency Hazards in C#

## What is Concurrency Hazards?

Concurrency hazards refer to unexpected behaviors and errors that occur when multiple threads access shared resources simultaneously without proper synchronization. These include race conditions, deadlocks, livelocks, and resource contention issues that can lead to data corruption, inconsistent state, or application hangs.

## How Concurrency Hazards Work in C#

### Task Patterns for Hazard Prevention

**Safe Task Continuation Patterns**:
```csharp
public class SafeTaskPatterns
{
    private readonly CancellationTokenSource _cts = new();
    
    // Proper async/await pattern with cancellation
    public async Task ProcessDataSafelyAsync(IEnumerable<Data> data)
    {
        // Use ConfigureAwait(false) to avoid deadlocks in UI contexts
        var tasks = data.Select(async item =>
        {
            // Check cancellation token periodically
            _cts.Token.ThrowIfCancellationRequested();
            
            // Simulate async work
            await ProcessItemAsync(item).ConfigureAwait(false);
        });
        
        // Safe exception handling with WhenAll
        try
        {
            await Task.WhenAll(tasks);
        }
        catch (OperationCanceledException)
        {
            // Handle cancellation gracefully
            Console.WriteLine("Operation was cancelled");
        }
        catch (AggregateException ae)
        {
            // Flatten and handle multiple exceptions
            ae.Flatten().Handle(ex => ex is not OperationCanceledException);
        }
    }
    
    private async Task ProcessItemAsync(Data item)
    {
        // Async work with proper resource disposal
        await using var resource = await AcquireResourceAsync();
        await resource.ProcessAsync(item);
    }
}
```

### Lock Management Strategies

**Fine-grained Locking with Monitor**:
```csharp
public class FineGrainedLocking
{
    private readonly object _lockObject = new();
    private readonly Dictionary<string, int> _sharedDictionary = new();
    
    public void SafeIncrement(string key)
    {
        // Double-check locking pattern with proper memory barriers
        if (!_sharedDictionary.ContainsKey(key))
        {
            // Use Monitor instead of lock for more control
            bool lockTaken = false;
            try
            {
                Monitor.Enter(_lockObject, ref lockTaken);
                
                // Re-check after acquiring lock
                if (!_sharedDictionary.ContainsKey(key))
                {
                    _sharedDictionary[key] = 0;
                }
            }
            finally
            {
                if (lockTaken) Monitor.Exit(_lockObject);
            }
        }
        
        // Increment safely
        lock (_lockObject)
        {
            _sharedDictionary[key]++;
        }
    }
    
    // Reader-writer lock for better concurrency
    private readonly ReaderWriterLockSlim _rwLock = new();
    
    public int SafeReadWithWriteLock(string key)
    {
        // Upgradeable read pattern
        _rwLock.EnterUpgradeableReadLock();
        try
        {
            if (_sharedDictionary.TryGetValue(key, out var value))
            {
                return value;
            }
            
            // Upgrade to write lock if needed
            _rwLock.EnterWriteLock();
            try
            {
                _sharedDictionary[key] = 0;
                return 0;
            }
            finally
            {
                _rwLock.ExitWriteLock();
            }
        }
        finally
        {
            _rwLock.ExitUpgradeableReadLock();
        }
    }
}
```

### Thread-Safe Collections

**Immutable and Concurrent Collections**:
```csharp
public class ThreadSafeCollectionsExample
{
    // Immutable collections for read-heavy scenarios
    private ImmutableDictionary<string, int> _immutableData = 
        ImmutableDictionary<string, int>.Empty;
    
    // Concurrent collections for mixed read/write
    private readonly ConcurrentDictionary<string, int> _concurrentData = new();
    private readonly ConcurrentQueue<WorkItem> _workQueue = new();
    private readonly BlockingCollection<WorkItem> _blockingQueue;
    
    public ThreadSafeCollectionsExample()
    {
        _blockingQueue = new BlockingCollection<WorkItem>(
            new ConcurrentQueue<WorkItem>());
    }
    
    public void ProducerConsumerPattern()
    {
        // Producer
        Task.Run(() =>
        {
            for (int i = 0; i < 100; i++)
            {
                var workItem = new WorkItem($"Item_{i}");
                _blockingQueue.Add(workItem);
                
                // Safe addition to concurrent dictionary
                _concurrentData.AddOrUpdate(workItem.Id, 1, (key, old) => old + 1);
            }
            _blockingQueue.CompleteAdding();
        });
        
        // Multiple consumers
        var consumers = Enumerable.Range(0, 4).Select(_ => Task.Run(() =>
        {
            foreach (var workItem in _blockingQueue.GetConsumingEnumerable())
            {
                ProcessWorkItem(workItem);
            }
        }));
        
        Task.WhenAll(consumers).Wait();
    }
    
    // Safe atomic operations
    public bool TryUpdateSafely(string key, int newValue)
    {
        // Compare-and-swap pattern
        while (true)
        {
            if (_concurrentData.TryGetValue(key, out int currentValue))
            {
                if (_concurrentData.TryUpdate(key, newValue, currentValue))
                {
                    return true;
                }
                // Retry if value changed between read and update
            }
            else
            {
                return false;
            }
        }
    }
    
    // Immutable collection updates
    public void UpdateImmutableData(string key, int value)
    {
        // Atomic update with interlocked pattern
        var before = _immutableData;
        var after = before.SetItem(key, value);
        
        // Compare-exchange for immutable updates
        if (Interlocked.CompareExchange(ref _immutableData, after, before) != before)
        {
            // Retry logic if another thread modified the reference
            UpdateImmutableData(key, value);
        }
    }
}
```

### Deadlock Prevention Techniques

**Ordered Lock Acquisition and Timeout Patterns**:
```csharp
public class DeadlockPrevention
{
    private readonly object _lock1 = new();
    private readonly object _lock2 = new();
    private readonly TimeSpan _lockTimeout = TimeSpan.FromSeconds(5);
    
    // Ordered locking to prevent circular dependencies
    public void OrderedLockAcquisition(object resource1, object resource2)
    {
        // Establish a global locking order based on hash codes
        var locks = new[] { resource1, resource2 }
            .OrderBy(x => x.GetHashCode())
            .ToArray();
        
        lock (locks[0])
        lock (locks[1])
        {
            // Critical section with guaranteed lock order
            ProcessResources(resource1, resource2);
        }
    }
    
    // Monitor.TryEnter with timeout for deadlock detection
    public bool TryAcquireLocksWithTimeout()
    {
        bool lock1Acquired = false;
        bool lock2Acquired = false;
        
        try
        {
            // Attempt to acquire first lock with timeout
            lock1Acquired = Monitor.TryEnter(_lock1, _lockTimeout);
            if (!lock1Acquired) return false;
            
            // Attempt to acquire second lock with timeout
            lock2Acquired = Monitor.TryEnter(_lock2, _lockTimeout);
            if (!lock2Acquired) return false;
            
            // Both locks acquired successfully
            PerformCriticalWork();
            return true;
        }
        finally
        {
            // Release locks in reverse order
            if (lock2Acquired) Monitor.Exit(_lock2);
            if (lock1Acquired) Monitor.Exit(_lock1);
        }
    }
    
    // Async-compatible locking with SemaphoreSlim
    private readonly SemaphoreSlim _asyncLock = new(1, 1);
    
    public async Task AsyncLockPattern()
    {
        // Async-compatible lock acquisition with timeout
        if (await _asyncLock.WaitAsync(_lockTimeout))
        {
            try
            {
                await PerformAsyncCriticalWork();
            }
            finally
            {
                _asyncLock.Release();
            }
        }
        else
        {
            throw new TimeoutException("Failed to acquire lock within timeout");
        }
    }
    
    // Deadlock detection using lock level tracking
    [ThreadStatic]
    private static int _lockLevel;
    
    public void LockLevelTracking()
    {
        if (_lockLevel > 2)
        {
            throw new InvalidOperationException(
                "Potential deadlock detected: lock level too deep");
        }
        
        _lockLevel++;
        try
        {
            lock (_lock1)
            {
                // Nested locking with level tracking
                _lockLevel++;
                try
                {
                    lock (_lock2)
                    {
                        PerformNestedWork();
                    }
                }
                finally
                {
                    _lockLevel--;
                }
            }
        }
        finally
        {
            _lockLevel--;
        }
    }
}
```

## Why are Concurrency Hazards Important?

### 1. **Data Integrity** 
Prevents race conditions that can corrupt shared state, ensuring consistent application behavior through proper synchronization mechanisms.

### 2. **Application Reliability** 
Eliminates deadlocks and livelocks that cause application hangs, improving overall system stability and user experience.

### 3. **Performance Scalability** 
Enables efficient resource utilization through fine-grained locking and concurrent data structures, allowing applications to scale across multiple processors.

## Advanced Nuances

### 1. **Memory Barrier Semantics**
Understanding the difference between `Volatile.Read()`, `Thread.MemoryBarrier()`, and `Interlocked` operations for proper visibility guarantees across CPU cores.

### 2. **Lock-Free Algorithms**
Advanced patterns like Michael-Scott queues or hazard pointers that eliminate locking entirely through careful atomic operation sequencing.

### 3. **ExecutionContext Flow**
How `ExecutionContext.SuppressFlow()` and `AsyncLocal<T>` interact with thread pool operations, affecting ambient data propagation in async methods.

## How This Fits the Roadmap

Within the "Threading Smells" section, Concurrency Hazards serve as the foundation for recognizing and preventing common multi-threading antipatterns. Mastery of these concepts is prerequisite for understanding more advanced topics like lock-free programming, parallel algorithms, and distributed concurrency patterns. This knowledge directly enables developers to write robust, scalable concurrent applications that properly handle the complexities of modern multi-core systems.