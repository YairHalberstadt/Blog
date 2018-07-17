---
Title: Mitigating the Cost of Abstractions. Using Immutable Arrays
---

### Mitigating the Cost of Abstractions. Using Immutable Arrays

I am currently working on a linear algebra library.

One of the core types in linear algebra is a matrix. A matrix is effectively a two dimensional array of values. One can add two equal sized matrices together by returning a matrix where each entry is the sum of the corresponding entries in the two matrices being added.

Here is a simple implementation:

``` csharp

public class Matrix
{
    private int[][] _items;
    
    // Constructor and properties etc. go here

    public Matrix Add(Matrix addend)
    {
        var result = new int[NumRows][];
        for(int i = 0; i < NumRows; i++)
        {
            result[i] = new int[];
            for(int j = 0; j < NumColumns; j++)
                result[i][j] = addend[i,j] + _items[i][j];
        }
        return new Matrix(addend);
    }
}

```

There's plenty of ways to implement this. Multidimensional arrays are not advised as they are not tuned for performance. 

The two obvious remaining methods are jagged arrays and a normal one dimensional array, which iterates through all the values in each row in order.

There are advantages to both approaches, but in the end I chose to use the single one dimensional array, as I felt it would be easier to tune for performance.

Thus to access the 3rd item on the 5th row is done as `_items[5*NumColumns + 3]`.

So our accessor for an item will require a multiplication, an addition, and an array access.

``` csharp

public class Matrix : IMatrix
{
    private int[] _items;
    
    // Constructor and properties etc. go here

    public int this[row, column] => _items[row * NumColumns + column];

    public Matrix Add(IMatrix addend)
    {
        var result = new int[NumRows*NumColumns];
        for(int i = 0, pointer = 0; i < NumRows; i++)
        {
            for(int j = 0; j < NumColumns; j++, pointer++)
                result[pointer] = addend[i,j] + _items[pointer];
        }
        return new Matrix(addend);
    }
}

```

This significantly reduces the performance of our add function. We now have to iterate 3 counters. We have to call a virtual function IMatrix[row, column] to access the addends value. That function requires some addition and multiplication to be done before it can do a straightforward array access. As a virtual function, it cannot be inlined.

It would be far simpler if we could have access to the underlying array of addend, and use that.

However if we provide a public accessor to the items, we run the risk of someone else mutating them.

The solution is very simple. More recent versions of .Net provide an ImmutableArray<T>. This is a struct which provides a read-only wrapper around an array. Since it is a struct, and the wrapper methods are inlined by the Jitter, it's performance is about identical to that of an array. Theoretically it would even be possible to make it faster than an array under certain circumstances, as the jitter can make optimising assumptions due to it's immutable nature. I don't know if this happens so far, but it gives room for that to happen in the future.

By using this instead of an array for items, our code can remain identical, but we can provide a public accessor to the underlying items, which can be used to optimise our Add method:

``` csharp

public class Matrix : IMatrix
{
    public ImmutableArray<int>  Items {get;}
    
    // Constructor and properties etc. go here

    public int this[row, column] => Items[row * NumColumns + column];

    public Matrix Add(IMatrix addend)
    {
        var result = new int[NumRows*NumColumns];
        var addendItems = addend.Items;
        for(int i = 0; i < Items.Count; i++)
        {
                result[pointer] = addend.Items[i] + Items[pointer];
        }
        return new Matrix(ImmutableArray.Create(addend));
    }
}

```

This also provides another advantage. Previously we would have used a constructor accepting an array of items to create our matrix. As the client may change the array after they've passed it to us, the constructor forced to copy the array in order to guarantee immutability. This reduces the performance of our add function, as we copy the array even though we know that it doesn't change.

Now we can pass an ImmutableArray to our matrix, and the relevant constructor knows that there is no need to copy.

There's just one problem...

ImmutableArray<T>.Create(T[] array) also copies the array in order to prevent the client changing the array later.

The authors of the core library realised that this would be a problem, so provided a builder which allows you to dynamically build your ImmutableArray, and then freeze it, without having to copy the array.

Unfortunately it is not at all performant - in tests I ran it was actually quicker to create an array normally and then call ImmutableArray.Create, even with the extra overhead of copying it.

The solution requires a bit of hackery.

In order to guarantee array-like performance, ImmutableArray is a struct containing exactly one field - the array.

As such it is exactly the same in memory as the array it stores.

This means that we can call unsafe code to simply tell the runtime that our array is actually an ImmutableArray. Since they are exactly the same, this works fine.

Although we are relying on an implementation detail, it is one that is very unlikely to change, as doing so would remove the performance guarantees ImmutableArrays provide.

Fortunately we don't even have to call any unsafe code directly. There is a package System.Runtime.CompilerServices.UnSafe which provides type-safe generic methods to this for you.

We just create a utility method:

``` csharp

public static ImmutableArray<T> UnsafeMakeImmutable<T>(this T[] array)
{
		 return Unsafe.As<T[], ImmutableArray<T>>(ref array);
}

```

And call it on our array to magically convert it to an ImmutableArray.

#### Part 2: fixing the architecture.
