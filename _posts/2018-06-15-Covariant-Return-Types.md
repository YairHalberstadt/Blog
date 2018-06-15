---
layout: post
title: Covariant Return Types in C#
---
## What are Covariant Return Types?

Consider the following problem:

You have a class called Animal

```
public class Animal
{
    Animal GiveBirth() => return new Animal();
}
```

Cat inherits from animal

```
public class Cat : Animal
{
    Cat GiveBirth() => return new Cat();
}
```

Great! That works fine. 
Now what happens when we do this:
```
var undercoverCat = (animal)new Cat();
var babyCat = (cat)undercoverCat().GiveBirth();
```

Oops. We get an InvalidCastException.

If we want GiveBirth to return a new animal with the same type as the underlying type of the original animal when we call GiveBirth, we really have to make the method virtual. We also ought to make Animal abstract, as it doesn't make sense to have an  animal which isnt a specific species.




