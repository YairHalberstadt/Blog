---
Title: Mitigating The Cost Of Abstractions pt5. Virtual Functions.
---

## Mitigating The Cost Of Abstractions pt5. Virtual Functions.

This post is going to discuss both performance and architecture relating to the use of virtual methods.

#### A description of virtual functions

Both virtual function calls, and calls to interface methods, are less straightforward than a normal method call.

With a normal method call, the compiler knows what function is going to be called, and so the Jitter is able to simply jump to the correct function when the code runs.

With virtual functions and interface methods, the compiler can't know which method is going to be called, since it depends on the type passed in.

As such it builds a VTable, or uses virtual stub dispatch, which the runtime checks to see which method it should call, and jumps to that.

There are 3 relevant performance costs here.

1. The VTable check takes time. This of course slows down performance.

2. The Jitter cannot inline a virtual function, since it doesn't know which function will finally be called. This limits it's ability to optimise function calls, especially when a fast function is called in a tight loop.

3. Every class has a vtable, and this vtable grows longer the more virtual functions the class inherits or declares. This increases the total amount of stuff the runtime has to hold in memory to run the code. This uses RAM, and also makes a cache miss more likely.

For plenty of code bases this will be irrelevant. If one uses virtual functions judiciously, and only on methods that are anyways long and slow, there is unlikely to be a noticeable performance hit.

On the other hand, just one virtual function call in the wrong place can wreck performance under certain circumstances. See my [post](https://yairhalberstadt.github.io/YairHalberstadtBlog/2018/07/09/mitigating-the-costs-of-abstractions-pt2.html) about using functions to do arithmetic for an example of an interface method which caused a serious performance hit, and the trick I used to avoid it.

Note that if you call a method on an implementation of interface directly, or you call a sealed override of a virtual method directly, there is no performance hit. It's only when the compiler cannot be sure what method will be called - ie when you call the method on the interface, not the implementation, or call a non-sealed virtual function - that you get a performance hit.

So the first tips I can give you are:

**Don't use virtual functions unless there is a specific reason too!** 

**Always seal your virtual function overrides, unless there is a specific reason not to!**

#### Virtual Method Internals

The implementation of virtual methods is actually relatively straightforward. Since multiple inheritance is disallowed in C#, every class has a single inheritance chain descending from obj.

We order all the virtual methods of a class in order of where they are declared up the inheritance chain. Thus the first three methods for any class will be some permutation of object.GetHashCode, object.Equals and object.ToString. Then after that will be the virtual methods declared on the class inheriting directly from object, then the virtual methods of the class inheriting from that, etc, all the way down to our class at the bottom of the chain. Lets call this the method ordering. 

We then build a virtual table for each class, which is basically an array of function pointers to each virtual method, ordered using the method ordering.

Every object has a pointer to it's classes virtual table in its object header.

A virtual function call gets the location of the virtual table from the object header, adds the index of the method in the method ordering to the location, and gets the function pointer from there.

So given that eg. MyClass.Method() is the 5th method in the method ordering this is how a callsite would be viewed by the Jitter.

``` csharp

    public static void CallMethod(MyClass obj) => obj.Method();

    // This turns into something like

    public static void CallMethod(MyClass obj) => obj.Header.VirtualTable[5]();

    // Or its equivalent in machine code at any rate.

```
With interfaces this wouldn't work, since a class can implement multiple interfaces. As a result it's impossible for all classes to have the same index for the methods in the interface they implement.

For example, let's say Interface1 usually occupies slots 6 - 10 in a vtable, and Interface2 usually occupies slots 8-15.

If a class implements both interfaces, how would we know where in the VTable to find the relevant method?

Instead virtual stub dispatch is used for interfaces. Whilst the internals are complicated, the basic mechanism is simple - the virtual stub dispatch is basically just a dictionary of types to methods. The type is extracted from the object, and the method is looked up in the dictionary, and called.

Caching is used to increase performance with the most commonly called implementations of the method.

As a result interfaces calls are slower than virtual function calls. The exact performance cost depends on whether the interface call is usually being made on the same type, which will allow effective caching of the method during virtual stub dispatch.

In tests I performed, interface calls were about double as slow as virtual calls in the best case scenario, where it is only ever called on one type.

Either way, both virtual functions and interfaces calls should be avoided in tight loops in performance critical code.

If they are necessary, one can use structs to implement an interface, and call the interface using a generic method with a type constraint, as discussed in a [previous post](https://yairhalberstadt.github.io/YairHalberstadtBlog/2018/07/09/mitigating-the-costs-of-abstractions-pt2.html). That offers no performance costs other than that of initialising a new instance of the generic type.

If that still isn't acceptable, then virtual functions are better than interface calls from a performance perspective, but still slow, and rife with other issues as we shall soon see.

#### Architectural Advantages Of Interfaces

As I said above, virtual functions and interfaces have similiar costs, although interfacea will be slower, depending on use case. However interfaces tend to have a positive effect on architecture, whereas virtual functions raise a number of problems.

Firstly let's look at interfaces.

When you program to an interface, it tends to improve the quality of your code, as it more strictly divides the inside and the outside of the class. As such it encourages a loosely coupled architecture. Furthermore it makes testing easier, and makes it simple to plug in a different class or remove an old one.

Especially if one uses standard naming conventions, an interface is easy to spot, and is explicit about not providing an implementation. As such when one is using an interface one is fully aware that one is dealing with a contract, not an implementation.

A class can inherit multiple interface, and can implement them both explicitly and implicitly. Thus accepting an interface in your API as opposed to a class gives far more flexibility to the client code as to how to make use of the API.

An interface implementation is also non-virtual by default. As such you only pay the cost of using a virtual function if you access the implementation via the interface rather than directly calling the method on the implementing class, which means interfaces are pay as you go. If you don't use them, they don't cost you. They will still increase the amount of memory needed in RAM, but if they're never used, they won't clog up the cache.

As such I believe that even on the most performance critical code it's worth programming to an interface  where one is not facing a performance bottleneck. The wonderful thing about interfaces is they're a doddle to remove if they are only implemented once, and so once the code is done, one can remove any unneeded interfaces on internal code, and get an instant performance boost.

#### Architectural Issues With Virtual Functions

Now virtual functions do have some advantages in specific cases. However they also come with significant costs.

Let's consider Java for example. In Java all methods are virtual by default, unless explicitly sealed. As a result not only is there a huge performance cost, but there is a huge safety cost.

A class can never be sure when calling one of it's own methods what it's effect will be. Maybe the method will be overridden, and bugs will appear.

Similarly, one can never safely override a method, because one doesn't know how the method is used internally. The fact that it's virtual is no proof it's safe to override it, since all methods are virtual by default.

Fortunately in C# virtual methods are less pervasive, as methods are sealed by default.

However as a technique to modify behaviour they are less elegant than interfaces.

Firstly, they don't divide implementation and definition neatly, but instead cause tight coupling as inheriting classes and base classes both are directly effected by the others implementation.

Secondly, accepting a class, even an abstract one, in an API gives far less flexibility to the client code, as all objects passed in to that API must inherit from that class, and as such cannot inherit from anything else.

Thirdly, virtual function overrides are virtual by default. As such, even if you access the inheriting class directly, you still pay for a virtual function call. They are not pay as you go, unless explicitly sealed, moving away from the 'pit of success' interfaces try to provide.

Fourthly, the implementation of the parent class may depend on the virtual function that is being overridden. It is easy to unwittingly write code that is dependent on the specifics of the base method implementation. Thus one can unknowingly break code by overriding a function. Meanwhile when relying on interface methods it is explicit that the interface has no given implementation, and so developers are aware not to rely on a specific implementation of the interface.

Fifthly, interfaces are more explicit than virtual methods. When implementing an interface it is clear what has to be done: you provide a method for every member of the interface. When inheriting from a parent class it is not clear which virtual functions should be overridden, and what effect they will have on the parent class. Similarly when recieving an interface, it is obvious that the exact implementation will change, so one does not rely on it. However when receiving a class, it is not at all obvious at first sight if the class has virtual functions, and what their effect is.

Given my preferences for interfaces over virtual functions, let's go through common use cases for virtual functions, and whether or not we can replace them with interfaces.

#### Use Case 1: Abstract Classes

Unfortunately interfaces in C# have limitations. As such if one wants to provide operator overloads or implicit casts to interfaces, one has no choice but to use an abstract class instead, and replace interface methods with abstract methods.

With C# 8 we should hopefully see this issue corrected, and we can start converting abstract classes to interfaces.

Another limitation with interfaces is that one cannot provide an implementation in interfaces for behaviour that is resultant of the interface methods.

So for example

``` csharp

public interface MathsDefiner
{
    int Add(int a, int b);
    int Negative(int a);
}

```

Now I want to define that subtraction is the same as adding the negative of the number, so there should be a method ` int Subtract(int a, int b) => Add(a, Negative(b))`.

However I can't put Subtract in the interface because then someone may implement it incorrectly, and because I don't want to pay for an extra virtual function call.

Currently one can use extension methods like so

``` csharp

public interface IMathsDefiner
{
    int Add(int a, int b);
    int Negative(int a);
}

public static class IMathsDefinerExtensions
{
    public static int Subtract(this IMathsDefiner mathsDef, int a, int b) => Add(a, Negative(b));
}

```

However it would be understandable if someone wanted all the methods to be in the same place, or didn't want the extra noise of extension methods, and so chose to use an abstract class instead.

C# 8 should again hopefully mitigate that issue.

Abstract classes are annoying as they do not support multiple inheritance. However they are often necessary, and so long as they are just used as an interface+ they're not terribly problematic. Just remember to seal any abstract methods that your implement where possible.

#### Use Case 2. Allowing slight modifications to an existing class.

Oftentimes one has a class that is very similiar to an existing class, but has some features changed/added. As such one wants to avoid reimplementing the entire class, and so chooses to make one of the parents classes functions virtual, and to override it.

For example, you might have a BinaryTree class, and then have a SubscribableBinaryTree which is the same as a BinaryTree, but sends an event when an item is added.

So you do something like this:

``` csharp

public class BinaryTree
{
 
...

    public virtual void Add(Node parentNode, Node newNode, bool rhs)
    {
    ...
    }
}

public class SubscribableBinaryTree : BinaryTree
{
...

    public override void Add(Node parentNode, Node newNode, bool rhs)
    {
        AddEvent.Invoke();
        base.Add(parentNode, newNode, rhs);
    }
}


```

Now this should in and of itself be a code smell, since you are modifying the parent class for the sake of a different class, even though it has no internal need to be virtual. BinaryTree is a perfectly valid, self contained class, which no-one would ever think to provide with virtual methods untill you came along with your SubscribableBinaryTree.

Besides for all the problems with virtual functions I've outlined above, this pattern forces every method in every class to be virtual just in case someone wants to modify its behaviour later. Ugh.

Besides, all you want to modify is the external interface of the class, but you've ended up modifying the internals as well. What happens if BinaryTree uses add internally in other methods? You'll have events firing when you don't want them to.

Using interfaces there is a more general solution.

Firstly BinaryTree should implement an IBinaryTree interface. It should because every reusable class should, to aid flexibility and testing.

Secondly SubscribableBinaryTree should also implement the IBinaryTree interface, and delegate implementation to BinaryTree. Resharper can do all this for you in two clicks.

Thirdly you edit the Add method to call your custom logic before it calls BinaryTree.Add().

``` csharp

public interface IBinaryTree
{
    void Method1(T param);
    void Method2(T param);
    void Method3(T param);
    void Add(Node parentNode, Node newNode, bool rhs);
}

public class BinaryTree : IBinaryTree
{
 
...

    public void Add(Node parentNode, Node newNode, bool rhs)
    {
    ...
    }
}

public class SubscribableBinaryTree : IBinaryTree
{
    private BinaryTree _tree;

...

    public void Method1(T param) => _tree.Method1;
    public void Method2(T param) => _tree.Method2;
    public void Method3(T param) => _tree.Method3;

    public void Add(Node parentNode, Node newNode, bool rhs)
    {
        AddEvent.Invoke();
        _tree.Add(parentNode, newNode, rhs);
    }
}


```

An added advantage of this technique is that one could make the `_tree` an IBinaryTree and inject the tree into SubscribableBinaryTree, or make the class `SubscribableBinaryTree<T> where T : IBinaryTree` and have `_Tree` an instance of T. This allows you to extend this code to work with all IBinaryTrees, something that one can't do by inheriting from BinaryTree.

As well as all the advantages outlined above, this has both performance advantages and disadvantages.

The advantages are that when dealing directly with a BinaryTree there are no performance costs, as it has no virtual methods. If this is likely to be far more used than SubscribableBinaryTree, this is extremely important.

On the other hand, any method which accepts an IBinaryTree now has to use an interface call on any IBinaryTree method it accesses, whereas with the original code, it would have been able to accept a BinaryTree, call non-virtual methods for all except Add, and still work with a SubscribableBinaryTree.

#### Use Case 3. Modifying the internal behaviour of a class.

The Animal example is one of the most used examples of virtual functions.

You have an abstract class Animal, with an abstract method MakeNoise(). You then inherit from Animal with a Cat and a Dog, each of which provides a different implementation for MakeNoise(). The Cat meows when MakeNoise() is called, and the Dog barks.

The idea here being that the virtual function is not just there to provide you with a method to save your having to copy the class out over again for each animal, but is an intrinsic part of how Animal works. Animal expects you to override MakeNoise in order to allow Animal to have many different behaviours.

The way this would be done with interfaces would be with dependency injection. You would pass an INoiseMaker into Animal's constructor, or make Animal generic on a T where T inherits INoiseMaker. Then MakeNoise would call INoiseMaker.MakeNoise().

Then Cat or Dog could inherit from Animal, and have a Meower or Barker baked in.

At first sight the abstract Animal seems neater and more reflective of the underlying reality of the objects we are modelling. It seems to be a natural way of expressing the relationship between animals.

Alas the real world isn't all kittens and puppies. When you're dealing with StemmerFactories and ChunkPointDetectors or whatnot, it becomes more difficult to justify that one architecture is more reflective of the true nature of these objects. Instead you care about which technique is more flexible and poweful.

Let's consider two real world examples.

The first is implementing a SortedList<T>.

Now you have to define how the SortedList should sort it's objects. You could add an IComparable constraint to T, but then you're limiting yourself to only a few types. Really you want the user to be able to define the sorting mechanism for each T.

You could do this by giving SortedList an abstract Compare method, which you then implement.

Or you could do this by injecting an IComparer<T> into the SortedList constructor.

Either way, if you have a hundred different classes you'll need to inherit from SortedList a hundred times, or implement IComparer 100 times.

The difference is what happens after that. Because soon you decide to create a SortedDictionary and a SortedTree and a SortedHashSet and a SortedBalancedBinaryTree, and decide to create the Linq extension methods .SortBy, .MaxBy and .MedianBy, and all of them require using a custom definition for Compare.

If you went through the virtual method route, you're back to square one, needing to inherit from each Sorted class 100 times. And you'll still be stuck once you get to your custom Linq functions.

However if you used dependency injection, you can inject the same IComparer into each of your classes and methods. Those 100 implementations of IComparer are reusable.

The second example is a library with internal classes. As a random example it's a Linear Regression Library.

Now the Linear Regression Library has a nice neat public API, but somewhere in its belly it has an internal class called `Scorer` which calculates the mean squared distance between a line and some points.

Now for different data you need different definitions of distance.

You might think to inherit from Scorer and override a virtual method which defines distance. However the class is internal so that won't work.

The only option here is to inject an IDistanceCalculator through the public API of the Linear Regression Library, and for Scorer to use that.

I think both of these examples reflect the same concept. Often using dependency injection encourages good separation of concerns, which aids code reuse and flexibility. Whereas using virtual methods can end up with putting everything and the kitchen sink in the same class.

Just consider an abstract class with 3 different abstract methods each of which is overriden 3 different times. That requires 3^3 = 81 different classes to cover all the possible options, whereas using dependency injection would only require 9 interfaces to be defined.

This indicates that the methods being overriden are not actually key to the class, and could be better put outside the class, in an interface.
