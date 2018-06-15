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
    public Animal GiveBirth() => new Animal();
}
```

Cat inherits from animal

``` csharp
public class Cat : Animal
{
    public Cat GiveBirth() => new Cat();
}
```

Great! That works fine. 
Now what happens when we do this:
``` csharp
var undercoverCat = (Animal)new Cat();
var babyCat = (Cat)undercoverCat.GiveBirth();
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
    public override Cat GiveBirth() => new Cat();
}
```
That's great! Now everything works as expected:

``` csharp
Cat cat = new Cat();
var undercoverCat = (IAnimal)Cat;
Cat babyCat = (Cat)undercoverCat.GiveBirth();
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
    public Cat GiveBirth() => new Cat();
    
    IAnimal IAnimal.GiveBirth() => GiveBirth();
}
```

This behaves as we want it to.

``` csharp
Cat cat = new Cat();
var undercoverCat = (IAnimal)Cat;
Cat babyCat = (Cat)undercoverCat.GiveBirth();
babyCat = Cat.GiveBirth();
```

There are two disadvantages to this method. Firstly it increases code bloat, and secondly it only works when you can turn the base class into an interface., which isn't always possible. 

However when it is possible I believe this is the cleanest solution.

### Solution 2: Create 2 Methods

The following code compiles fine:

``` csharp
public abstract class Animal
{
    public abstract Animal GiveBirth();
}

public class Cat : Animal
{
    public override Animal GiveBirth() => new Cat();
}
```

The problem with it is that calling Cat.GiveBirth, returns an Animal, which I then have to downcast to a cat. This is tedious, reduces the readability of the code, and increases the chances of a runtime error by reducing type safety.

So we can improve the code like this:

``` csharp
public abstract class Animal
{
    public abstract Animal GiveBirth();
}

public class Cat : Animal
{
    public override Animal GiveBirth() => GiveBirthToCat();
    
    public override Cat GiveBirthToCat() => new Cat();
}
```

Then whenever we deal with a cat we call Cat.GiveBirthToCat(), and a Cat is returned. This though increases the number of functions, and is confusing to the consumer of this class - what is the difference between the two methods? It also feels odd to have two methods with different names do the same thing, and feels very much at odds with polymorphism. 

Sometimes when a method takes a parameter, we can improve on this, and give both functions the same name. 

``` csharp
public abstract class Animal
{
    public abstract Animal Mate(Animal mate);
}

public class Cat : Animal
{
    public override Animal Mate(Animal mate) => mate is Cat cat? Mate(cat) : throw new ArgumentException("Cross Species Interbreeding Not Allowed");
    
    public override Cat Mate(Cat cat) => new Cat();
}
```

This works very well in a case where the parameter ought to be of the same type as the class, but is of limited use elsewhere. 

### Solution 3: Use a Private Virtual Method and Keep the Public Method Non-Virtual
