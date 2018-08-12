---
Title: CLR Internals. How does the GC know when objects are dead?
---
## CLR Internals. How does the GC know when objects are dead?

Unlike C and C++ where memory must be manually allocated and freed using Malloc or similar mechanisms (as discussed in a previous post), the CLR provides a garbage collector (GC) that automatically frees memory when necessary or when too many 'dead' objects are allocated on the heap.

### Brief overview of the GC

The .Net GC is a generational mark-and-sweep garbage collector.

For simplicity we will assume all objects are small, and none are allocated on the large object heap.

All new objects are allowed on an area of memory known as the ephemereal segment. They are automatically given generation 0.

The GC will be called by the memory allocator, usually when the ephemereal segment is full.

Assuming that a generation 0 garbage collection is called:

The GC calculates which Gen0 objects are reachable from the program and marks them as such. All other Gen0 objects can be safely deleted.

The GC calculates where all the live objects would go should a compaction occur. If it decides it is worth it, it will ask for a new ephemereal segment from the OS, and copies the live objects to it, leaving the original segment free for deletion. The surviving objects are marked as generation 1.

If the GC decides not to compact the heap, a free object is made out of all the dead objects.

Generation 1 and generation 2 garbage collections are similar, but run less frequently. There is no generation 3 - objects which survive generation 2 are not promoted up a generation.

### How does the GC know which objects are dead.

What I want to focus on here, is how the GC can tell which objects are live, and are not safe to delete, and which are dead, and can be safely removed.

All live objects can be somehow accessed by the program. This can be through 3 ways:

1. A reference to an object exists on the stack.

2. A reference to the object is held in a static field/property/function.

3. A reference to the object is held by another reachable object.

Thus by traversing the tree of objects reachable from  the stack, or from static objects one can find all live objects, and hence all other objects are dead.

The GC stops traversing a branch of the tree when it reaches an object that has already been marked live. It then can backtrack and continue down another branch.

Eventually all branches end in live objects, meaning that all live objects have been found.

The problem with this method is that it requires going through the whole heap each time, not just generation 0.

The solution is to keep a list of all references to Gen0 objects which are held by Gen2 or Gen1 objects.

Then, during a Gen0 collection one can traverse the tree starting at the stack, static objects, and this list, and can stop traversing a branch once one reaches either a live object, or a Gen1 or Gen2 object.

### Memory leaks in C#

If C# is garbage collected, how can memory leaks occur?

The GC only looks to see if objects are accessible from the stack, or static objects. It makes no attempt to reason if theoretically accessable objects will ever in practice be accessed.

As such certain data structures can leak memory by holding on to data which the type system renders impossible to access.

As an obvious example, consider the following wrapper for a list.

``` csharp

public class FirstItemGetter<T>
{
     private readonly List<T> _list;

     public FirstItemGetter(List<T> list ) => _list = list;

     public T GetFirstItem => _list.FirstOrDefault();
}

```

If the only reference to a list is stored in a FirstItemGetter, the List will not be freed. Neither will any of the objects the items in the list reference, nor the objects they reference, etc.

This is despite it being impossible to access any object other than the first item in the list. Effectively the memory has leaked.

Whilst this is quite obvious, it's not something .Net programmers readily think about. We're too used to relying on the garbage collector to realise that if we still have references to objects, the garbage collector can't free them. And I've come across more subtle versions of the problem, for example in a drop-out stack I implemented.

Sometimes we 'leak' memory even when the type system renders it accessible.

If we persist a giant list of items, throughout the lifetime of our program, but never use them except at the very first stage, those items can't be freed, and memory is wasted.

If we persist a data structure which directly or indirectly references 1000s or even 100s of thousands of objects (and such data structures are surprisingly common), when all we want to do is access one field of that object, we've wasted Mb of memory completely pointlessly. And God help us if we persist a list of such objects.

The GC is a wonderful tool. But we must be aware of how it works, rather than treating it as a peace of magic, if we are to extract the greatest benefit from it.
