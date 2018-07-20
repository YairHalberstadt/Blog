---
Title: Mitigating the cost of Abstractions pt5. Linq
---

Linq is one of the most powerful features of C#. I find it incredible how I find most of the code I write can be expressed as a series of Linq statements.

I think Linq highlights the fact that the core of computing is just data manipulation. By providing some seriously powerful tools to do that manipulation, it can replace the vast majority of code extremely effectively.

However all that power comes at a cost. Let's look at a simple Linq query, and it's for loop alternative, and consider the costs.

``` csharp

public static List<S> selectFirst<S, T>(List<(S Item1, T Item2)> tuples) => tuples.Select(x => x.Item1).ToList();

public static List<S> selectFirst<S, T>(List<(S Item1, T Item2)> tuples)
{
    var results = new List<S>(tuples.Count)
    for(int i = 0; i < tuples.Count; i++)
    {
        results.Add(tuples[i].Item1);
    }
}

```

Now the Linq version is far shorter, and more importantly, to the point. There's no boiler plate, no need for loops, no risk of off-by-one errors. It just tells you straight out - select Item1 from every member of tuples.

However, in this case since we're calling ToList() after the select statement the two versions of this method are functionally identical. So let's compare what goes on in the two versions:

Version one - Linq:

- A lambda is instantiated.
- an IEnumerable is created, containing the information necessary to call select on the list.
- an iterator is created.
- an empty list is created.
- Iterator.MoveNext and Iterator.Current are called in a loop, as well the lambda expression on each value in the list. Note that a lambda expression call is significantly slower than a normal function call.
- the results are stored in a list, which has to be continuously resized as the results come in.
- the List is returned.

Version two - for loop:

- A list is created with the required capacity, avoiding for having to copy the underlying array repeatedly later.
- an integer is set to 0. Essentially free.
- in a loop an integer is incremented and checked against another integer, a list is accessed by index, a property is accessed, an item is added to a list which is already of the required size. No virtual functions are called, meaning any methods called will probably be inlined. Each of the methods is just a handful of machine code instructions at most.
- the List is returned.