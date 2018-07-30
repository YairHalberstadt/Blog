---
Title: Implementing Malloc. A Thought Experiment
---

#### Implementing Malloc. A Thought Experiment

I recently saw a post online about an interview question somebody had. He was simply asked to implement Malloc.

For those that don't know, Malloc is the method used by C (and C++ under the hood) to assign memory on the heap.

Now I have to make clear, **I have no idea how Malloc is implemented in practice**. However the question did get me thinking to myself, how _would_ I implement Malloc?

#### Defining the Question

There are two relevant functions we will have to implement, on a class called memory manager.

``` csharp

public static class MemoryManager
{
    public static byte[] Memory { get; } = new byte[1000000000];

    public static int Malloc(int size) => throw new NotImplementedException();

    public static void Free(int location, int size) => throw new NotImplementedException();
}

```

At start-up we are allocated a large amount of RAM to be used by our program. In this case I've given ourselves a gigabyte.

This memory is accessed through our MemoryManager. We use a pointer to get the value at a particular point in the memory via the Memory array. Since at runtime C is type agnostic, we will consider every location in the memory to be a byte. It is up to the compiler to preserve type safety.

The function Malloc asks for a specific amount of memory to be allocated for the callers use, and returns a pointer to that memory. Malloc cannot allocate the same memory twice.

In order to prevent us running out of memory, Free must used. This frees a chunk of memory of a given size, allowing Malloc to reallocate the memory.

This I think is a suitable model for the problem in C# terms. There are of course added complications that arise in practice, but they don't particularly change this fundamental model.

The problem is to provide a safe, performant implementation of Malloc and Free. They must not allocate memory twice, unless it's been freed first, and and must not 'leak' memory, such that memory cannot be reallocated even after it's been freed.

#### Attempt 1.

In this attempt, we will not worry about managing the memory in the data structures Malloc and Free use internally. We will assume that they are all stored on some external memory and are garbage collected. We will of course remove this assumption later.

To begin with Allocation is easy. We can store a pointer to the first free memory address, and when somebody asks for n units of memory, we can return the value of this pointer to them, and then increment our pointer by n.

This works fine until a full Gigabyte of memory is allocated. Then we need to start allocating memory that has already been allocated, but has since been freed.

To do this we have to store locations that have been allocated but not freed. The obvious way to do this, is to store a list of where the contigous blocks of memory are, and update these blocks every time malloc or free are called.

So the list looks something like this.

```
Start: 0, End: 1009
Start: 1457, End: 3876,
Start: 3877, End 3879,
Start: 154089, End 987523,
...
```

Then we store a pointer to where we are up to in this list. When somebody asks for 1000 bytes of memory, we check if there's any memory after the current block in the list and before the next block. If yes we increase the end value of the current item in the list by 1000. If not we move the pointer to the next item in the list,and repeat. If we reach the end of the list, and there is not enough memory between the final block in the list and the end of our memory array, we loop back to the start of the list.

If we get back to our original start point, and we haven't found any contiguous length of free memory large enough to store 1000 bytes, we throw an OutOfMemoryException.

Freeing memory involves running through the list till the correct block is found, and readjusting the list accordingly.

So if 100 bytes are freed at location 2000, our list turns into

```
Start: 0, End: 1009
Start: 1457, End: 2000
Start: 2100, End: 3876,
Start: 3877, End 3879,
Start: 154089, End 987523,
...
```

And if two bytes are then freed at location 3877 we get.

```
Start: 0, End: 1009
Start: 1457, End: 2000
Start: 2100, End: 3876,
Start: 154089, End 987523,
...
```

As you can see, we keep on adding or removing items in the middle of the list. As such the correct data structure for this is a linked list, as an array based list would require copying half the contents of the list on average, if an item is added at a random location.

So our implementation looks like this in C#

``` csharp

public static class MemoryManager
{
    public static byte[] Memory { get; } = new byte[1000000000];

    public static int Malloc(int size)
        {
            var startingMemLoc = CurrentMemLoc;
            do
            {
                var nextLoc = CurrentMemLoc.Next;
                var space = nextLoc?.Value.Start ?? Memory.Length - CurrentMemLoc.Value.End;
                if (size < space)
                {
                    var newItemStart = CurrentMemLoc.Value.End;
                    CurrentMemLoc.Value.End += size;
                    return newItemStart;
                }
                if (size == space)
                {
                    var newItemStart = CurrentMemLoc.Value.End;
                    CurrentMemLoc.Value.End = CurrentMemLoc?.Value.End ?? Memory.Length;
                    if (nextLoc != null)
                        AllocMemList.Remove(nextLoc);
                    return newItemStart;
                }
                CurrentMemLoc = CurrentMemLoc.Next ?? AllocMemList.First;
            }
            while (startingMemLoc != CurrentMemLoc);
            throw new OutOfMemoryException();
        }

    public static void Free(int location, int size)
        {
            var containingLoc = AllocMemList.First;
            while(containingLoc.Value.End < location)
            {
                if (containingLoc.Next == null)
                    throw new Exception("Freeing Unallocated Memory");
                containingLoc = containingLoc.Next;
            }
            var end = location + size;
            if(containingLoc.Value.Start > location || ContainingLoc.Value.End < end)
                throw new Exception("Freeing Unallocated Memory");
            if(containingLoc.Value.Start == location && location != 0)
            {
                if(containingLoc.Value.End == end)
                    AllocMemList.Remove(containingLoc);
                else
                    containingLoc.Value.Start = location;
                return;
            }
            if(containingLoc.Value.End == end)
            {
                containingLoc.Value.End = location;
                return
            }
            AllocMemList.AddAfter(containingLoc, new AllocatedMemory(end, containingLoc.Value.End));
            containingLoc.Value.End = location;
        }

    private static LinkedList<AllocatedMemory> AllocMemList = new LinkedList<AllocatedMemory>();

    private static LinkedListNode<AllocatedMemory> CurrentMemLoc { get; set; }

    static MemoryManager()
    {
        CurrentMemLoc = new LinkedListNode<AllocatedMemory>(new AllocatedMemory(0, 0));
        AllocMemList.AddFirst(CurrentMemLoc);
    }

    private class AllocatedMemory
    {
        public int Start { get; set; }
        public int End { get; set; }
        public int Length => End - Start;

        public AllocatedMemory(int start, int end)
        {
            Start = start;
            End = end;
        }
    }
}

```
#### Attempt 2. Moving to C

Our implementation above is a perfectly valid one. It suffers from a number of performance problems though, as it is written using standard C# patterns and data structures. It also requires a garbage collector, which of course renders pointless the entire exercise of implementing Malloc.

However the core algorithm is not going to change, and the big O space and time complexity of this implementation is the same as it would be in C.

Now I'm not going to write out the implementation in C. Instead I will explain how it will be different.

Firstly, we won't have any explicit structure called a linked list. Instead we will use structs containing the following 3 pieces of information: the start location of a contiguous block of allocated memory, the end location, and a pointer to the next such struct.

We then keep a pointer to the first such struct.

Other than that, the implementation is pretty much the same.

Now when we create or delete these structs, we will have to call Malloc/Free. This will occur inside Malloc/Free, so we have to check whether infinite recursion could occur.

We would only create or delete these structs at the end of Malloc/Free, once everything else has been done, to prevent our data structures being left in an inconsistent state.

Now Malloc usually just appends to the end of a contiguous data block, so rarely creates or deletes a struct. The exception is when it exactly uses up the memory between two blocks. Then it combines the two structs, and will call free.

Free rarely removes a block, usually adds a block, and sometimes does neither.

Thus it will usually call Malloc, and rarely call free. Since Malloc rarely calls free, it is very unlikely for this to recurse long enough to cause a stack overflow. Thus our implementation is valid in C as well.

#### Improving performance

Assuming a large enough chunk of memory is readily available, Malloc should be an O(1) operation in the number of memory blocks allocated, since it just looks for the next large enough chunk.

Free however is O(n) since it has to traverse through the linked list.

The following heuristic could be used to improve the speed of Free, at the expense of space complexity.

Memory tends to be freed in inverse order to when it was allocated, and close to where it was previously freed.

Thus if we make the linked list a doubly-linked list, we could traverse backwards from the current location, to look for the memory location to free.

We could also store a pointer to the last freed location. Then we could see which of the pointers we have to various locations on the linked list is closest to the location we have to free, and start from there.

This will increase the space usage of our linked list by 33%. It will also increase the cost of adding or removing a node in the linked list slightly.

An alternative is to have a singly linked list as before. Then store pointers to the first block after every n bytes.

So for example, we could have a pointer array of size 20. The first pointer in the array points to the begining of the LinkedList. The next pointer points to the block containing the first allocated memory after 50 mb. The next to the first after 100 mb, the third to the first after 150 mb, etc.

Then freeing memory at location 159,869,423, one jumps straight to the node pointed at by the third member of the array, and starts iterating through the linked list from there.

Whilst Free remains an O(n) operation in the number of contiguous blocks of memory, this is extremely effective. We used only 20 bytes, but decreased the average cost of a free operation by 95%.

Getting the next 95% reduction would use up 400 bytes, and the 95% after that would cost 8 kb. Then 160 kb, 3.2 mb, 64 mb, 1.28 gb, etc.

At some point the amount of required memory required to increase performance further isn't worth it. This is especially the case, as at some point the main cost of Free() will be the actual free operation, not the cost of iterating through the list to find the relevant block.
