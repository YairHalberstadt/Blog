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

    public IMatrix Add(IMatrix addend)
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

    public IMatrix Add(IMatrix addend)
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

Of course clients can do exactly the same thing to pass in an ImmutableArray to our constructor, but once they're explicitly calling something called "Unsafe" they are entirely responsible if things don't work as expected.

#### Part 2: fixing the architecture.

To get this to work we've had to introduce a property called Items which returns an ImmutableArray to the IMatrix interface.

This is a code smell - that an ImmutableArray is used to store the values is an implementation detail, and an interface should not contain implementation details.

Indeed it soon turns out that this is a bad idea. Consider for example a diagonal matrix. This is a square matrix only containing values along it's diagonal - all other values are zero. As a result it only needs to store those values, leading to significant memory saving.

By forcing it to implement the Items property, it now needs to either store an ImmutableArray of the entire   matrix, or generate it each time the property is called, leading to significant memory or performance overhead.

So we must remove the Items Property from IMatrix.

We could have a new interface IAccessableMatrix, or something like that with the Items property, and add a new overload to Matrix.Add which accepts an IAccessableMatrix.

However, in the end I decided that even this isn't necessary. This property will only exist on a matrix if it has an implementation where every value is stored in a single ImmutableArray. At this point it's hard to see what functionality it provides that can't be had by deriving from Matrix. Even if it does, it's unlikely you would be mixing and matching the two types that often - you would presumably either work mainly with one or the other. As such I felt that the interface was overkill.

Instead I decided that each implementation of IMatrix should be optimised for the types of Matrix it  is most likely to be used with. Other than that it should make no assumptions about the underlying implementation of the IMatrix.

As such Matrix.Add has a specific overload accepting another Matrix as a parameter so it can use the Items property. It also does a type check when an IMatrix is passed in to see if it should use the Matrix overload instead.

It could do similar things for a diagonal matrix, or any other type of Matrix.

This provides another advantage. As discussed in my first post, C# does not support covariant return types. As such Matrix.Add(IMatrix addend) must return an IMatrix instead of a Matrix, since IMatrix.Add returns an IMatrix.

However Matrix.Add(Matrix addend) is not an implementation of IMatrix.Add, so can return whatever it wants - in this case a Matrix. This is useful as Matrix has some functionality - like Items - that IMatrix doesn't. It also means that we don't have to use virtual dispatch when calling a method on Matrix, like we would if we were calling it on an IMatrix. This can significantly increase performance, not least since short methods can now be inlined by the Jitter.
