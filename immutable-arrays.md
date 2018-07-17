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
        for(int i = 0, pointer = 0; i < NumRows; i++, pointer++)
        {
            for(int j = 0; j < NumColumns; j++)
                result[pointer] = addend[i,j] + _items[pointer];
        }
        return new Matrix(addend);
    }
}

```

This significantly reduces the performance of our add function. We now have to iterate 3 counters. We have to call a virtual function IMatrix[row, column] to access the addends value. That function requires some addition and multiplication to be done before it can do a straightforward array access. As a virtual function, it cannot be inlined.

It would be far simpler if we could have access to the underlying array of addend, and use that.

However if we provide a public accessor to the items, we run the risk of someone else mutating them.
