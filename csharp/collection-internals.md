# Collection Internals in C#

## What is Collection Internals?

**Collection Internals** refers to the underlying implementation details of C# collection types—how they store data, manage memory, handle growth, and optimize performance. Also known as "collection implementation details" or "data structure internals," understanding this concept reveals the mechanics behind collection behavior. The core purpose is to enable developers to make informed choices about which collection to use based on performance characteristics, memory usage, and thread safety requirements rather than treating collections as black boxes.

## How it works in C#

### List<T>
**Explanation:** `List<T>` is a dynamic array implementation that automatically resizes when capacity is exceeded. Internally, it maintains a backing array and tracks both the number of elements (`Count`) and total capacity. When adding elements exceeds capacity, it creates a new larger array (typically doubling in size) and copies existing elements—an O(n) operation amortized to O(1) per addition.

```csharp
// List<T> internal behavior demonstration
var numbers = new List<int>(capacity: 2); // Initial capacity of 2
Console.WriteLine($"Initial Capacity: {numbers.Capacity}"); // 2

numbers.Add(1); // Fits in backing array: [1]
numbers.Add(2); // Fits: [1, 2]
numbers.Add(3); // Exceeds capacity! New array: [1, 2, 3], Capacity doubles to 4

// Internally: backing array grows exponentially
for (int i = 4; i <= 8; i++)
{
    numbers.Add(i);
    Console.WriteLine($"Count: {numbers.Count}, Capacity: {numbers.Capacity}");
}
// Output shows capacity doubling: 4 → 8 when Count reaches 5
```

### Dictionary<TKey, TValue>
**Explanation:** `Dictionary<TKey, TValue>` uses a hash table with chaining collision resolution. Keys are hashed to calculate bucket indices. Collisions (multiple keys hashing to same bucket) are handled via linked lists. The dictionary maintains an array of entries containing key-value pairs and next pointers for chaining. Load factor triggers resizing when approximately 70% full.

```csharp
// Dictionary internal behavior example
var dict = new Dictionary<string, int>();
dict.Add("apple", 1);
dict.Add("banana", 2);

// Internal structure:
// buckets[]: array of indices into entries[]
// entries[]: array of structs containing key, value, next pointer

// Hash collision demonstration
public class CustomKey
{
    public int Id { get; set; }
    
    // Poor hash function causing collisions
    public override int GetHashCode() => 1; // All keys hash to same bucket
}

var collisionDict = new Dictionary<CustomKey, string>();
collisionDict.Add(new CustomKey { Id = 1 }, "First");
collisionDict.Add(new CustomKey { Id = 2 }, "Second"); 
// Both go to same bucket, creating chain
```

### LinkedList<T>
**Explanation:** `LinkedList<T>` is a doubly-linked list where each node contains a value and references to previous/next nodes. This provides O(1) insertion/deletion at known positions but O(n) random access. Memory is non-contiguous, making it cache-unfriendly but excellent for frequent insertions/deletions in the middle.

```csharp
// LinkedList internal structure demonstration
var linkedList = new LinkedList<string>();
var node1 = linkedList.AddLast("First");  // Creates node with null prev/next
var node2 = linkedList.AddLast("Second"); // node1.Next = node2, node2.Prev = node1
var node3 = linkedList.AddFirst("New First"); // Insert at head: O(1) operation

// Demonstrate node relationships
Console.WriteLine(node1.Previous.Value); // "New First"
Console.WriteLine(node2.Previous.Value); // "First"

// Internal node structure (conceptual):
// class LinkedListNode<T> {
//     public T Value;
//     public LinkedListNode<T> Next;
//     public LinkedListNode