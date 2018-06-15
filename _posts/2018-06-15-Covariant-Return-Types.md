---
title: Covariant Return Types in C#
---
## Covariant Return Types in CSharp
### What are Covariant Return Types?

Consider the following problem:

You have a class called Animal

``` csharp
public class Animal
{
    public Animal GiveBirth() => return new Animal();
}
```

Cat inherits from animal

``` csharp
public class Cat : Animal
{
    public Cat GiveBirth() => return new Cat();
}
```

Great! That works fine. 
Now what happens when we do this:
``` csharp
var undercoverCat = (animal)new Cat();
var babyCat = (cat)undercoverCat.GiveBirth();
```

Oops. We get an InvalidCastException.

If we want GiveBirth to return a new animal with the same type as the underlying type of the original animal when we call GiveBirth, we have to make the method virtual. We also ought to make Animal abstract, as it doesn't make sense to have an  animal which isnt a specific species.

``` csharp
public abstract class Animal
{
    public abstract Animal GiveBirth();
}

public class Cat : Animal
{
    public override Cat GiveBirth() => return new Cat();
}
```
That's great! Now everything works as expected:

``` csharp
Cat cat = new Cat();
var undercoverCat = (animal)Cat;
Cat babyCat = (cat)undercoverCat.GiveBirth();
babyCat = Cat.GiveBirth();
```

And there you have it! Covariant return types, or the ability to return a more derived type when overriding a virtual function.


### The Problem

There's just one problem:

The code, although making perfect logical sense, doesn't compile. C# doesn't yet support Covariant return types. They are supported by Java and some brands of C++, but C# doesn't yet have them. There is a feature request asking to implement them, but for now I will work through some stopgap solutions.

### Solution 1: Explicit Implementation of Interfaces

Whenever possible interfaces should be preferred to classes anyway. Over here though, changing Animal to an interface gives us something functionally identical to Covariant return types.
``` csharp
public interface IAnimal
{
    public IAnimal GiveBirth();
}

public class Cat : IAnimal
{
    public Cat GiveBirth() => return new Cat();
    
    IAnimal IAnimal.GiveBirth() => GiveBirth();
}
```

This behaves as we want it to.
