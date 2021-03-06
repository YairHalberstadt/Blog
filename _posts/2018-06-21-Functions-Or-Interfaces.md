---
title: Functions Or Interfaces
---
## Functions Or Interfaces

Consider the following problem:

You want to create a sum function, which adds up all the elements in a list.

No problem - that's an easy one:

``` csharp

public static int Add(List<int> vals)
{
    int sum = 0;
    foreach(var val in vals)
        sum += val;
    return sum;    
}

```

Now that's great - but who told you the list would be a list of ints. What if it's a list of doubles? Or Complex numbers?

You could go and define the add function for each of those types, but then you'll still need to rewrite the function if someone created their own type.

### Method 1: Interfaces

What we really want is something generic like this:

``` csharp

public static T Add<T>(List<T> vals) where  T : IAddable
{
    T sum = 0;
    foreach(var val in vals)
        sum = sum.Add(val);
    return sum;    
}

```

Unfortunately there is no IAddable, and even if you create it, it still won't work for Standard Library types, or any types not created with this function specifically in mind.

If only C# had duck typing!

But it has, I hear you cry:

``` csharp

public static T Add<T>(List<T> vals) 
{
    T sum = (dynamic)0;
    foreach(var val in vals)
        sum = ((dynamic) sum).Add(val);
    return sum;    
}

```

Very funny.

Let me rephrase that; If only C# had type-safe, static, fast duck typing, like Go for example.

Alas it does not. So in old C# we would have had to define an interface IAdder

``` csharp

public static TAddable Add<TAddable, TAdder>(List<TAddable> vals) where  TAdder : IAdder<TAddable>, new()
{
    var adder = new TAdder();
    T sum = adder.Zero;
    foreach(var val in vals)
         sum = adder.Add(sum, val);
    return sum;    
}

public interface Tadder<T>
{
    T Zero {get;}
    T Sum(T first, T second);
}

```

So we have too pass in an instance of IAdder, which tells us how to go about adding TAddable.

This is horrible for a number of reasons. Firstly, it's overengineered, obtuse, and confusing.

Secondly it requires instantiating an instance of IAdder just to access methods which require no state. That's code smell. Really the Zero property and the Sum method should be static, but that is impossible ( even if we were to use an abstract class instead of an interface for IAdder) since static methods cannot be overridden.

The third point is a design one. By doing it this way it becomes impossible to change the definition of Sum at runtime, as the interface implementation has to be created at compile time. Sometimes this is what you want, and when that is the case this method is really you're only choice. Often though this is far too limiting.

You can of course change the signature to

``` csharp

public static TAddable Add<TAddable>(List<TAddable> vals, IAdder<TAddable> adder)
{
    T sum = adder.Zero;
    foreach(var val in vals)
         sum = adder.Add(sum, val);
    return sum;    
}

```

This hides some of the design problems, and gives you more flexibility at runtime, but the underlying problems are still there. 

You still have to instantiate an instance of an interface just to get at it's methods. And you still have to define the interfaces at compile time, even if you can wait till runtime to decide which version to use.

We can avoid the second problem by creating this special instance of IAdder

``` csharp

public class RuntimeAdder<T> : IAdder<T>
{
    public T Zero {get;}
    public T Sum(T first, T second) => _addFunc(first, second);
    private Func<T,T,T> _addFunc;
    public RuntimeAdder(T zero, Func<T,T,T> addFunc)
    {
        Zero = zero;
        _addFunc = addFunc;
    }
}

```

Now this gives us full runtime control of the functions. It also voids the first problem, since you need an instantiated interface in order to hold state - namely the desired functions. However it adds another layer of indirection, as you now need to call an instance method to access a private Function of the interface, rather than calling the interface directly.

### Method 2: Functions

What was the core thing that was wrong with our attempt to implement this behaviour with interfaces?

It was that we were passing in an implementation of an interface, just in order to get to it's methods. All we were really interested in was the Add Function, and the property Zero. So instead of beating around the bush, passing in interfaces, let's just go straight to functions:

``` csharp

public static T Sum<T>(List<T> vals, T zero, Func<T,T,T> addFunc)
{
    var Sum = zero;
    foreach(var val in vals)
         sum = addFunc(sum, val);
    return sum;
}

```

Far cleaner, and about as clean as we're going to get.

There's just one further improvement we can make. Note that we have no guarantee that the add function does any adding. This method will work whether or not addFunc is remotely connected to adding, and no matter what value we pass in as zero.

All that matters is that zero is a T, and addFunc is a function that takes two Ts and returns a T. So in truth we've actually created an aggregate function which combines a list of Ts into one single T by repeatedly applying a function.

Let's rename Sum to reflect that:

``` csharp

public static T Aggregate<T>(List<T> vals, T seed, Func<T,T,T> aggregator)
{
    var current = seed;
    foreach(var val in vals)
         current = aggregator(current, val);
    return current;
}

```

In fact, if T is a group under aggregator, and seed is the identity element, Aggregate becomes a Monad, one of the core patterns in functional programming, and can be easily scaled using MapReduce. It's funny how just thinking about implementing a sum function in C#, bought us all the way deep into the heart of functional programming.

Now this looks very similar to Linq.Aggregate. The two differences are that this is not an extension method, and that it takes a List not an IEnumerable.

The first is purely cosmetic, but the second is actually quite interesting. IEnumerable is the way C# allows you to write as generic a function on enumeratable things as possible, and it uses an interface to do so.

Of course this has the standard problems with interfaces - it only works if the class happens to implement the interface, and it only gives one possible way to use the interface per class.

How then would you give your own IEnumerable definition to a class if it doesn't implement IEnumerable, or doesn't implement it in the way you want?

The easiest solution is to build a wrapper around the class and implement IEnumerable from there:

``` csharp

public class EnumerableWrapper : IEnumerable<SomeType>
{
     private NonEnumerableClass _instance;
     public Wrapper(NonEnumerableClass instance) => _instance = instance;
     public IEnumerator GetEnumerator() ...
     ...
}

```

Now let's go down the same path as we took previously, and try to make this wrapper as generic as possible, whilst allowing the method of enumeration to be specified at runtime.

This time I'll skip all the intermediary steps, and go straight to the final outcome

``` csharp

public class EnumerableWrapper<T> : IEnumerable<T>
{
     private Func<NonEnumerableClass, IEnumerator<T>> _getEnumerator;
     private NonEnumerableClass _instance;
     public Wrapper(NonEnumerableClass instance, Func<NonEnumerableClass, IEnumerator<T>> getEnumerator) 
     {
          _instance = instance;
          _getEnumerator = getEnumerator;
     }
     public IEnumerator GetEnumerator() => _getEnumerator(_instance);
     ...
}

```

So if we wanted to do this functionally we would do something like this

``` csharp

public static T Aggregate<S, T>(S vals, Func<S, IEnumerator<T>> getEnumerator, T seed, Func<T,T,T> aggregator)
{
    var current = seed;
    var enumerator = getEnumerator(vals);
    while(enumerator.MoveNext())
         current = aggregator(current, enumerator.CurrentValue);
    return current;
}

```

Is this now fully functional?

Not really - we are relying on the interface IEnumerator to do all the heavy lifting. We ought to replace that with a function.

So how does IEnumerator work? It contains all the information necessary to generate the next item in the collection, plus a method which retrieves that next item and updates the information

So we need a function that takes a seed containing the information necessary to get to the next item,  and returns a bool indicating if there are any more items, the next item, and a new seed to get the item after that.

i.e a Func<TSeed, (bool, TItem, TSeed)>

You would then pass in a function into Aggregate that takes the collection and returns this function, as well as the first seed Thus the signature of Aggregate looks like this:

``` csharp

public static TItem Aggregate<TCollection, TItem, TSeed>(TCollection vals, Func<TCollection, (Func<TSeed, (bool, TItem, TSeed)>, TSeed)> getEnumeratingFunc, T seed, Func<T,T,T> aggregator)

```

Ugh. I'll spare you the implementation, but it involves using a tail recursive algorithm to apply the aggregate step. Thus the recursion can be optimised to a loop.

This method might look ugly, but it's actually doing exactly the same thing as an IEnumerator would, once decent optimizations are applied. The reason it's so ugly is because we haven't encapsulated the logic, so all of it takes place in this one method.

So how do we encapsulate things from a functional perspective?

Usually we use closures and partial application to create a function with certain parameters already set. It's not immediately obvious how to this here though.

One might instead use the original function we had which took a list. One could then create a different Function called ToList whose job is to generate a list in the first place.

``` csharp

public static List<TItem> ToList<TCollection, TItem, TSeed>(TCollection vals, Func<TCollection, (Func<TSeed, (bool, TItem, TSeed)>, TSeed)> getEnumeratingFunc)

```

This is slightly better, but still very ugly. Ignoring that for now, if this were Haskell where everything is lazy by default, this will be still be lazy. In C# we might want to make this return a Lazy<List<T>> instead.

However the biggest problem is that we've now lost the generality of our original method, as it now only accepts a List. It is only general in conjunction with another method. So let's make it take a ToList function instead of a list:

``` csharp

public static T Aggregate<S, T>(S vals, Function<S, List<T>> toListFunc, T seed, Func<T,T,T> aggregator)
{
    var list = toListFunc(vals)
    var current = seed;
    foreach(var val in list)
         current = aggregator(current, val);
    return current;
}

```

Now one could partially apply the ToList function we had earlier to create a toListFunc to pass in to Aggregate. This is neater than passing in a Func<TCollection, (Func<TSeed, (bool, TItem, TSeed)>, TSeed)> to Aggregate, as it makes no assumptions about how the collection is converted to a list.

However the truth is that in most functional languages you're unlikely to see anything like our ridiculously over-engineered ToList function. One of the reasons for this is that there is usually only one sensible way to enumerate any given type. As such it is usually easier to define a single function specifically for each type you would like to enumerate, and define how a list is generated there.

Once you've done that, it's only a small logical step to storing the function with the data type as a method, and once I've done that I might as well make it implement an interface.

(As an aside, the reason IEnumerable doesn't just use a single function called ToList() is because using the more complex GetEnumerator allows for lazy evaluation. ToList() would be extremely non-performant if all you wanted was the first item.)

There's one big advantage to using an interface though. Now you can pass in the same collection to any method accepting an IEnumerable without any fanfare. And that includes the whole of linq, plus thousands of other methods on most large codebases.

Similiarly implementing an IEqualityComparer is often a pain in the neck. It's often simpler to define the necessary functions where you need them. However, once the IEqualityComparer is done, it neatly binds together all the code necessary for equality comparison in one useful class, and can be easily used all over the place.

### Conclusion

This has been a rather winding journey, and one which didn't have a specific goal in mind.

I think if we were to some up the ideas we've learnt it would be as follows:

1. Both functions and interfaces are used to solve the same problem: how do you solve the general problem as opposed to the specific problem.

2. Functions offer far greater flexibility, and are often more direct. They can be defined locally and at runtime far more easily, and thus offer greater generality than functions.

Thus accepting functions as a parameter increases the reusability of your method, by allowing it to be manipulated in an infinite number of ways.

3. Interfaces offer encapsulation. They neatly cut off complexity from you, so that you don't have to see it. They also offer greater consistency - if there's only one sensible way of doing something, than an interface can make it likely that that way will be used every time. They can be used easily across your codebase, without any fanfare.

Thus interfaces increase the reusability of your classes, by allowing them to be used with many methods.

I think the questions to ask when deciding whether a method accepts an interface or a function is this:

If I implemented an interface to be passed into this method, how often would I use it again?

How many different interface implementations might I have to create just to use this method?

How many different functions does this method need to accept, and could that complexity be encapsulated by putting it into an interface?

Of course that's an over-simplification of the decision, but I believe that this blog post is far too long as it is.

So for now, Goodbye!
