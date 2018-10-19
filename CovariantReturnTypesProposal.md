## Design Proposals For Adding Covariant Return Types in C#

### Contents

1.  Description
2.  Test Cases
3.  Design1 (creating a new method as well as a bridging overriding method)
4.  How Design1 plays with other .Net code
5.  Design1 Advantages/Disadvantages
6.  Design2 (using an attribute to indicate the desired return type)
7.  How Design2 plays with other .Net code
8.  Design2 Advantages/Disadvantages
9.  Design3 (explicit virtual method overrides)
10. How Design3 plays with other .Net code
11. Design3 Advantages/Disadvantages
12. Personal Conclusions

### 1. Description

##### Support for covariant return types in derived classes.
Note: I am referring to version 5.0 of the C# specification, as it is the last completed version.

The proposal is to relax the constraint defined in 10.6.4 (override methods):
>A compile-time error occurs unless all of the following are true for an override declaration: 
>
>...
>
>The override method and the overridden base method have the same return type. 

The constraint will be similiarly relaxed for implicit interface implementations.

The constraint will not be relaxed for explicit interface implementations, as to do so would make no difference to consuming code.

The "same return type" constraint" is relaxed using a definition similar to 15.2 (delegate compatibility) for the return type:
>An identity or implicit reference conversion exists from the return type of M to the return type of D.

Thus the new constraint will be

>A compile-time error occurs unless all of the following are true for an override declaration: 
>
>...
>
>An identity or implicit reference conversion exists from the return type of the override method to the return type of the overriden base method.

### 2. Test Cases

Note that all of thse test cases should be repeated for cases where the virtual method has any number of parameters.

**case a - overriding a virtual method**
```csharp
class Program
{
    static void Main(string[] args)
    {
        Animal animal = new Animal();
        var babyAnimal = animal.GiveBirth(); //type of var should be Animal
        babyAnimal.GetType(); // should be Animal
        Dog dog = new Dog();
        Dog babyDog = dog.GiveBirth(); // should compile and run
        var babyDog2 = dog.GiveBirth(); // type of var should be Dog
        babyDog2.GetType(); // should be Dog
        Animal animal2 = dog;
        var babyAnimal2 = animal2.GiveBirth(); //type of var should be Animal
        babyAnimal2.GetType(); // should be Dog
    }
}

public class Animal
{
    public virtual Animal GiveBirth() => new Animal();
}

public class Dog : Animal
{
    public override Dog GiveBirth() => new Dog(); //Should Compile
}
```

**case b - overriding abstract method in abstract class**
```csharp
class Program
{
    static void Main(string[] args)
    {
        Dog dog = new Dog();
        Dog babyDog = dog.GiveBirth(); // should compile and run
        var babyDog2 = dog.GiveBirth(); // type of var should be Dog
        babyDog2.GetType(); // should be Dog
        Animal animal2 = dog;
        var babyAnimal2 = animal2.GiveBirth(); //type of var should be Animal
        babyAnimal2.GetType(); // should be Dog
    }
}

public abstract class Animal
{
    public abstract Animal GiveBirth();
}

public class Dog : Animal
{
    public override Dog GiveBirth() => new Dog(); //Should Compile
}
```

**case c - overriding a virtual method in an abstract class**
```csharp
class Program
{
    static void Main(string[] args)
    {
        Cat cat = new Cat();
        var babyCat = cat.GiveBirth(); // type of var should be Animal
        babyCat.GetType(); // should be Cat
        //Cat babyCat2 = cat.GiveBirth(); // should not compile
        Animal animal = cat;
        var babyAnimal = animal.GiveBirth(); // type of var should be Animal
        babyAnimal.GetType(); // should be Cat
        Dog dog = new Dog();
        Dog babyDog = dog.GiveBirth(); // should compile and run
        var babyDog2 = dog.GiveBirth(); // type of var should be Dog
        babyDog2.GetType(); // should be Dog
        Animal animal2 = dog;
        var babyAnimal2 = animal2.GiveBirth(); //type of var should be Animal
        babyAnimal2.GetType(); // should be Dog
    }
}

public abstract class Animal
{
    public virtual Animal GiveBirth() => new Cat();
}

public class Dog : Animal
{
    public override Dog GiveBirth() => new Dog(); //Should Compile
}

public class Cat : Animal
{
}
```

**case d - overriding an interface method**
```csharp
class Program
{
    static void Main(string[] args)
    {
        Cat cat = new Cat();
        var babyCat = cat.GiveBirth(); // type of var should be Cat
        babyCat.GetType(); // should be Cat
        Cat babyCat2 = cat.GiveBirth(); // should compile
        IAnimal animal = cat;
        var babyAnimal = animal.GiveBirth(); // type of var should be IAnimal
        babyAnimal.GetType(); // should be Dog
        Dog dog = new Dog();
        Dog babyDog = dog.GiveBirth(); // should compile
        var babyDog2 = dog.GiveBirth(); // type of var should be Dog
        babyDog2.GetType(); // should be Dog
        IAnimal animal2 = dog;
        var babyAnimal2 = animal2.GiveBirth(); //type of var should be Animal
        babyAnimal2.GetType(); // should be Dog
    }
}

public interface IAnimal
{
    IAnimal GiveBirth();
}

public class Dog : IAnimal
{
    public Dog GiveBirth() => new Dog(); // Should Compile
}

public class Cat : IAnimal
{
    //Cat IAnimal.GiveBirth() => new Cat(); // Should not Compile

    IAnimal IAnimal.GiveBirth() => new Dog(); // Should Compile

    public Cat GiveBirth() => new Cat(); // Should Compile
}
```

**case e - overriding a covarient override**
```csharp
class Program
{
    static void Main(string[] args)
    {
        Retriever retriever = new Retriever();
        var babyRetriever = retriever.GiveBirth(); // type of var should be Retriever
        Dog dog = retriever;
        var babyDog = dog.GiveBirth(); // Type of var should be Dog
        babyDog.GetType(); // should be Retriever
        Animal animal = retriever;
        var babyAnimal = animal.GiveBirth(); // Type of var should be Animal
        babyAnimal.GetType(); // should be Retriever
    }
}

public class Animal
{
    public virtual Animal GiveBirth() => new Animal();
}

public class Dog : Animal
{
    public override Dog GiveBirth() => new Dog(); //Should Compile
}

public class Poodle : Dog
{
    public override Dog GiveBirth() => new Poodle(); // Should compile
}

public class Retriever : Dog
{
    public override Retriever GiveBirth() => new Retriever(); // Should Compile
}

//public class StBernard : Dog
//{
//    public override Animal GiveBirth() => new StBernard(); // Should not Compile
//}
```

**case f - overriding a covarient abstract override**
```csharp
class Program
{
    static void Main(string[] args)
    {
        Retriever retriever = new Retriever();
        var babyRetriever = retriever.GiveBirth(); // type of var should be Retriever
        Dog dog = retriever;
        var babyDog = dog.GiveBirth(); // Type of var should be Dog
        babyDog.GetType(); // should be Retriever
        Animal animal = retriever;
        var babyAnimal = animal.GiveBirth(); // Type of var should be Animal
        babyAnimal.GetType(); // should be Retriever
    }
}

public abstract class Animal
{
    public abstract Animal GiveBirth();
}

public abstract class Dog : Animal
{
    public abstract override Dog GiveBirth(); //Should Compile
}

public class Poodle : Dog
{
    public override Dog GiveBirth() => new Poodle(); // Should compile
}

public class Retriever : Dog
{
    public override Retriever GiveBirth() => new Retriever(); // Should Compile
}

//public class StBernard : Dog
//{
//    public override Animal GiveBirth() => new StBernard(); // Should not Compile
//}
```

**case g - sealed overrides**

```csharp
class Program
{
    static void Main(string[] args)
    {
        Dog dog = new Dog();
        var babyDog = dog.GiveBirth(); // type of var should be DogRetriever
        Animal animal = dog;
        var babyAnimal = animal.GiveBirth(); // type of var should be Animal
        babyAnimal.GetType(); // should be dog
    }
}

public class Animal
{
    public virtual Animal GiveBirth() => new Animal();
}

public class Dog : Animal
{
    public sealed override Dog GiveBirth() => new Dog(); //Should Compile
}

//public class Poodle : Dog
//{
//    public override Dog GiveBirth() => new Poodle(); // Should not compile
//}

//public class Retriever : Dog
//{
//    public override Retriever GiveBirth() => new Retriever(); // Should not Compile
//}

public class Cat : Animal
{
    public sealed override Animal GiveBirth() => new Cat();
}

//public class Tiger : Cat
//{
//    public override Tiger GiveBirth() => new Tiger(); // Should not compile
//}
```

**case h - attribute inheritance**

```csharp
[AttributeUsage(validOn: AttributeTargets.Method, AllowMultiple = false, Inherited = true)]
public class InheritedAtrributeSingleInstance : Attribute {
    public InheritedAtrributeSingleInstance(int id){}
}

[AttributeUsage(validOn: AttributeTargets.Method, AllowMultiple = true, Inherited = true)]
public class InheritedAtrributeMultipleInstance : Attribute {
    public InheritedAtrributeMultipleInstance(int id) { }
}

[AttributeUsage(validOn: AttributeTargets.Method, AllowMultiple = false, Inherited = false)]
public class NonInheritedAtrributeSingleInstance : Attribute {
    public NonInheritedAtrributeSingleInstance(int id) { }
}

public class Animal
{
    [InheritedAtrributeSingleInstance(0)]
    [InheritedAtrributeMultipleInstance(0)]
    [NonInheritedAtrributeSingleInstance(0)]
    public virtual Animal GiveBirth() => new Animal();
}

public class Dog : Animal
{
    /* Should have following Attributes
     * [InheritedAtrributeSingleInstance(0)]
     * [InheritedAtrributeMultipleInstance(0)]
     */
    public override Dog GiveBirth() => new Dog();
}

public class Poodle : Dog
{
    /* Should have following Attributes
     * [InheritedAtrributeSingleInstance(0)]
     * [InheritedAtrributeMultipleInstance(0)]
     */
    public override Dog GiveBirth() => new Poodle(); 
}

public class Retriever : Dog
{
    /* Should have following Attributes
     * [InheritedAtrributeSingleInstance(0)]
     * [InheritedAtrributeMultipleInstance(0)]
     */
    public override Retriever GiveBirth() => new Retriever();
}

public class StBernard : Dog
{
    [InheritedAtrributeMultipleInstance(1)]
    [InheritedAtrributeSingleInstance(1)]
    /* Should have following Attributes
     * [InheritedAtrributeSingleInstance(1)]
     * [InheritedAtrributeMultipleInstance(1)]
     * [InheritedAtrributeMultipleInstance(0)]
     */
    public override StBernard GiveBirth() => new StBernard();
}

public class Collie : Dog
{
    [InheritedAtrributeMultipleInstance(1)]
    [InheritedAtrributeSingleInstance(1)]
    /* Should have following Attributes
     * [InheritedAtrributeSingleInstance(1)]
     * [InheritedAtrributeMultipleInstance(1)]
     * [InheritedAtrributeMultipleInstance(0)]
     */
    public override Dog GiveBirth() => new Collie();
}

public class Cat : Animal
{
    
    [InheritedAtrributeMultipleInstance(1)]
    [InheritedAtrributeSingleInstance(1)]
    [NonInheritedAtrributeSingleInstance(1)]
    /* Should have following Attributes
     * [InheritedAtrributeSingleInstance(1)]
     * [InheritedAtrributeMultipleInstance(1)]
     * [InheritedAtrributeMultipleInstance(0)]
     * [NonInheritedAtrributeSingleInstance(1)]
     */
    public override Cat GiveBirth() => new Cat();
}

public class Tiger : Cat
{
    /* Should have following Attributes
     * [InheritedAtrributeSingleInstance(1)]
     * [InheritedAtrributeMultipleInstance(1)]
     * [InheritedAtrributeMultipleInstance(0)]
     */
    public override Tiger GiveBirth() => new Tiger();
}

public class Leopard : Cat
{
    /* Should have following Attributes
     * [InheritedAtrributeSingleInstance(1)]
     * [InheritedAtrributeMultipleInstance(1)]
     * [InheritedAtrributeMultipleInstance(0)]
     */
    public override Cat GiveBirth() => new Leopard();
}

public class Cheetah : Cat
{
    [InheritedAtrributeMultipleInstance(2)]
    [InheritedAtrributeSingleInstance(2)]
    /* Should have following Attributes
     * [InheritedAtrributeSingleInstance(2)]
     * [InheritedAtrributeMultipleInstance(2)]
     * [InheritedAtrributeMultipleInstance(1)]
     * [InheritedAtrributeMultipleInstance(0)]
     */
    public override Cheetah GiveBirth() => new Cheetah();
}

public class Jaguar : Cat
{
    [InheritedAtrributeMultipleInstance(2)]
    [InheritedAtrributeSingleInstance(2)]
    /* Should have following Attributes
     * [InheritedAtrributeSingleInstance(2)]
     * [InheritedAtrributeMultipleInstance(2)]
     * [InheritedAtrributeMultipleInstance(1)]
     * [InheritedAtrributeMultipleInstance(0)]
     */
    public override Cat GiveBirth() => new Jaguar();
}
```

**case i - creating a Delegate type**

```csharp
class Program
{
    static void Main(string[] args)
    {
        var dog = new Dog();
        Func<Dog> dogFunc = dog.GiveBirth; //should compile
        var babyDog = FunctionApplier(dog.GiveBirth); //type of var should be Dog
    }

    static T FunctionApplier<T>(Func<T> func) => func();
}

public class Animal
{
    public virtual Animal GiveBirth() => new Animal();
}

public class Dog : Animal
{
    public override Dog GiveBirth() => new Dog();
}
```

**case j - implementing an interface which requires the more derived return type**

```csharp
public interface IDog
{
    Dog GiveBirth();
}

public class Animal
{
    public virtual Animal GiveBirth() => new Animal();
}

public class Dog : Animal, IDog //Should Compile
{
    public override Dog GiveBirth() => new Dog();
}
```

**case k - Generic return types**

```csharp
public abstract class Factory<T>
{
    public abstract T Create();
}

public abstract class DerivedFactory<TDerived, TBase> : Factory<TBase> where TDerived : TBase
{
    public abstract override TDerived Create(); //Should Compile
}

public class Animal
{
}

public class Dog : Animal
{
}

public class DogFactory : Factory<Animal>
{
    public override Dog Create() => new Dog(); //should compile
}
```

### 3. Design1 (creating a new method as well as a bridging overriding method)

#### Central Idea

Consider the code from test case a once more:
```csharp
public class Animal
{
    public virtual Animal GiveBirth() => new Animal();
}

public class Dog : Animal
{
    public override Dog GiveBirth() => new Dog();
}
```

In IL it is possible to override a method with a differently named method.
Hence in the IL for class Dog we create a private sealed method GiveBirth'() that overrides Animal.GiveBirth();
GiveBirth' calls a new virtual method GiveBirth() that returns a Dog.

Hence code calling Dog.GiveBirth() will call the new virtual method that returns a Dog.

This is similiar to a technique already used to achieve Covariant return types with interfaces:
```csharp
public interface IAnimal
{
    Animal GiveBirth();
}

public class Dog : IAnimal
{
    public Dog GiveBirth() => new Dog();
    
    Animal Animal.GiveBirth() => GiveBirth();
}
```

#### Generated IL for all Test Cases

Note all IL has been tested using https://www.tutorialspoint.com/compile_ilasm_online.php

**case a**
```csharp
.assembly Covariant {}
.assembly extern mscorlib {}
.class private auto ansi beforefieldinit Program
    extends [mscorlib]System.Object
{
    // Methods
    .method private hidebysig static 
        void Main (
            string[] args
        ) cil managed 
    {
        // Method begins at RVA 0x2050
        // Code size 71 (0x47)
        .entrypoint
        .maxstack 1
        .locals init (
            [0] class Animal,
            [1] class Animal,
            [2] class Dog,
            [3] class Dog,
            [4] class Dog,
            [5] class Animal,
            [6] class Animal
        )

        IL_0000: nop
        IL_0001: newobj instance void Animal::.ctor()
        IL_0006: stloc.0
        IL_0007: ldloc.0
        IL_0008: callvirt instance class Animal Animal::GiveBirth()
        IL_000d: stloc.1
        IL_000e: ldloc.1
        IL_000f: callvirt instance class [mscorlib]System.Type [mscorlib]System.Object::GetType()
        IL_0014: pop
        IL_0015: newobj instance void Dog::.ctor()
        IL_001a: stloc.2
        IL_001b: ldloc.2
        IL_001c: callvirt instance class Dog Dog::GiveBirth()
        IL_0021: stloc.3
        IL_0022: ldloc.2
        IL_0023: callvirt instance class Dog Dog::GiveBirth()
        IL_0028: stloc.s 4
        IL_002a: ldloc.s 4
        IL_002c: callvirt instance class [mscorlib]System.Type [mscorlib]System.Object::GetType()
        IL_0031: pop
        IL_0032: ldloc.2
        IL_0033: stloc.s 5
        IL_0035: ldloc.s 5
        IL_0037: callvirt instance class Animal Animal::GiveBirth()
        IL_003c: stloc.s 6
        IL_003e: ldloc.s 6
        IL_0040: callvirt instance class [mscorlib]System.Type [mscorlib]System.Object::GetType()
        IL_0045: pop
        IL_0046: ret
    } // end of method Program::Main

    .method public hidebysig specialname rtspecialname 
        instance void .ctor () cil managed 
    {
        // Method begins at RVA 0x20a3
        // Code size 8 (0x8)
        .maxstack 8

        IL_0000: ldarg.0
        IL_0001: call instance void [mscorlib]System.Object::.ctor()
        IL_0006: nop
        IL_0007: ret
    } // end of method Program::.ctor

} // end of class Program


.class public auto ansi beforefieldinit Animal
    extends [mscorlib]System.Object
{
    // Methods
    .method public hidebysig newslot virtual 
        instance class Animal GiveBirth () cil managed 
    {
        // Method begins at RVA 0x2050
        // Code size 6 (0x6)
        .maxstack 8

        IL_0000: newobj instance void Animal::.ctor()
        IL_0005: ret
    } // end of method Animal::GiveBirth

    .method public hidebysig specialname rtspecialname 
        instance void .ctor () cil managed 
    {
        // Method begins at RVA 0x2057
        // Code size 8 (0x8)
        .maxstack 8

        IL_0000: ldarg.0
        IL_0001: call instance void [mscorlib]System.Object::.ctor()
        IL_0006: nop
        IL_0007: ret
    } // end of method Animal::.ctor

} // end of class Animal

.class public auto ansi beforefieldinit Dog
    extends Animal
{
    // Methods
    .method private final hidebysig virtual 
        instance class Animal Animal.GiveBirth () cil managed 
    {
    
        .override Animal::GiveBirth
        // Method begins at RVA 0x2058
        // Code size 7 (0x7)
        .maxstack  8
        .locals init (object V_0)

        IL_0000:  nop
        IL_0001:  ldarg.0
        IL_0002:  tail.
        IL_0004:  callvirt   instance class Dog Dog::GiveBirth()
        IL_0009:  ret
    } // end of method Dog::Animal.GiveBirth

    .method public hidebysig newslot virtual 
        instance class Dog GiveBirth() cil managed
    {
        // Method begins at RVA 0x2060
        // Code size 6 (0x6)
        .maxstack 8

        IL_0000: newobj instance void Dog::.ctor()
        IL_0005: ret
    }// end of method Dog::GiveBirth

    .method public hidebysig specialname rtspecialname 
        instance void .ctor () cil managed 
    {
        // Method begins at RVA 0x2067
        // Code size 8 (0x8)
        .maxstack 8

        IL_0000: ldarg.0
        IL_0001: call instance void Animal::.ctor()
        IL_0006: nop
        IL_0007: ret
    } // end of method Dog::.ctor

} // end of class Dog

```

**case b**
```csharp
.assembly Covariant { }
.assembly extern mscorlib {}
.class private auto ansi beforefieldinit Program
    extends [mscorlib]System.Object
{
    // Methods
    .method private hidebysig static 
        void Main (
            string[] args
        ) cil managed 
    {
        // Method begins at RVA 0x2050
        // Code size 47 (0x2f)
        .entrypoint
        .maxstack 1
        .locals init (
            [0] class Dog,
            [1] class Dog,
            [2] class Animal,
            [3] class Animal,
            [4] class Animal
        )

        IL_0000: nop
        IL_0001: newobj instance void Dog::.ctor()
        IL_0006: stloc.0
        IL_0007: ldloc.0
        IL_0008: callvirt instance class Dog Dog::GiveBirth()
        IL_000d: stloc.1
        IL_000e: ldloc.0
        IL_000f: callvirt instance class Dog Dog::GiveBirth()
        IL_0014: stloc.2
        IL_0015: ldloc.2
        IL_0016: callvirt instance class [mscorlib]System.Type [mscorlib]System.Object::GetType()
        IL_001b: pop
        IL_001c: ldloc.0
        IL_001d: stloc.3
        IL_001e: ldloc.3
        IL_001f: callvirt instance class Animal Animal::GiveBirth()
        IL_0024: stloc.s 4
        IL_0026: ldloc.s 4
        IL_0028: callvirt instance class [mscorlib]System.Type [mscorlib]System.Object::GetType()
        IL_002d: pop
        IL_002e: ret
    } // end of method Program::Main

    .method public hidebysig specialname rtspecialname 
        instance void .ctor () cil managed 
    {
        // Method begins at RVA 0x208b
        // Code size 8 (0x8)
        .maxstack 8

        IL_0000: ldarg.0
        IL_0001: call instance void [mscorlib]System.Object::.ctor()
        IL_0006: nop
        IL_0007: ret
    } // end of method Program::.ctor

} // end of class Program

.class public auto ansi abstract beforefieldinit Animal
    extends [mscorlib]System.Object
{
    // Methods
    .method public hidebysig newslot abstract virtual 
        instance class Animal GiveBirth () cil managed 
    {
    } // end of method Animal::GiveBirth

    .method family hidebysig specialname rtspecialname 
        instance void .ctor () cil managed 
    {
        // Method begins at RVA 0x208b
        // Code size 8 (0x8)
        .maxstack 8

        IL_0000: ldarg.0
        IL_0001: call instance void [mscorlib]System.Object::.ctor()
        IL_0006: nop
        IL_0007: ret
    } // end of method Animal::.ctor

} // end of class Animal

.class public auto ansi beforefieldinit Dog
    extends Animal
{
    // Methods
    .method private final hidebysig virtual
        instance class Animal Animal.GiveBirth () cil managed 
    {
        .override Animal::GiveBirth
        // Method begins at RVA 0x2094
        // Code size 7 (0x7)
        .maxstack 8

        IL_0000: ldarg.0
        IL_0001: call instance class Dog Dog::GiveBirth()
        IL_0006: ret
    } // end of method Dog::Animal.GiveBirth

    .method public hidebysig newslot virtual
        instance class Dog GiveBirth () cil managed 
    {
        // Method begins at RVA 0x209c
        // Code size 6 (0x6)
        .maxstack 8

        IL_0000: newobj instance void Dog::.ctor()
        IL_0005: ret
    } // end of method Dog::GiveBirth

    .method public hidebysig specialname rtspecialname 
        instance void .ctor () cil managed 
    {
        // Method begins at RVA 0x20a3
        // Code size 8 (0x8)
        .maxstack 8

        IL_0000: ldarg.0
        IL_0001: call instance void Animal::.ctor()
        IL_0006: nop
        IL_0007: ret
    } // end of method Dog::.ctor

} // end of class Dog
```

**case c**

```csharp
.assembly Covariant {}
.assembly extern mscorlib {}
.class private auto ansi beforefieldinit Program
    extends [mscorlib]System.Object
{
    // Methods
    .method private hidebysig static 
        void Main (
            string[] args
        ) cil managed 
    {
        // Method begins at RVA 0x2050
        // Code size 92 (0x5c)
        .entrypoint
        .maxstack 1
        .locals init (
            [0] class Cat,
            [1] class Animal,
            [2] class Animal,
            [3] class Animal,
            [4] class Dog,
            [5] class Dog,
            [6] class Dog,
            [7] class Animal,
            [8] class Animal
        )

        IL_0000: nop
        IL_0001: newobj instance void Cat::.ctor()
        IL_0006: stloc.0
        IL_0007: ldloc.0
        IL_0008: callvirt instance class Animal Animal::GiveBirth()
        IL_000d: stloc.1
        IL_000e: ldloc.1
        IL_000f: callvirt instance class [mscorlib]System.Type [mscorlib]System.Object::GetType()
        IL_0014: pop
        IL_0015: ldloc.0
        IL_0016: stloc.2
        IL_0017: ldloc.2
        IL_0018: callvirt instance class Animal Animal::GiveBirth()
        IL_001d: stloc.3
        IL_001e: ldloc.3
        IL_001f: callvirt instance class [mscorlib]System.Type [mscorlib]System.Object::GetType()
        IL_0024: pop
        IL_0025: newobj instance void Dog::.ctor()
        IL_002a: stloc.s 4
        IL_002c: ldloc.s 4
        IL_002e: callvirt instance class Dog Dog::GiveBirth()
        IL_0033: stloc.s 5
        IL_0035: ldloc.s 4
        IL_0037: callvirt instance class Dog Dog::GiveBirth()
        IL_003c: stloc.s 6
        IL_003e: ldloc.s 6
        IL_0040: callvirt instance class [mscorlib]System.Type [mscorlib]System.Object::GetType()
        IL_0045: pop
        IL_0046: ldloc.s 4
        IL_0048: stloc.s 7
        IL_004a: ldloc.s 7
        IL_004c: callvirt instance class Animal Animal::GiveBirth()
        IL_0051: stloc.s 8
        IL_0053: ldloc.s 8
        IL_0055: callvirt instance class [mscorlib]System.Type [mscorlib]System.Object::GetType()
        IL_005a: pop
        IL_005b: ret
    } // end of method Program::Main

    .method public hidebysig specialname rtspecialname 
        instance void .ctor () cil managed 
    {
        // Method begins at RVA 0x20b8
        // Code size 8 (0x8)
        .maxstack 8

        IL_0000: ldarg.0
        IL_0001: call instance void [mscorlib]System.Object::.ctor()
        IL_0006: nop
        IL_0007: ret
    } // end of method Program::.ctor

} // end of class Program

.class public auto ansi abstract beforefieldinit Animal
    extends [mscorlib]System.Object
{
    // Methods
    .method public hidebysig newslot virtual 
        instance class Animal GiveBirth () cil managed 
    {
        // Method begins at RVA 0x20c1
        // Code size 6 (0x6)
        .maxstack 8

        IL_0000: newobj instance void Cat::.ctor()
        IL_0005: ret
    } // end of method Animal::GiveBirth

    .method family hidebysig specialname rtspecialname 
        instance void .ctor () cil managed 
    {
        // Method begins at RVA 0x20b8
        // Code size 8 (0x8)
        .maxstack 8

        IL_0000: ldarg.0
        IL_0001: call instance void [mscorlib]System.Object::.ctor()
        IL_0006: nop
        IL_0007: ret
    } // end of method Animal::.ctor

} // end of class Animal

.class public auto ansi beforefieldinit Dog
    extends Animal
{
    // Methods
    .method private final hidebysig virtual 
        instance class Animal Animal.GiveBirth () cil managed 
    {
    
        .override Animal::GiveBirth
        // Method begins at RVA 0x20c8
        // Code size 7 (0x7)
        .maxstack 8

        IL_0000: ldarg.0
        IL_0001: call instance class Dog Dog::GiveBirth()
        IL_0006: ret
    } // end of method Dog::Animal.GiveBirth

    .method public hidebysig newslot virtual
        instance class Dog GiveBirth () cil managed 
    {
        // Method begins at RVA 0x20d0
        // Code size 6 (0x6)
        .maxstack 8

        IL_0000: newobj instance void Dog::.ctor()
        IL_0005: ret
    } // end of method Dog::GiveBirth

    .method public hidebysig specialname rtspecialname 
        instance void .ctor () cil managed 
    {
        // Method begins at RVA 0x20d7
        // Code size 8 (0x8)
        .maxstack 8

        IL_0000: ldarg.0
        IL_0001: call instance void Animal::.ctor()
        IL_0006: nop
        IL_0007: ret
    } // end of method Dog::.ctor

} // end of class Dog

.class public auto ansi beforefieldinit Cat
    extends Animal
{
    // Methods
    .method public hidebysig specialname rtspecialname 
        instance void .ctor () cil managed 
    {
        // Method begins at RVA 0x20d7
        // Code size 8 (0x8)
        .maxstack 8

        IL_0000: ldarg.0
        IL_0001: call instance void Animal::.ctor()
        IL_0006: nop
        IL_0007: ret
    } // end of method Cat::.ctor

} // end of class Cat
```

**case d**

This case produces exactly the same IL as if we'd used explicit interface implementations. Thus 

```csharp
public interface IAnimal
{
    IAnimal GiveBirth();
}

public class Dog : IAnimal
{
    public Dog GiveBirth() => new Dog();
}
```

Is translated to the same IL as

```csharp
public interface IAnimal
{
    IAnimal GiveBirth();
}

public class Dog : IAnimal
{
    IAnimal IAnimal.GiveBirth() => GiveBirth();
    
    public Dog GiveBirth() => new Dog();
}
```

Here is the generated IL for the test case:

```csharp
.assembly Covariant { }
.assembly extern mscorlib {}
.class private auto ansi beforefieldinit Program
    extends [mscorlib]System.Object
{
    // Methods
    .method private hidebysig static 
        void Main (
            string[] args
        ) cil managed 
    {
        // Method begins at RVA 0x2050
        // Code size 101 (0x65)
	.entrypoint
        .maxstack 1
        .locals init (
            [0] class Cat,
            [1] class Cat,
            [2] class Cat,
            [3] class IAnimal,
            [4] class IAnimal,
            [5] class Dog,
            [6] class Dog,
            [7] class Dog,
            [8] class IAnimal,
            [9] class IAnimal
        )

        IL_0000: nop
        IL_0001: newobj instance void Cat::.ctor()
        IL_0006: stloc.0
        IL_0007: ldloc.0
        IL_0008: callvirt instance class Cat Cat::GiveBirth()
        IL_000d: stloc.1
        IL_000e: ldloc.1
        IL_000f: callvirt instance class [mscorlib]System.Type [mscorlib]System.Object::GetType()
        IL_0014: pop
        IL_0015: ldloc.0
        IL_0016: callvirt instance class Cat Cat::GiveBirth()
        IL_001b: stloc.2
        IL_001c: ldloc.0
        IL_001d: stloc.3
        IL_001e: ldloc.3
        IL_001f: callvirt instance class IAnimal IAnimal::GiveBirth()
        IL_0024: stloc.s 4
        IL_0026: ldloc.s 4
        IL_0028: callvirt instance class [mscorlib]System.Type [mscorlib]System.Object::GetType()
        IL_002d: pop
        IL_002e: newobj instance void Dog::.ctor()
        IL_0033: stloc.s 5
        IL_0035: ldloc.s 5
        IL_0037: callvirt instance class Dog Dog::GiveBirth()
        IL_003c: stloc.s 6
        IL_003e: ldloc.s 5
        IL_0040: callvirt instance class Dog Dog::GiveBirth()
        IL_0045: stloc.s 7
        IL_0047: ldloc.s 7
        IL_0049: callvirt instance class [mscorlib]System.Type [mscorlib]System.Object::GetType()
        IL_004e: pop
        IL_004f: ldloc.s 5
        IL_0051: stloc.s 8
        IL_0053: ldloc.s 8
        IL_0055: callvirt instance class IAnimal IAnimal::GiveBirth()
        IL_005a: stloc.s 9
        IL_005c: ldloc.s 9
        IL_005e: callvirt instance class [mscorlib]System.Type [mscorlib]System.Object::GetType()
        IL_0063: pop
        IL_0064: ret
    } // end of method Program::Main

    .method public hidebysig specialname rtspecialname 
        instance void .ctor () cil managed 
    {
        // Method begins at RVA 0x20c1
        // Code size 8 (0x8)
        .maxstack 8

        IL_0000: ldarg.0
        IL_0001: call instance void [mscorlib]System.Object::.ctor()
        IL_0006: nop
        IL_0007: ret
    } // end of method Program::.ctor

} // end of class Program

.class interface public auto ansi abstract IAnimal
{
    // Methods
    .method public hidebysig newslot abstract virtual 
        instance class IAnimal GiveBirth () cil managed 
    {
    } // end of method IAnimal::GiveBirth

} // end of class IAnimal

.class public auto ansi beforefieldinit Dog
    extends [mscorlib]System.Object
    implements IAnimal
{
    // Methods
    .method private final hidebysig newslot virtual 
        instance class IAnimal IAnimal.GiveBirth () cil managed 
    {
        .override method instance class IAnimal IAnimal::GiveBirth()
        // Method begins at RVA 0x20ca
        // Code size 7 (0x7)
        .maxstack 8

        IL_0000: ldarg.0
        IL_0001: call instance class Dog Dog::GiveBirth()
        IL_0006: ret
    } // end of method Dog::IAnimal.GiveBirth

    .method public hidebysig 
        instance class Dog GiveBirth () cil managed 
    {
        // Method begins at RVA 0x20d2
        // Code size 6 (0x6)
        .maxstack 8

        IL_0000: newobj instance void Dog::.ctor()
        IL_0005: ret
    } // end of method Dog::GiveBirth

    .method public hidebysig specialname rtspecialname 
        instance void .ctor () cil managed 
    {
        // Method begins at RVA 0x20c1
        // Code size 8 (0x8)
        .maxstack 8

        IL_0000: ldarg.0
        IL_0001: call instance void [mscorlib]System.Object::.ctor()
        IL_0006: nop
        IL_0007: ret
    } // end of method Dog::.ctor

} // end of class Dog

.class public auto ansi beforefieldinit Cat
    extends [mscorlib]System.Object
    implements IAnimal
{
    // Methods
    .method private final hidebysig newslot virtual 
        instance class IAnimal IAnimal.GiveBirth () cil managed 
    {
        .override method instance class IAnimal IAnimal::GiveBirth()
        // Method begins at RVA 0x20d2
        // Code size 6 (0x6)
        .maxstack 8

        IL_0000: newobj instance void Dog::.ctor()
        IL_0005: ret
    } // end of method Cat::IAnimal.GiveBirth

    .method public hidebysig 
        instance class Cat GiveBirth () cil managed 
    {
        // Method begins at RVA 0x20d9
        // Code size 6 (0x6)
        .maxstack 8

        IL_0000: newobj instance void Cat::.ctor()
        IL_0005: ret
    } // end of method Cat::GiveBirth

    .method public hidebysig specialname rtspecialname 
        instance void .ctor () cil managed 
    {
        // Method begins at RVA 0x20c1
        // Code size 8 (0x8)
        .maxstack 8

        IL_0000: ldarg.0
        IL_0001: call instance void [mscorlib]System.Object::.ctor()
        IL_0006: nop
        IL_0007: ret
    } // end of method Cat::.ctor

} // end of class Cat

```

**case e**

Note that an extra method `Retriever::Animal.GiveBirth` is inserted into `Retriever` that overides `Animal::GiveBirth` directly.

Whilst not strictly neccessary from a functional perspective, this increases performance, as it means only one extra function call will occur, even in a large chain of overrides with covariant return types.

Thus when a `Retriever` is cast to an `Animal`, and `Animal::GiveBirth` is called, the virtual function call is resolved to `Retriever::Animal.GiveBirth`, which then delegates to `Retriever::GiveBirth` directly.

If we didn't have this extra method, the virtual function call would have been resolved to `Dog::Animal.GiveBirth` which would have delegated to `Dog::GiveBirth` which would have resolved to `Retriever::Dog.GiveBirth` which would have delegated to `Retriever::GiveBirth`. This extra virtual function call could degrade performance. Thus the addition of `Retriever::Animal.GiveBirth`.

For each extra step in the chain of covariant overrides, an extra method will be inserted into the most derived class overriding each newslot method in the chain of parent methods.

```csharp
.assembly Covariant {}
.assembly extern mscorlib {}
.class private auto ansi beforefieldinit Program
    extends [mscorlib]System.Object
{
    // Methods
    .method private hidebysig static 
        void Main (
            string[] args
        ) cil managed 
    {
        // Method begins at RVA 0x2050
        // Code size 51 (0x33)
        .entrypoint
        .maxstack 1
        .locals init (
            [0] class Retriever,
            [1] class Retriever,
            [2] class Dog,
            [3] class Dog,
            [4] class Animal,
            [5] class Animal
        )

        IL_0000: nop
        IL_0001: newobj instance void Retriever::.ctor()
        IL_0006: stloc.0
        IL_0007: ldloc.0
        IL_0008: callvirt instance class Retriever Retriever::GiveBirth()
        IL_000d: stloc.1
        IL_000e: ldloc.0
        IL_000f: stloc.2
        IL_0010: ldloc.2
        IL_0011: callvirt instance class Dog Dog::GiveBirth()
        IL_0016: stloc.3
        IL_0017: ldloc.3
        IL_0018: callvirt instance class [mscorlib]System.Type [mscorlib]System.Object::GetType()
        IL_001d: pop
        IL_001e: ldloc.0
        IL_001f: stloc.s 4
        IL_0021: ldloc.s 4
        IL_0023: callvirt instance class Animal Animal::GiveBirth()
        IL_0028: stloc.s 5
        IL_002a: ldloc.s 5
        IL_002c: callvirt instance class [mscorlib]System.Type [mscorlib]System.Object::GetType()
        IL_0031: pop
        IL_0032: ret
    } // end of method Program::Main

    .method public hidebysig specialname rtspecialname 
        instance void .ctor () cil managed 
    {
        // Method begins at RVA 0x208f
        // Code size 8 (0x8)
        .maxstack 8

        IL_0000: ldarg.0
        IL_0001: call instance void [mscorlib]System.Object::.ctor()
        IL_0006: nop
        IL_0007: ret
    } // end of method Program::.ctor

} // end of class Program

.class public auto ansi beforefieldinit Animal
    extends [mscorlib]System.Object
{
    // Methods
    .method public hidebysig newslot virtual 
        instance class Animal GiveBirth () cil managed 
    {
        // Method begins at RVA 0x2098
        // Code size 6 (0x6)
        .maxstack 8

        IL_0000: newobj instance void Animal::.ctor()
        IL_0005: ret
    } // end of method Animal::GiveBirth

    .method public hidebysig specialname rtspecialname 
        instance void .ctor () cil managed 
    {
        // Method begins at RVA 0x208f
        // Code size 8 (0x8)
        .maxstack 8

        IL_0000: ldarg.0
        IL_0001: call instance void [mscorlib]System.Object::.ctor()
        IL_0006: nop
        IL_0007: ret
    } // end of method Animal::.ctor

} // end of class Animal

.class public auto ansi beforefieldinit Dog
    extends Animal
{
    // Methods
    .method private final hidebysig virtual 
        instance class Animal Animal.GiveBirth () cil managed 
    {
    
        .override Animal::GiveBirth
        // Method begins at RVA 0x209f
        // Code size 7 (0x7)
        .maxstack 8

        IL_0000: ldarg.0
        IL_0001: callvirt instance class Dog Dog::GiveBirth()
        IL_0006: ret
    } // end of method Dog::Animal.GiveBirth

    .method public hidebysig newslot virtual 
        instance class Dog GiveBirth () cil managed 
    {
        // Method begins at RVA 0x20a7
        // Code size 6 (0x6)
        .maxstack 8

        IL_0000: newobj instance void Dog::.ctor()
        IL_0005: ret
    } // end of method Dog::GiveBirth

    .method public hidebysig specialname rtspecialname 
        instance void .ctor () cil managed 
    {
        // Method begins at RVA 0x20ae
        // Code size 8 (0x8)
        .maxstack 8

        IL_0000: ldarg.0
        IL_0001: call instance void Animal::.ctor()
        IL_0006: nop
        IL_0007: ret
    } // end of method Dog::.ctor

} // end of class Dog

.class public auto ansi beforefieldinit Poodle
    extends Dog
{
    // Methods
    .method public hidebysig virtual 
        instance class Dog GiveBirth () cil managed 
    {
        // Method begins at RVA 0x20b7
        // Code size 6 (0x6)
        .maxstack 8

        IL_0000: newobj instance void Poodle::.ctor()
        IL_0005: ret
    } // end of method Poodle::GiveBirth

    .method public hidebysig specialname rtspecialname 
        instance void .ctor () cil managed 
    {
        // Method begins at RVA 0x20be
        // Code size 8 (0x8)
        .maxstack 8

        IL_0000: ldarg.0
        IL_0001: call instance void Dog::.ctor()
        IL_0006: nop
        IL_0007: ret
    } // end of method Poodle::.ctor

} // end of class Poodle

.class public auto ansi beforefieldinit Retriever
    extends Dog
{
    // Methods
    .method private final hidebysig virtual 
        instance class Animal Animal.GiveBirth () cil managed 
    {
    
        .override Animal::GiveBirth
        // Method begins at RVA 0x209f
        // Code size 7 (0x7)
        .maxstack 8

        IL_0000: ldarg.0
        IL_0001: callvirt instance class Retriever Retriever::GiveBirth()
        IL_0006: ret
    } // end of method Retriever::Animal.GiveBirth
    
    .method private final hidebysig virtual 
        instance class Dog Dog.GiveBirth () cil managed 
    {
    
        .override Dog::GiveBirth
        // Method begins at RVA 0x20c7
        // Code size 7 (0x7)
        .maxstack 8

        IL_0000: ldarg.0
        IL_0001: callvirt instance class Retriever Retriever::GiveBirth()
        IL_0006: ret
    } // end of method Retriever::Dog.GiveBirth

    .method public hidebysig newslot virtual 
        instance class Retriever GiveBirth () cil managed 
    {
        // Method begins at RVA 0x20cf
        // Code size 6 (0x6)
        .maxstack 8

        IL_0000: newobj instance void Retriever::.ctor()
        IL_0005: ret
    } // end of method Retriever::GiveBirth

    .method public hidebysig specialname rtspecialname 
        instance void .ctor () cil managed 
    {
        // Method begins at RVA 0x20be
        // Code size 8 (0x8)
        .maxstack 8

        IL_0000: ldarg.0
        IL_0001: call instance void Dog::.ctor()
        IL_0006: nop
        IL_0007: ret
    } // end of method Retriever::.ctor

} // end of class Retriever
```

**case f**

Note that an extra method `Retriever::Animal.GiveBirth` is inserted into `Retriever` that overides `Animal::GiveBirth` directly.

Whilst not strictly neccessary from a functional perspective, this increases performance, as it means only one extra function call will occur, even in a large chain of overrides with covariant return types.

Thus when a `Retriever` is cast to an `Animal`, and `Animal::GiveBirth` is called, the virtual function call is resolved to `Retriever::Animal.GiveBirth`, which then delegates to `Retriever::GiveBirth` directly.

If we didn't have this extra method, the virtual function call would have been resolved to `Dog::Animal.GiveBirth` which would have delegated to `Dog::GiveBirth` which would have resolved to `Retriever::Dog.GiveBirth` which would have delegated to `Retriever::GiveBirth`. This extra virtual function call could degrade performance. Thus the addition of `Retriever::Animal.GiveBirth`.

For each extra step in the chain of covariant overrides, an extra method will be inserted into the most derived class overriding each newslot method in the chain of parent methods.

```csharp
.assembly Covariant {}
.assembly extern mscorlib {}
.class private auto ansi beforefieldinit Program
    extends [mscorlib]System.Object
{
    // Methods
    .method private hidebysig static 
        void Main (
            string[] args
        ) cil managed 
    {
        // Method begins at RVA 0x2050
        // Code size 51 (0x33)
        .entrypoint
        .maxstack 1
        .locals init (
            [0] class Retriever,
            [1] class Animal,
            [2] class Dog,
            [3] class Animal,
            [4] class Animal,
            [5] class Animal
        )

        IL_0000: nop
        IL_0001: newobj instance void Retriever::.ctor()
        IL_0006: stloc.0
        IL_0007: ldloc.0
        IL_0008: callvirt instance class Animal Animal::GiveBirth()
        IL_000d: stloc.1
        IL_000e: ldloc.0
        IL_000f: stloc.2
        IL_0010: ldloc.2
        IL_0011: callvirt instance class Animal Animal::GiveBirth()
        IL_0016: stloc.3
        IL_0017: ldloc.3
        IL_0018: callvirt instance class [mscorlib]System.Type [mscorlib]System.Object::GetType()
        IL_001d: pop
        IL_001e: ldloc.0
        IL_001f: stloc.s 4
        IL_0021: ldloc.s 4
        IL_0023: callvirt instance class Animal Animal::GiveBirth()
        IL_0028: stloc.s 5
        IL_002a: ldloc.s 5
        IL_002c: callvirt instance class [mscorlib]System.Type [mscorlib]System.Object::GetType()
        IL_0031: pop
        IL_0032: ret
    } // end of method Program::Main

    .method public hidebysig specialname rtspecialname 
        instance void .ctor () cil managed 
    {
        // Method begins at RVA 0x208f
        // Code size 8 (0x8)
        .maxstack 8

        IL_0000: ldarg.0
        IL_0001: call instance void [mscorlib]System.Object::.ctor()
        IL_0006: nop
        IL_0007: ret
    } // end of method Program::.ctor

} // end of class Program

.class public auto ansi abstract beforefieldinit Animal
    extends [mscorlib]System.Object
{
    // Methods
    .method public hidebysig newslot abstract virtual 
        instance class Animal GiveBirth () cil managed 
    {
    } // end of method Animal::GiveBirth

    .method family hidebysig specialname rtspecialname 
        instance void .ctor () cil managed 
    {
        // Method begins at RVA 0x208f
        // Code size 8 (0x8)
        .maxstack 8

        IL_0000: ldarg.0
        IL_0001: call instance void [mscorlib]System.Object::.ctor()
        IL_0006: nop
        IL_0007: ret
    } // end of method Animal::.ctor

} // end of class Animal

.class public auto ansi abstract beforefieldinit Dog
    extends Animal
{
    // Methods
    .method private final hidebysig virtual 
        instance class Animal Animal.GiveBirth () cil managed 
    {
        .override Animal::GiveBirth
        // Method begins at RVA 0x2098
        // Code size 7 (0x7)
        .maxstack 8

        IL_0000: ldarg.0
        IL_0001: callvirt instance class Dog Dog::GiveBirth()
        IL_0006: ret
    } // end of method Dog::Animal.GiveBirth

    .method public hidebysig newslot abstract virtual 
        instance class Dog GiveBirth () cil managed 
    {
    } // end of method Dog::GiveBirth

    .method family hidebysig specialname rtspecialname 
        instance void .ctor () cil managed 
    {
        // Method begins at RVA 0x20a0
        // Code size 8 (0x8)
        .maxstack 8

        IL_0000: ldarg.0
        IL_0001: call instance void Animal::.ctor()
        IL_0006: nop
        IL_0007: ret
    } // end of method Dog::.ctor

} // end of class Dog

.class public auto ansi beforefieldinit Poodle
    extends Dog
{
    // Methods
    .method public hidebysig virtual 
        instance class Dog GiveBirth () cil managed 
    {
        // Method begins at RVA 0x20a9
        // Code size 6 (0x6)
        .maxstack 8

        IL_0000: newobj instance void Poodle::.ctor()
        IL_0005: ret
    } // end of method Poodle::GiveBirth

    .method public hidebysig specialname rtspecialname 
        instance void .ctor () cil managed 
    {
        // Method begins at RVA 0x20b0
        // Code size 8 (0x8)
        .maxstack 8

        IL_0000: ldarg.0
        IL_0001: call instance void Dog::.ctor()
        IL_0006: nop
        IL_0007: ret
    } // end of method Poodle::.ctor

} // end of class Poodle

.class public auto ansi beforefieldinit Retriever
    extends Dog
{
    // Methods
        .method private final hidebysig virtual 
        instance class Animal Animal.GiveBirth () cil managed 
    {
    
        .override Animal::GiveBirth
        // Method begins at RVA 0x209f
        // Code size 7 (0x7)
        .maxstack 8

        IL_0000: ldarg.0
        IL_0001: callvirt instance class Retriever Retriever::GiveBirth()
        IL_0006: ret
    } // end of method Retriever::Animal.GiveBirth
    
    .method private final hidebysig virtual 
        instance class Dog Dog.GiveBirth () cil managed 
    {
    
        .override Dog::GiveBirth
        // Method begins at RVA 0x20c7
        // Code size 7 (0x7)
        .maxstack 8

        IL_0000: ldarg.0
        IL_0001: callvirt instance class Retriever Retriever::GiveBirth()
        IL_0006: ret
    } // end of method Retriever::Dog.GiveBirth

    .method public hidebysig newslot virtual 
        instance class Retriever GiveBirth () cil managed 
    {
        // Method begins at RVA 0x20cf
        // Code size 6 (0x6)
        .maxstack 8

        IL_0000: newobj instance void Retriever::.ctor()
        IL_0005: ret
    } // end of method Retriever::GiveBirth

    .method public hidebysig specialname rtspecialname 
        instance void .ctor () cil managed 
    {
        // Method begins at RVA 0x20b0
        // Code size 8 (0x8)
        .maxstack 8

        IL_0000: ldarg.0
        IL_0001: call instance void Dog::.ctor()
        IL_0006: nop
        IL_0007: ret
    } // end of method Retriever::.ctor

} // end of class Retriever
```

**case g**
```csharp
.assembly Covariant {}
.assembly extern mscorlib {}
.class private auto ansi beforefieldinit Program
    extends [mscorlib]System.Object
{
    // Methods
    .method private hidebysig static 
        void Main (
            string[] args
        ) cil managed 
    {
        // Method begins at RVA 0x2050
        // Code size 31 (0x1f)
        .entrypoint
        .maxstack 1
        .locals init (
            [0] class Dog,
            [1] class Animal,
            [2] class Animal,
            [3] class Animal
        )

        IL_0000: nop
        IL_0001: newobj instance void Dog::.ctor()
        IL_0006: stloc.0
        IL_0007: ldloc.0
        IL_0008: callvirt instance class Animal Animal::GiveBirth()
        IL_000d: stloc.1
        IL_000e: ldloc.0
        IL_000f: stloc.2
        IL_0010: ldloc.2
        IL_0011: callvirt instance class Animal Animal::GiveBirth()
        IL_0016: stloc.3
        IL_0017: ldloc.3
        IL_0018: callvirt instance class [mscorlib]System.Type [mscorlib]System.Object::GetType()
        IL_001d: pop
        IL_001e: ret
    } // end of method Program::Main

    .method public hidebysig specialname rtspecialname 
        instance void .ctor () cil managed 
    {
        // Method begins at RVA 0x207b
        // Code size 8 (0x8)
        .maxstack 8

        IL_0000: ldarg.0
        IL_0001: call instance void [mscorlib]System.Object::.ctor()
        IL_0006: nop
        IL_0007: ret
    } // end of method Program::.ctor

} // end of class Program

.class public auto ansi beforefieldinit Animal
    extends [mscorlib]System.Object
{
    // Methods
    .method public hidebysig newslot virtual 
        instance class Animal GiveBirth () cil managed 
    {
        // Method begins at RVA 0x2084
        // Code size 6 (0x6)
        .maxstack 8

        IL_0000: newobj instance void Animal::.ctor()
        IL_0005: ret
    } // end of method Animal::GiveBirth

    .method public hidebysig specialname rtspecialname 
        instance void .ctor () cil managed 
    {
        // Method begins at RVA 0x207b
        // Code size 8 (0x8)
        .maxstack 8

        IL_0000: ldarg.0
        IL_0001: call instance void [mscorlib]System.Object::.ctor()
        IL_0006: nop
        IL_0007: ret
    } // end of method Animal::.ctor

} // end of class Animal

.class public auto ansi beforefieldinit Dog
    extends Animal
{
    // Methods
    .method private final hidebysig virtual 
        instance class Animal Animal.GiveBirth () cil managed 
    {
        .override Animal::GiveBirth
        // Method begins at RVA 0x208b
        // Code size 7 (0x7)
        .maxstack 8

        IL_0000: ldarg.0
        IL_0001: call instance class Dog Dog::GiveBirth()
        IL_0006: ret
    } // end of method Dog::Animal.GiveBirth

    .method public hidebysig 
        instance class Dog GiveBirth () cil managed 
    {
        // Method begins at RVA 0x2093
        // Code size 6 (0x6)
        .maxstack 8

        IL_0000: newobj instance void Dog::.ctor()
        IL_0005: ret
    } // end of method Dog::GiveBirth

    .method public hidebysig specialname rtspecialname 
        instance void .ctor () cil managed 
    {
        // Method begins at RVA 0x209a
        // Code size 8 (0x8)
        .maxstack 8

        IL_0000: ldarg.0
        IL_0001: call instance void Animal::.ctor()
        IL_0006: nop
        IL_0007: ret
    } // end of method Dog::.ctor

} // end of class Dog

.class public auto ansi beforefieldinit Cat
    extends Animal
{
    // Methods
    .method public final hidebysig virtual 
        instance class Animal GiveBirth () cil managed 
    {
        // Method begins at RVA 0x20a3
        // Code size 6 (0x6)
        .maxstack 8

        IL_0000: newobj instance void Cat::.ctor()
        IL_0005: ret
    } // end of method Cat::GiveBirth

    .method public hidebysig specialname rtspecialname 
        instance void .ctor () cil managed 
    {
        // Method begins at RVA 0x209a
        // Code size 8 (0x8)
        .maxstack 8

        IL_0000: ldarg.0
        IL_0001: call instance void Animal::.ctor()
        IL_0006: nop
        IL_0007: ret
    } // end of method Cat::.ctor

} // end of class Cat
```

**case h**

In this case we have to manually add in the attributes to the relevant methods. Whilst this works fine when all the methods are int he same assembly, it could cause issues when they are in multiple assemblies.

```csharp
.assembly Covariant {}
.assembly extern mscorlib {}
.class public auto ansi beforefieldinit InheritedAtrributeSingleInstance
    extends [mscorlib]System.Attribute
{
    .custom instance void [mscorlib]System.AttributeUsageAttribute::.ctor(valuetype [mscorlib]System.AttributeTargets) = (
        01 00 40 00 00 00 02 00 54 02 0d 41 6c 6c 6f 77
        4d 75 6c 74 69 70 6c 65 00 54 02 09 49 6e 68 65
        72 69 74 65 64 01
    )
    // Methods
    .method public hidebysig specialname rtspecialname 
        instance void .ctor (
            int32 id
        ) cil managed 
    {
        // Method begins at RVA 0x2050
        // Code size 9 (0x9)
        .maxstack 8

        IL_0000: ldarg.0
        IL_0001: call instance void [mscorlib]System.Attribute::.ctor()
        IL_0006: nop
        IL_0007: nop
        IL_0008: ret
    } // end of method InheritedAtrributeSingleInstance::.ctor

} // end of class InheritedAtrributeSingleInstance

.class public auto ansi beforefieldinit InheritedAtrributeMultipleInstance
    extends [mscorlib]System.Attribute
{
    .custom instance void [mscorlib]System.AttributeUsageAttribute::.ctor(valuetype [mscorlib]System.AttributeTargets) = (
        01 00 40 00 00 00 02 00 54 02 0d 41 6c 6c 6f 77
        4d 75 6c 74 69 70 6c 65 01 54 02 09 49 6e 68 65
        72 69 74 65 64 01
    )
    // Methods
    .method public hidebysig specialname rtspecialname 
        instance void .ctor (
            int32 id
        ) cil managed 
    {
        // Method begins at RVA 0x2050
        // Code size 9 (0x9)
        .maxstack 8

        IL_0000: ldarg.0
        IL_0001: call instance void [mscorlib]System.Attribute::.ctor()
        IL_0006: nop
        IL_0007: nop
        IL_0008: ret
    } // end of method InheritedAtrributeMultipleInstance::.ctor

} // end of class InheritedAtrributeMultipleInstance

.class public auto ansi beforefieldinit NonInheritedAtrributeSingleInstance
    extends [mscorlib]System.Attribute
{
    .custom instance void [mscorlib]System.AttributeUsageAttribute::.ctor(valuetype [mscorlib]System.AttributeTargets) = (
        01 00 40 00 00 00 02 00 54 02 0d 41 6c 6c 6f 77
        4d 75 6c 74 69 70 6c 65 00 54 02 09 49 6e 68 65
        72 69 74 65 64 00
    )
    // Methods
    .method public hidebysig specialname rtspecialname 
        instance void .ctor (
            int32 id
        ) cil managed 
    {
        // Method begins at RVA 0x2050
        // Code size 9 (0x9)
        .maxstack 8

        IL_0000: ldarg.0
        IL_0001: call instance void [mscorlib]System.Attribute::.ctor()
        IL_0006: nop
        IL_0007: nop
        IL_0008: ret
    } // end of method NonInheritedAtrributeSingleInstance::.ctor

} // end of class NonInheritedAtrributeSingleInstance

.class public auto ansi beforefieldinit Animal
    extends [mscorlib]System.Object
{
    // Methods
    .method public hidebysig newslot virtual 
        instance class Animal GiveBirth () cil managed 
    {
        .custom instance void InheritedAtrributeSingleInstance::.ctor(int32) = (
            01 00 00 00 00 00 00 00
        )
        .custom instance void InheritedAtrributeMultipleInstance::.ctor(int32) = (
            01 00 00 00 00 00 00 00
        )
        .custom instance void NonInheritedAtrributeSingleInstance::.ctor(int32) = (
            01 00 00 00 00 00 00 00
        )
        // Method begins at RVA 0x205a
        // Code size 6 (0x6)
        .maxstack 8

        IL_0000: newobj instance void Animal::.ctor()
        IL_0005: ret
    } // end of method Animal::GiveBirth

    .method public hidebysig specialname rtspecialname 
        instance void .ctor () cil managed 
    {
        // Method begins at RVA 0x2061
        // Code size 8 (0x8)
        .maxstack 8

        IL_0000: ldarg.0
        IL_0001: call instance void [mscorlib]System.Object::.ctor()
        IL_0006: nop
        IL_0007: ret
    } // end of method Animal::.ctor

} // end of class Animal

.class public auto ansi beforefieldinit Dog
    extends Animal
{
    // Methods
    .method private final hidebysig virtual 
        instance class Animal Animal.GiveBirth () cil managed 
    {
        .override Animal::GiveBirth
        // Method begins at RVA 0x206a
        // Code size 7 (0x7)
        .maxstack 8

        IL_0000: ldarg.0
        IL_0001: callvirt instance class Dog Dog::GiveBirth()
        IL_0006: ret
    } // end of method Dog::Animal.GiveBirth

    .method public hidebysig newslot virtual 
        instance class Dog GiveBirth () cil managed 
    {
        .custom instance void InheritedAtrributeSingleInstance::.ctor(int32) = (
            01 00 00 00 00 00 00 00
        )
        .custom instance void InheritedAtrributeMultipleInstance::.ctor(int32) = (
            01 00 00 00 00 00 00 00
        )
        // Method begins at RVA 0x2072
        // Code size 6 (0x6)
        .maxstack 8

        IL_0000: newobj instance void Dog::.ctor()
        IL_0005: ret
    } // end of method Dog::GiveBirth

    .method public hidebysig specialname rtspecialname 
        instance void .ctor () cil managed 
    {
        // Method begins at RVA 0x2079
        // Code size 8 (0x8)
        .maxstack 8

        IL_0000: ldarg.0
        IL_0001: call instance void Animal::.ctor()
        IL_0006: nop
        IL_0007: ret
    } // end of method Dog::.ctor

} // end of class Dog

.class public auto ansi beforefieldinit Poodle
    extends Dog
{
    // Methods
    .method public hidebysig virtual 
        instance class Dog GiveBirth () cil managed 
    {
        // Method begins at RVA 0x2082
        // Code size 6 (0x6)
        .maxstack 8

        IL_0000: newobj instance void Poodle::.ctor()
        IL_0005: ret
    } // end of method Poodle::GiveBirth

    .method public hidebysig specialname rtspecialname 
        instance void .ctor () cil managed 
    {
        // Method begins at RVA 0x2089
        // Code size 8 (0x8)
        .maxstack 8

        IL_0000: ldarg.0
        IL_0001: call instance void Dog::.ctor()
        IL_0006: nop
        IL_0007: ret
    } // end of method Poodle::.ctor

} // end of class Poodle

.class public auto ansi beforefieldinit Retriever
    extends Dog
{
    // Methods
    .method private final hidebysig virtual 
        instance class Animal Animal.GiveBirth () cil managed 
    {
        .override Animal::GiveBirth
        // Method begins at RVA 0x2092
        // Code size 7 (0x7)
        .maxstack 8

        IL_0000: ldarg.0
        IL_0001: callvirt instance class Retriever Retriever::GiveBirth()
        IL_0006: ret
    } // end of method Retriever::Animal.GiveBirth
    
    .method private final hidebysig virtual 
        instance class Dog Dog.GiveBirth () cil managed 
    {
        .override Dog::GiveBirth
        // Method begins at RVA 0x2092
        // Code size 7 (0x7)
        .maxstack 8

        IL_0000: ldarg.0
        IL_0001: callvirt instance class Retriever Retriever::GiveBirth()
        IL_0006: ret
    } // end of method Retriever::Dog.GiveBirth

    .method public hidebysig newslot virtual 
        instance class Retriever GiveBirth () cil managed 
    {
        .custom instance void InheritedAtrributeSingleInstance::.ctor(int32) = (
            01 00 00 00 00 00 00 00
        )
        .custom instance void InheritedAtrributeMultipleInstance::.ctor(int32) = (
            01 00 00 00 00 00 00 00
        )
        // Method begins at RVA 0x209a
        // Code size 6 (0x6)
        .maxstack 8

        IL_0000: newobj instance void Retriever::.ctor()
        IL_0005: ret
    } // end of method Retriever::GiveBirth

    .method public hidebysig specialname rtspecialname 
        instance void .ctor () cil managed 
    {
        // Method begins at RVA 0x2089
        // Code size 8 (0x8)
        .maxstack 8

        IL_0000: ldarg.0
        IL_0001: call instance void Dog::.ctor()
        IL_0006: nop
        IL_0007: ret
    } // end of method Retriever::.ctor

} // end of class Retriever

.class public auto ansi beforefieldinit StBernard
    extends Dog
{
    // Methods
    .method private final hidebysig virtual 
        instance class Animal Animal.GiveBirth () cil managed 
    {
        .custom instance void InheritedAtrributeMultipleInstance::.ctor(int32) = (
            01 00 01 00 00 00 00 00
        )
        .custom instance void InheritedAtrributeSingleInstance::.ctor(int32) = (
            01 00 01 00 00 00 00 00
        )
        .override Animal::GiveBirth
        // Method begins at RVA 0x20a1
        // Code size 7 (0x7)
        .maxstack 8

        IL_0000: ldarg.0
        IL_0001: callvirt instance class StBernard StBernard::GiveBirth()
        IL_0006: ret
    } // end of method StBernard::Animal.GiveBirth
    
    .method private final hidebysig virtual 
        instance class Dog Dog.GiveBirth () cil managed 
    {
        .custom instance void InheritedAtrributeMultipleInstance::.ctor(int32) = (
            01 00 01 00 00 00 00 00
        )
        .custom instance void InheritedAtrributeSingleInstance::.ctor(int32) = (
            01 00 01 00 00 00 00 00
        )
        .override Dog::GiveBirth
        // Method begins at RVA 0x20a1
        // Code size 7 (0x7)
        .maxstack 8

        IL_0000: ldarg.0
        IL_0001: callvirt instance class StBernard StBernard::GiveBirth()
        IL_0006: ret
    } // end of method StBernard::Dog.GiveBirth

    .method public hidebysig newslot virtual 
        instance class StBernard GiveBirth () cil managed 
    {
        .custom instance void InheritedAtrributeSingleInstance::.ctor(int32) = (
            01 00 01 00 00 00 00 00
        )
        .custom instance void InheritedAtrributeMultipleInstance::.ctor(int32) = (
            01 00 01 00 00 00 00 00
        )
        .custom instance void InheritedAtrributeMultipleInstance::.ctor(int32) = (
            01 00 00 00 00 00 00 00
        )
        // Method begins at RVA 0x20a9
        // Code size 6 (0x6)
        .maxstack 8

        IL_0000: newobj instance void StBernard::.ctor()
        IL_0005: ret
    } // end of method StBernard::GiveBirth

    .method public hidebysig specialname rtspecialname 
        instance void .ctor () cil managed 
    {
        // Method begins at RVA 0x2089
        // Code size 8 (0x8)
        .maxstack 8

        IL_0000: ldarg.0
        IL_0001: call instance void Dog::.ctor()
        IL_0006: nop
        IL_0007: ret
    } // end of method StBernard::.ctor

} // end of class StBernard

.class public auto ansi beforefieldinit Collie
    extends Dog
{
    // Methods
    .method public hidebysig virtual 
        instance class Dog GiveBirth () cil managed 
    {
        .custom instance void InheritedAtrributeMultipleInstance::.ctor(int32) = (
            01 00 01 00 00 00 00 00
        )
        .custom instance void InheritedAtrributeSingleInstance::.ctor(int32) = (
            01 00 01 00 00 00 00 00
        )
        // Method begins at RVA 0x20b0
        // Code size 6 (0x6)
        .maxstack 8

        IL_0000: newobj instance void Collie::.ctor()
        IL_0005: ret
    } // end of method Collie::GiveBirth

    .method public hidebysig specialname rtspecialname 
        instance void .ctor () cil managed 
    {
        // Method begins at RVA 0x2089
        // Code size 8 (0x8)
        .maxstack 8

        IL_0000: ldarg.0
        IL_0001: call instance void Dog::.ctor()
        IL_0006: nop
        IL_0007: ret
    } // end of method Collie::.ctor

} // end of class Collie

.class public auto ansi beforefieldinit Cat
    extends Animal
{
    // Methods
    .method private final hidebysig virtual 
        instance class Animal Animal.GiveBirth () cil managed 
    {
        .custom instance void InheritedAtrributeMultipleInstance::.ctor(int32) = (
            01 00 01 00 00 00 00 00
        )
        .custom instance void InheritedAtrributeSingleInstance::.ctor(int32) = (
            01 00 01 00 00 00 00 00
        )
        .custom instance void NonInheritedAtrributeSingleInstance::.ctor(int32) = (
            01 00 01 00 00 00 00 00
        )
        .override Animal::GiveBirth
        // Method begins at RVA 0x20b7
        // Code size 7 (0x7)
        .maxstack 8

        IL_0000: ldarg.0
        IL_0001: callvirt instance class Cat Cat::GiveBirth()
        IL_0006: ret
    } // end of method Cat::Animal.GiveBirth

    .method public hidebysig newslot virtual 
        instance class Cat GiveBirth () cil managed 
    {
        .custom instance void InheritedAtrributeSingleInstance::.ctor(int32) = (
            01 00 01 00 00 00 00 00
        )
        .custom instance void InheritedAtrributeMultipleInstance::.ctor(int32) = (
            01 00 01 00 00 00 00 00
        )
        .custom instance void InheritedAtrributeMultipleInstance::.ctor(int32) = (
            01 00 00 00 00 00 00 00
        )
        .custom instance void NonInheritedAtrributeSingleInstance::.ctor(int32) = (
            01 00 01 00 00 00 00 00
        )
        // Method begins at RVA 0x20bf
        // Code size 6 (0x6)
        .maxstack 8

        IL_0000: newobj instance void Cat::.ctor()
        IL_0005: ret
    } // end of method Cat::GiveBirth

    .method public hidebysig specialname rtspecialname 
        instance void .ctor () cil managed 
    {
        // Method begins at RVA 0x2079
        // Code size 8 (0x8)
        .maxstack 8

        IL_0000: ldarg.0
        IL_0001: call instance void Animal::.ctor()
        IL_0006: nop
        IL_0007: ret
    } // end of method Cat::.ctor

} // end of class Cat

.class public auto ansi beforefieldinit Tiger
    extends Cat
{
    // Methods
    .method private final hidebysig virtual 
        instance class Animal Animal.GiveBirth () cil managed 
    {
        .custom instance void InheritedAtrributeSingleInstance::.ctor(int32) = (
            01 00 01 00 00 00 00 00
        )
        .custom instance void InheritedAtrributeMultipleInstance::.ctor(int32) = (
            01 00 01 00 00 00 00 00
        )
        .custom instance void InheritedAtrributeMultipleInstance::.ctor(int32) = (
            01 00 00 00 00 00 00 00
        )
        .override Animal::GiveBirth
        // Method begins at RVA 0x20c6
        // Code size 7 (0x7)
        .maxstack 8

        IL_0000: ldarg.0
        IL_0001: callvirt instance class Tiger Tiger::GiveBirth()
        IL_0006: ret
    } // end of method Tiger::Animal.GiveBirth
    
    .method private final hidebysig virtual 
        instance class Cat Cat.GiveBirth () cil managed 
    {
        .override Cat::GiveBirth
        // Method begins at RVA 0x20c6
        // Code size 7 (0x7)
        .maxstack 8

        IL_0000: ldarg.0
        IL_0001: callvirt instance class Tiger Tiger::GiveBirth()
        IL_0006: ret
    } // end of method Tiger::Cat.GiveBirth

    .method public hidebysig newslot virtual 
        instance class Tiger GiveBirth () cil managed 
    {
        .custom instance void InheritedAtrributeSingleInstance::.ctor(int32) = (
            01 00 01 00 00 00 00 00
        )
        .custom instance void InheritedAtrributeMultipleInstance::.ctor(int32) = (
            01 00 01 00 00 00 00 00
        )
        .custom instance void InheritedAtrributeMultipleInstance::.ctor(int32) = (
            01 00 00 00 00 00 00 00
        )
        // Method begins at RVA 0x20ce
        // Code size 6 (0x6)
        .maxstack 8

        IL_0000: newobj instance void Tiger::.ctor()
        IL_0005: ret
    } // end of method Tiger::GiveBirth

    .method public hidebysig specialname rtspecialname 
        instance void .ctor () cil managed 
    {
        // Method begins at RVA 0x20d5
        // Code size 8 (0x8)
        .maxstack 8

        IL_0000: ldarg.0
        IL_0001: call instance void Cat::.ctor()
        IL_0006: nop
        IL_0007: ret
    } // end of method Tiger::.ctor

} // end of class Tiger

.class public auto ansi beforefieldinit Leopard
    extends Cat
{
    // Methods
    .method public hidebysig virtual 
        instance class Cat GiveBirth () cil managed 
    {
        // Method begins at RVA 0x20de
        // Code size 6 (0x6)
        .maxstack 8

        IL_0000: newobj instance void Leopard::.ctor()
        IL_0005: ret
    } // end of method Leopard::GiveBirth

    .method public hidebysig specialname rtspecialname 
        instance void .ctor () cil managed 
    {
        // Method begins at RVA 0x20d5
        // Code size 8 (0x8)
        .maxstack 8

        IL_0000: ldarg.0
        IL_0001: call instance void Cat::.ctor()
        IL_0006: nop
        IL_0007: ret
    } // end of method Leopard::.ctor

} // end of class Leopard

.class public auto ansi beforefieldinit Cheetah
    extends Cat
{
    // Methods
    .method private final hidebysig virtual 
        instance class Animal Animal.GiveBirth () cil managed 
    {
        .custom instance void InheritedAtrributeMultipleInstance::.ctor(int32) = (
            01 00 01 00 00 00 00 00
        )
        .custom instance void InheritedAtrributeMultipleInstance::.ctor(int32) = (
            01 00 00 00 00 00 00 00
        )
        .custom instance void InheritedAtrributeMultipleInstance::.ctor(int32) = (
            01 00 02 00 00 00 00 00
        )
        .custom instance void InheritedAtrributeSingleInstance::.ctor(int32) = (
            01 00 02 00 00 00 00 00
        )
        .override Animal::GiveBirth
        // Method begins at RVA 0x20e5
        // Code size 7 (0x7)
        .maxstack 8

        IL_0000: ldarg.0
        IL_0001: callvirt instance class Cheetah Cheetah::GiveBirth()
        IL_0006: ret
    } // end of method Cheetah::Animal.GiveBirth
    
    .method private final hidebysig virtual 
        instance class Cat Cat.GiveBirth () cil managed 
    {
        .custom instance void InheritedAtrributeMultipleInstance::.ctor(int32) = (
            01 00 02 00 00 00 00 00
        )
        .custom instance void InheritedAtrributeSingleInstance::.ctor(int32) = (
            01 00 02 00 00 00 00 00
        )
        .override Cat::GiveBirth
        // Method begins at RVA 0x20e5
        // Code size 7 (0x7)
        .maxstack 8

        IL_0000: ldarg.0
        IL_0001: callvirt instance class Cheetah Cheetah::GiveBirth()
        IL_0006: ret
    } // end of method Cheetah::Cat.GiveBirth

    .method public hidebysig newslot virtual 
        instance class Cheetah GiveBirth () cil managed 
    {
        .custom instance void InheritedAtrributeSingleInstance::.ctor(int32) = (
            01 00 02 00 00 00 00 00
        )
        .custom instance void InheritedAtrributeMultipleInstance::.ctor(int32) = (
            01 00 02 00 00 00 00 00
        )
        .custom instance void InheritedAtrributeMultipleInstance::.ctor(int32) = (
            01 00 01 00 00 00 00 00
        )
        .custom instance void InheritedAtrributeMultipleInstance::.ctor(int32) = (
            01 00 00 00 00 00 00 00
        )
        // Method begins at RVA 0x20ed
        // Code size 6 (0x6)
        .maxstack 8

        IL_0000: newobj instance void Cheetah::.ctor()
        IL_0005: ret
    } // end of method Cheetah::GiveBirth

    .method public hidebysig specialname rtspecialname 
        instance void .ctor () cil managed 
    {
        // Method begins at RVA 0x20d5
        // Code size 8 (0x8)
        .maxstack 8

        IL_0000: ldarg.0
        IL_0001: call instance void Cat::.ctor()
        IL_0006: nop
        IL_0007: ret
    } // end of method Cheetah::.ctor

} // end of class Cheetah

.class public auto ansi beforefieldinit Jaguar
    extends Cat
{
    // Methods
    .method public hidebysig virtual 
        instance class Cat GiveBirth () cil managed 
    {
        .custom instance void InheritedAtrributeMultipleInstance::.ctor(int32) = (
            01 00 02 00 00 00 00 00
        )
        .custom instance void InheritedAtrributeSingleInstance::.ctor(int32) = (
            01 00 02 00 00 00 00 00
        )
        // Method begins at RVA 0x20f4
        // Code size 6 (0x6)
        .maxstack 8

        IL_0000: newobj instance void Jaguar::.ctor()
        IL_0005: ret
    } // end of method Jaguar::GiveBirth

    .method public hidebysig specialname rtspecialname 
        instance void .ctor () cil managed 
    {
        // Method begins at RVA 0x20d5
        // Code size 8 (0x8)
        .maxstack 8

        IL_0000: ldarg.0
        IL_0001: call instance void Cat::.ctor()
        IL_0006: nop
        IL_0007: ret
    } // end of method Jaguar::.ctor

} // end of class Jaguar
```

**case i**
```csharp
.assembly Covariant {}
.assembly extern mscorlib {}
.class private auto ansi beforefieldinit Program
    extends [mscorlib]System.Object
{
    // Methods
    .method private hidebysig static 
        void Main (
            string[] args
        ) cil managed 
    {
        // Method begins at RVA 0x2050
        // Code size 41 (0x29)
        .entrypoint
        .maxstack 2
        .locals init (
            [0] class Dog,
            [1] class [mscorlib]System.Func`1<class Dog>,
            [2] class Dog
        )

        IL_0000: nop
        IL_0001: newobj instance void Dog::.ctor()
        IL_0006: stloc.0
        IL_0007: ldloc.0
        IL_0008: dup
        IL_0009: ldvirtftn instance class Dog Dog::GiveBirth()
        IL_000f: newobj instance void class [mscorlib]System.Func`1<class Dog>::.ctor(object, native int)
        IL_0014: stloc.1
        IL_0015: ldloc.0
        IL_0016: dup
        IL_0017: ldvirtftn instance class Dog Dog::GiveBirth()
        IL_001d: newobj instance void class [mscorlib]System.Func`1<class Dog>::.ctor(object, native int)
        IL_0022: call !!0 Program::FunctionApplier<class Dog>(class [mscorlib]System.Func`1<!!0>)
        IL_0027: stloc.2
        IL_0028: ret
    } // end of method Program::Main

    .method private hidebysig static 
        !!T FunctionApplier<T> (
            class [mscorlib]System.Func`1<!!T> func
        ) cil managed 
    {
        // Method begins at RVA 0x2085
        // Code size 7 (0x7)
        .maxstack 8

        IL_0000: ldarg.0
        IL_0001: callvirt instance !0 class [mscorlib]System.Func`1<!!T>::Invoke()
        IL_0006: ret
    } // end of method Program::FunctionApplier

    .method public hidebysig specialname rtspecialname 
        instance void .ctor () cil managed 
    {
        // Method begins at RVA 0x208d
        // Code size 8 (0x8)
        .maxstack 8

        IL_0000: ldarg.0
        IL_0001: call instance void [mscorlib]System.Object::.ctor()
        IL_0006: nop
        IL_0007: ret
    } // end of method Program::.ctor

} // end of class Program

.class public auto ansi beforefieldinit Animal
    extends [mscorlib]System.Object
{
    // Methods
    .method public hidebysig newslot virtual 
        instance class Animal GiveBirth () cil managed 
    {
        // Method begins at RVA 0x2096
        // Code size 6 (0x6)
        .maxstack 8

        IL_0000: newobj instance void Animal::.ctor()
        IL_0005: ret
    } // end of method Animal::GiveBirth

    .method public hidebysig specialname rtspecialname 
        instance void .ctor () cil managed 
    {
        // Method begins at RVA 0x208d
        // Code size 8 (0x8)
        .maxstack 8

        IL_0000: ldarg.0
        IL_0001: call instance void [mscorlib]System.Object::.ctor()
        IL_0006: nop
        IL_0007: ret
    } // end of method Animal::.ctor

} // end of class Animal

.class public auto ansi beforefieldinit Dog
    extends Animal
{
    // Methods
    .method private final hidebysig virtual 
        instance class Animal Animal.GiveBirth () cil managed 
    {
        .override Animal::GiveBirth
        // Method begins at RVA 0x209d
        // Code size 7 (0x7)
        .maxstack 8

        IL_0000: ldarg.0
        IL_0001: callvirt instance class Dog Dog::GiveBirth()
        IL_0006: ret
    } // end of method Dog::Animal.GiveBirth

    .method public hidebysig newslot virtual 
        instance class Dog GiveBirth () cil managed 
    {
        // Method begins at RVA 0x20a5
        // Code size 6 (0x6)
        .maxstack 8

        IL_0000: newobj instance void Dog::.ctor()
        IL_0005: ret
    } // end of method Dog::GiveBirth

    .method public hidebysig specialname rtspecialname 
        instance void .ctor () cil managed 
    {
        // Method begins at RVA 0x20ac
        // Code size 8 (0x8)
        .maxstack 8

        IL_0000: ldarg.0
        IL_0001: call instance void Animal::.ctor()
        IL_0006: nop
        IL_0007: ret
    } // end of method Dog::.ctor

} // end of class Dog
```

**case j**
```csharp
.assembly Covariant {}
.assembly extern mscorlib {}
.class interface public auto ansi abstract IDog
{
    // Methods
    .method public hidebysig newslot abstract virtual 
        instance class Dog GiveBirth () cil managed 
    {
    } // end of method IDog::GiveBirth

} // end of class IDog

.class public auto ansi beforefieldinit Animal
    extends [mscorlib]System.Object
{
    // Methods
    .method public hidebysig newslot virtual 
        instance class Animal GiveBirth () cil managed 
    {
        // Method begins at RVA 0x2096
        // Code size 6 (0x6)
        .maxstack 8

        IL_0000: newobj instance void Animal::.ctor()
        IL_0005: ret
    } // end of method Animal::GiveBirth

    .method public hidebysig specialname rtspecialname 
        instance void .ctor () cil managed 
    {
        // Method begins at RVA 0x208d
        // Code size 8 (0x8)
        .maxstack 8

        IL_0000: ldarg.0
        IL_0001: call instance void [mscorlib]System.Object::.ctor()
        IL_0006: nop
        IL_0007: ret
    } // end of method Animal::.ctor

} // end of class Animal

.class public auto ansi beforefieldinit Dog
    extends Animal
    implements IDog
{
    // Methods
    .method private final hidebysig virtual 
        instance class Animal Animal.GiveBirth () cil managed 
    {
        .override Animal::GiveBirth
        // Method begins at RVA 0x209d
        // Code size 7 (0x7)
        .maxstack 8

        IL_0000: ldarg.0
        IL_0001: callvirt instance class Dog Dog::GiveBirth()
        IL_0006: ret
    } // end of method Dog::Animal.GiveBirth

    .method public hidebysig newslot virtual 
        instance class Dog GiveBirth () cil managed 
    {
        // Method begins at RVA 0x20a5
        // Code size 6 (0x6)
        .maxstack 8

        IL_0000: newobj instance void Dog::.ctor()
        IL_0005: ret
    } // end of method Dog::GiveBirth

    .method public hidebysig specialname rtspecialname 
        instance void .ctor () cil managed 
    {
        // Method begins at RVA 0x20ac
        // Code size 8 (0x8)
        .maxstack 8

        IL_0000: ldarg.0
        IL_0001: call instance void Animal::.ctor()
        IL_0006: nop
        IL_0007: ret
    } // end of method Dog::.ctor

} // end of class Dog
```

**case k**
```csharp
.assembly Covariant {}
.assembly extern mscorlib {}
.class public auto ansi abstract beforefieldinit Factory`1<T>
    extends [mscorlib]System.Object
{
    // Methods
    .method public hidebysig newslot abstract virtual 
        instance !T Create () cil managed 
    {
    } // end of method Factory`1::Create

    .method family hidebysig specialname rtspecialname 
        instance void .ctor () cil managed 
    {
        // Method begins at RVA 0x2050
        // Code size 8 (0x8)
        .maxstack 8

        IL_0000: ldarg.0
        IL_0001: call instance void [mscorlib]System.Object::.ctor()
        IL_0006: nop
        IL_0007: ret
    } // end of method Factory`1::.ctor

} // end of class Factory`1

.class public auto ansi abstract beforefieldinit DerivedFactory`2<(!TBase) TDerived, TBase>
    extends class Factory`1<!TBase>
{
    // Methods
    .method private final hidebysig virtual 
        instance !TBase 'Factory<TBase>.Create' () cil managed 
    {
        .override method instance !0 class Factory`1<!TBase>::Create()
        // Method begins at RVA 0x2059
        // Code size 17 (0x11)
        .maxstack 8

        IL_0000: ldarg.0
        IL_0001: callvirt instance !0 class DerivedFactory`2<!TDerived, !TBase>::Create()
        IL_0006: box !TDerived
        IL_000b: unbox.any !TBase
        IL_0010: ret
    } // end of method DerivedFactory`2::'Factory<TBase>.Create'

    .method public hidebysig newslot abstract virtual 
        instance !TDerived Create () cil managed 
    {
    } // end of method DerivedFactory`2::Create

    .method family hidebysig specialname rtspecialname 
        instance void .ctor () cil managed 
    {
        // Method begins at RVA 0x206b
        // Code size 8 (0x8)
        .maxstack 8

        IL_0000: ldarg.0
        IL_0001: call instance void class Factory`1<!TBase>::.ctor()
        IL_0006: nop
        IL_0007: ret
    } // end of method DerivedFactory`2::.ctor

} // end of class DerivedFactory`2

.class public auto ansi beforefieldinit Animal
    extends [mscorlib]System.Object
{
    // Methods
    .method public hidebysig specialname rtspecialname 
        instance void .ctor () cil managed 
    {
        // Method begins at RVA 0x2050
        // Code size 8 (0x8)
        .maxstack 8

        IL_0000: ldarg.0
        IL_0001: call instance void [mscorlib]System.Object::.ctor()
        IL_0006: nop
        IL_0007: ret
    } // end of method Animal::.ctor

} // end of class Animal

.class public auto ansi beforefieldinit Dog
    extends Animal
{
    // Methods
    .method public hidebysig specialname rtspecialname 
        instance void .ctor () cil managed 
    {
        // Method begins at RVA 0x2074
        // Code size 8 (0x8)
        .maxstack 8

        IL_0000: ldarg.0
        IL_0001: call instance void Animal::.ctor()
        IL_0006: nop
        IL_0007: ret
    } // end of method Dog::.ctor

} // end of class Dog

.class public auto ansi beforefieldinit DogFactory
    extends class Factory`1<class Animal>
{
    // Methods
    .method private final hidebysig virtual 
        instance class Animal 'Factory<Animal>.Create' () cil managed 
    {
        .override method instance !0 class Factory`1<class Animal>::Create()
        // Method begins at RVA 0x207d
        // Code size 7 (0x7)
        .maxstack 8

        IL_0000: ldarg.0
        IL_0001: callvirt instance class Dog DogFactory::Create()
        IL_0006: ret
    } // end of method DogFactory::'Factory<Animal>.Create'

    .method public hidebysig newslot virtual 
        instance class Dog Create () cil managed 
    {
        // Method begins at RVA 0x2085
        // Code size 6 (0x6)
        .maxstack 8

        IL_0000: newobj instance void Dog::.ctor()
        IL_0005: ret
    } // end of method DogFactory::Create

    .method public hidebysig specialname rtspecialname 
        instance void .ctor () cil managed 
    {
        // Method begins at RVA 0x208c
        // Code size 8 (0x8)
        .maxstack 8

        IL_0000: ldarg.0
        IL_0001: call instance void class Factory`1<class Animal>::.ctor()
        IL_0006: nop
        IL_0007: ret
    } // end of method DogFactory::.ctor

} // end of class DogFactory
```

### 4. How Design1 Plays with other .Net code

One of the chief advantages of Design1 is that all changes occur at the point where the IL for the covariant return type is produced, and there are no changes at all required in consuming code.

As such this Design will produce IL that is equally compatible with VB.net, F#, and older versions of C# as it will be with newer versions of C#.

This can be, and has been, tested right now by handwriting assemblies using this design, and calling these handwritten assemblies from other projects. They work exactly as expected, and whats more intellisense and object explorer both work fine.

As will be discussed in the next section, in the long run it may be advantageous to introduce changes in the runtime, tooling, core library, etc. to make this feature work more smoothly. Obviously code running on older runtimes wont be able to benefit from these features. However, as will be explained, these are not core parts of the feature, and as such I don't see this as a problem.

### 5. Design1 Advantages/Disadvantages

#### Advantages

**1. Requires only local changes**

The only place where any changes will have to be made to the compiler are when compiling overriding methods with covariant return types. As such this minimises the impact of the change.

**2. Fully Backwards Compatible**

As discussed in Section 4. there is no need for consuming code to be aware of this feature. As such older versions of C#, as well as other .Net languages can consume code using this feature in exactly the same way as the new version of C#.

**3. Avoids Boxing**

In the past Code such as this would have had to be written

```csharp
public class Producer
{
    public virtual object Produce() => new object();
}

public class Int32Producer : Producer
{
    public override object Produce() => 42;
}

class Program
{
    static void Main(string[] args)
    {
         int number = (int) new Int32Producer().Produce();
    }
}
```
This leads to the boxing and unboxing of `42` which can lead to performance issues and stress on the garbage collector.

Whereas now this can be written

```csharp
public class Producer
{
    public virtual object Produce() => new object();
}

public class Int32Producer : Producer
{
    public override int Produce() => 42;
}

class Program
{
    static void Main(string[] args)
    {
         int number = new Int32Producer().Produce();
    }
}
```

Under Design1 no boxing or unboxing occurs.

**4. Increased Type Safety**

As seen in the above example, there can be cases where code which preiously could not be statically checked to be type safe, and required downcasting, can now be statically type checked.

#### Disadvantages

**1. Hides Complexity**

This makes it appear as if a method is an override of another method, when actually it is not, and we just make it *act* as if it is one using various tricks. As such it may be argued that we are hiding what is actually happening from the developer, which in some cases (as we shall see) may lead to unexpected results. Rather it may be better if give the developer more control over what happens instead, so he can generate similiar IL himself, but using C# code that makes it clearer what is happening internally. One way to do that may be to allow C# to override a method explicitly, similiarly to explicit interface implementations.

**2. Performance issues**

This design leads to an extra function call when calling the base method, which is dificult to inline unless the covariant override is marked as sealed. Whilst this would not make much of a difference to large functions, it could lead to performance issues for small functions.

**3. Extra method call appears in the call stack**

This design leads to an extra function call when calling the base method. This will lead to an extra function call in the call stack, which may be confusing for the developer.

**4. Reflection wont pick up this is an override of the base method**

Reflection wont indicate that the covariant override is an override of the base method. For example:

```csharp
class Program
{
    static void Main(string[] args)
    {
         var t = typeof(Dog);
	 var method = t.GetMethod("GiveBirth");
	 Console.WriteLine(IsOverride(method)); // returns false, not true as might be expected.
    }
    
    public static bool IsOverride(MethodInfo m) 
    {
        return m.GetBaseDefinition().ReflectedType != m.ReflectedType;
    }
}


public class Animal
{
    public virtual Animal GiveBirth() => new Animal();
}

public class Dog : Animal
{
    public override Dog GiveBirth() => new Dog(); //Should Compile
}
```

**5. Adding Attributes to the base method requires recompiling covariant override**
Consider the following code

```csharp
\\Assembly 1
public class Animal
{
    [SomeInheritedAttribute]
    public virtual Animal GiveBirth() => new Animal();
}

\\Assembly 2
public class Dog : Animal
{
    public override Dog GiveBirth() => new Dog(); //Should Compile
}
```

When Assembly 2 is compiled `SomeInheritedAttribute` is applied to `Dog.GiveBirth()` by the compiler.

Now lets say `AnotherInheritedAttribute` is added to Animal.GiveBirth(), or `SomeInheritedAttribute` is removed.

Until `Dog.GiveBirth()` is recompiled whilst referencing the new assembly, it wont be aware of these changes, and so its attributes will be incorrect. It also means that it will be impossible to use the same version of Assembly 2 with different versions of Assembly 1, and still have the attributes for Dog.GiveBirth() be correct.

In short, adding an Inherited attribute to a public virtual method can now potentially be a Binary (but not Source) breaking change.

#### Potential Long Term Solution to disadvantages 3, 4 and 5

It is important that when we compile to IL we don't lose the information that a method overrides another method. As such I think it is neccessary to create an attribute to store that information, which the compiler will automatically insert.

The Attribute could look something like this:

```csharp
[AttributeUsage(AttributeTargets.Method, AllowMultiple = false, Inherited = true)]
public class CovariantOverideAttribute : Attribute
{
    public CovariantOverideAttribute(Type baseType, string baseMethodName)
    {
        BaseMethod = baseType.GetMethod(baseMethodName);
    }

    public MethodInfo BaseMethod { get; }
}
```

Then it could be inserted into the IL as follows:

```csharp
.class public auto ansi beforefieldinit Animal
    extends [mscorlib]System.Object
{
    // Methods
    .method public hidebysig newslot virtual 
        instance class Animal GiveBirth () cil managed 
    {
        // Method begins at RVA 0x2050
        // Code size 6 (0x6)
        .maxstack 8

        IL_0000: newobj instance void Animal::.ctor()
        IL_0005: ret
    } // end of method Animal::GiveBirth

    .method public hidebysig specialname rtspecialname 
        instance void .ctor () cil managed 
    {
        // Method begins at RVA 0x2057
        // Code size 8 (0x8)
        .maxstack 8

        IL_0000: ldarg.0
        IL_0001: call instance void [mscorlib]System.Object::.ctor()
        IL_0006: nop
        IL_0007: ret
    } // end of method Animal::.ctor

} // end of class Animal

.class public auto ansi beforefieldinit Dog
    extends Animal
{
    // Methods
    .method private hidebysig virtual final
        instance class Animal Animal_GiveBirth () cil managed 
    {
        .override Animal::GiveBirth
        .maxstack  8
        .locals init (object V_0)

        IL_0000:  nop
        IL_0001:  ldarg.0
        IL_0002:  tail.
        IL_0004:  callvirt   instance class Dog Dog::GiveBirth()
        IL_0009:  ret
    } // end of method Dog::Animal_GiveBirth

    .method public hidebysig newslot virtual 
        instance class Dog GiveBirth() cil managed
    {
        .custom instance void CovariantOverideAttribute::.ctor(class [mscorlib]System.Type, string) = (
            01 00 06 41 6e 69 6d 61 6c 09 47 69 76 65 42 69
            72 74 68 00 00
        )
        // Method begins at RVA 0x2060
        // Code size 6 (0x6)
        .maxstack 8

        IL_0000: newobj instance void Dog::.ctor()
        IL_0005: ret
    }// end of method Dog::GiveBirth

    .method public hidebysig specialname rtspecialname 
        instance void .ctor () cil managed 
    {
        // Method begins at RVA 0x2067
        // Code size 8 (0x8)
        .maxstack 8

        IL_0000: ldarg.0
        IL_0001: call instance void Animal::.ctor()
        IL_0006: nop
        IL_0007: ret
    } // end of method Dog::.ctor

} // end of class Dog
```

This gives the oppurtunity for other parts of .Net framework to access this information in the future. This means that in the long run it may be possible to solve disadvantages 3,4, and 5.

The advantage of these solutions is that although they require changes to the runtime, standard library, tooling etc. this work is completely distinct to any work done for the core feature. As such, whether and when these updates are carried out does not preclude the introduction of Covariant Return Types into the C# language via changes in Roslyn. In fact, so long as the attribute is defined first, the work could even take place before any work is done on Roslyn - it is completely distinct work.

**Solving disadvantage 3**

It may be possible for the libraries used to produce the stack trace, or the tools that consume the stack trace, to use this attribute to skip the bridging method, and show a cleaned stack trace where the covariant override is called directly.

**Solving disadvantage 4**

It may be possible for System.Reflection to use this attribute to generate a MethodInfo that is aware that the covariant override method overrides a base method. Alternatively, an extra method or extension method could be added to MethodInfo, `GetCovariantOverrideBaseMethod` which will return the correct method.

**Solving disadvantage 5**
When loading an assembly,the runtime could run through the ILasm, adding or removing attributes as neccessary whenever it finds the `CovariantOverrideAttribute`. Alternatively, to avoid changes in the runtime, a tool could be created to do the same directly to a dll. This tool could then be run at any stage to make sure the Attributes are up to date, for example when running a Nuget Restore, or when creating an executable.

### 6.  Design2 (using an attribute to indicate the desired return type)

Under this design an override with a covariant return type is compiled to an IL method with the same return type as its base method. However an attribute is added to the method which indicates the actual return type of the object. Consuming code then explicitly casts the object returned from the method to the type indicated by the Attribute.

This would only work for conversions which can be safely reversed. However by definition identity and implicit reference conversions do not change the object being referenced, so this is not an issue.

#### Prototype Attribute

Here is an example of the attribute that could be used, and which we will be using in all the examples of emited IL for the test cases:

```csharp
[AttributeUsage(AttributeTargets.Method,AllowMultiple =false,Inherited =true)]
public class ReturnTypeAttribute : Attribute
{
    public ReturnTypeAttribute(Type returnType)
    {
        ReturnType = returnType;
    }

    public Type ReturnType { get; }
}
```

Roslyn will then detect this attribute when compiling code that calls a method with this attribute, and then cast the object returned from the method to the type contained in the attribute.

An obvious optimisation is to combine both casts into one when the returned object undergoes another implicit reference conversion, or even to remove the cast completely:

```csharp
[ReturnType(typeof(int)]
public object Method() => default(int);

...

int a = Method(); \\ converted to int a = (int)Method();

ValueType b = Method(); \\ converted to ValueType b = (ValueType)Method();

object c = Method(); \\ converted to object c = Method();

var d = Method(); \\ converted to int d = (int)Method();
```

#### Generated IL for all Test Cases

Note all IL has been tested using https://www.tutorialspoint.com/compile_ilasm_online.php

**case a**

```csharp
.assembly Covariant {}
.assembly extern mscorlib {}
.class private auto ansi beforefieldinit Program
    extends [mscorlib]System.Object
{
    // Methods
    .method private hidebysig static 
        void Main (
            string[] args
        ) cil managed 
    {
        // Method begins at RVA 0x2050
        // Code size 81 (0x51)
	.entrypoint
        .maxstack 1
        .locals init (
            [0] class Animal,
            [1] class Animal,
            [2] class Dog,
            [3] class Dog,
            [4] class Dog,
            [5] class Animal,
            [6] class Animal
        )

        IL_0000: nop
        IL_0001: newobj instance void Animal::.ctor()
        IL_0006: stloc.0
        IL_0007: ldloc.0
        IL_0008: callvirt instance class Animal Animal::GiveBirth()
        IL_000d: stloc.1
        IL_000e: ldloc.1
        IL_000f: callvirt instance class [mscorlib]System.Type [mscorlib]System.Object::GetType()
        IL_0014: pop
        IL_0015: newobj instance void Dog::.ctor()
        IL_001a: stloc.2
        IL_001b: ldloc.2
        IL_001c: callvirt instance class Animal Animal::GiveBirth()
        IL_0021: castclass Dog
        IL_0026: stloc.3
        IL_0027: ldloc.2
        IL_0028: callvirt instance class Animal Animal::GiveBirth()
        IL_002d: castclass Dog
        IL_0032: stloc.s 4
        IL_0034: ldloc.s 4
        IL_0036: callvirt instance class [mscorlib]System.Type [mscorlib]System.Object::GetType()
        IL_003b: pop
        IL_003c: ldloc.2
        IL_003d: stloc.s 5
        IL_003f: ldloc.s 5
        IL_0041: callvirt instance class Animal Animal::GiveBirth()
        IL_0046: stloc.s 6
        IL_0048: ldloc.s 6
        IL_004a: callvirt instance class [mscorlib]System.Type [mscorlib]System.Object::GetType()
        IL_004f: pop
        IL_0050: ret
    } // end of method Program::Main

    .method public hidebysig specialname rtspecialname 
        instance void .ctor () cil managed 
    {
        // Method begins at RVA 0x20ad
        // Code size 8 (0x8)
        .maxstack 8

        IL_0000: ldarg.0
        IL_0001: call instance void [mscorlib]System.Object::.ctor()
        IL_0006: nop
        IL_0007: ret
    } // end of method Program::.ctor

} // end of class Program

.class public auto ansi beforefieldinit Animal
    extends [mscorlib]System.Object
{
    // Methods
    .method public hidebysig newslot virtual 
        instance class Animal GiveBirth () cil managed 
    {
        // Method begins at RVA 0x20b6
        // Code size 6 (0x6)
        .maxstack 8

        IL_0000: newobj instance void Animal::.ctor()
        IL_0005: ret
    } // end of method Animal::GiveBirth

    .method public hidebysig specialname rtspecialname 
        instance void .ctor () cil managed 
    {
        // Method begins at RVA 0x20ad
        // Code size 8 (0x8)
        .maxstack 8

        IL_0000: ldarg.0
        IL_0001: call instance void [mscorlib]System.Object::.ctor()
        IL_0006: nop
        IL_0007: ret
    } // end of method Animal::.ctor

} // end of class Animal

.class public auto ansi beforefieldinit Dog
    extends Animal
{
    // Methods
    .method public hidebysig virtual 
        instance class Animal GiveBirth () cil managed 
    {
        .custom instance void ReturnTypeAttribute::.ctor(class [mscorlib]System.Type) = (
            01 00 03 44 6f 67 00 00
        )
        // Method begins at RVA 0x20bd
        // Code size 6 (0x6)
        .maxstack 8

        IL_0000: newobj instance void Dog::.ctor()
        IL_0005: ret
    } // end of method Dog::GiveBirth

    .method public hidebysig specialname rtspecialname 
        instance void .ctor () cil managed 
    {
        // Method begins at RVA 0x20c4
        // Code size 8 (0x8)
        .maxstack 8

        IL_0000: ldarg.0
        IL_0001: call instance void Animal::.ctor()
        IL_0006: nop
        IL_0007: ret
    } // end of method Dog::.ctor

} // end of class Dog

```

**case b**

```csharp
.assembly Covariant {}
.assembly extern mscorlib {}
.class private auto ansi beforefieldinit Program
    extends [mscorlib]System.Object
{
    // Methods
    .method private hidebysig static 
        void Main (
            string[] args
        ) cil managed 
    {
        // Method begins at RVA 0x2050
        // Code size 57 (0x39)
	.entrypoint
        .maxstack 1
        .locals init (
            [0] class Dog,
            [1] class Dog,
            [2] class Dog,
            [3] class Animal,
            [4] class Animal
        )

        IL_0000: nop
        IL_0001: newobj instance void Dog::.ctor()
        IL_0006: stloc.0
        IL_0007: ldloc.0
        IL_0008: callvirt instance class Animal Animal::GiveBirth()
        IL_000d: castclass Dog
        IL_0012: stloc.1
        IL_0013: ldloc.0
        IL_0014: callvirt instance class Animal Animal::GiveBirth()
        IL_0019: castclass Dog
        IL_001e: stloc.2
        IL_001f: ldloc.2
        IL_0020: callvirt instance class [mscorlib]System.Type [mscorlib]System.Object::GetType()
        IL_0025: pop
        IL_0026: ldloc.0
        IL_0027: stloc.3
        IL_0028: ldloc.3
        IL_0029: callvirt instance class Animal Animal::GiveBirth()
        IL_002e: stloc.s 4
        IL_0030: ldloc.s 4
        IL_0032: callvirt instance class [mscorlib]System.Type [mscorlib]System.Object::GetType()
        IL_0037: pop
        IL_0038: ret
    } // end of method Program::Main

    .method public hidebysig specialname rtspecialname 
        instance void .ctor () cil managed 
    {
        // Method begins at RVA 0x2095
        // Code size 8 (0x8)
        .maxstack 8

        IL_0000: ldarg.0
        IL_0001: call instance void [mscorlib]System.Object::.ctor()
        IL_0006: nop
        IL_0007: ret
    } // end of method Program::.ctor

} // end of class Program

.class public auto ansi abstract beforefieldinit Animal
    extends [mscorlib]System.Object
{
    // Methods
    .method public hidebysig newslot abstract virtual 
        instance class Animal GiveBirth () cil managed 
    {
    } // end of method Animal::GiveBirth

    .method family hidebysig specialname rtspecialname 
        instance void .ctor () cil managed 
    {
        // Method begins at RVA 0x2095
        // Code size 8 (0x8)
        .maxstack 8

        IL_0000: ldarg.0
        IL_0001: call instance void [mscorlib]System.Object::.ctor()
        IL_0006: nop
        IL_0007: ret
    } // end of method Animal::.ctor

} // end of class Animal

.class public auto ansi beforefieldinit Dog
    extends Animal
{
    // Methods
    .method public hidebysig virtual 
        instance class Animal GiveBirth () cil managed 
    {
        .custom instance void ReturnTypeAttribute::.ctor(class [mscorlib]System.Type) = (
            01 00 03 44 6f 67 00 00
        )
        // Method begins at RVA 0x209e
        // Code size 6 (0x6)
        .maxstack 8

        IL_0000: newobj instance void Dog::.ctor()
        IL_0005: ret
    } // end of method Dog::GiveBirth

    .method public hidebysig specialname rtspecialname 
        instance void .ctor () cil managed 
    {
        // Method begins at RVA 0x20a5
        // Code size 8 (0x8)
        .maxstack 8

        IL_0000: ldarg.0
        IL_0001: call instance void Animal::.ctor()
        IL_0006: nop
        IL_0007: ret
    } // end of method Dog::.ctor

} // end of class Dog
```

**case c**

```csharp
.assembly Covariant {}
.assembly extern mscorlib {}
.class private auto ansi beforefieldinit Program
    extends [mscorlib]System.Object
{
    // Methods
    .method private hidebysig static 
        void Main (
            string[] args
        ) cil managed 
    {
        // Method begins at RVA 0x2050
        // Code size 102 (0x66)
	.entrypoint
        .maxstack 1
        .locals init (
            [0] class Cat,
            [1] class Animal,
            [2] class Animal,
            [3] class Animal,
            [4] class Dog,
            [5] class Dog,
            [6] class Dog,
            [7] class Animal,
            [8] class Animal
        )

        IL_0000: nop
        IL_0001: newobj instance void Cat::.ctor()
        IL_0006: stloc.0
        IL_0007: ldloc.0
        IL_0008: callvirt instance class Animal Animal::GiveBirth()
        IL_000d: stloc.1
        IL_000e: ldloc.1
        IL_000f: callvirt instance class [mscorlib]System.Type [mscorlib]System.Object::GetType()
        IL_0014: pop
        IL_0015: ldloc.0
        IL_0016: stloc.2
        IL_0017: ldloc.2
        IL_0018: callvirt instance class Animal Animal::GiveBirth()
        IL_001d: stloc.3
        IL_001e: ldloc.3
        IL_001f: callvirt instance class [mscorlib]System.Type [mscorlib]System.Object::GetType()
        IL_0024: pop
        IL_0025: newobj instance void Dog::.ctor()
        IL_002a: stloc.s 4
        IL_002c: ldloc.s 4
        IL_002e: callvirt instance class Animal Animal::GiveBirth()
        IL_0033: castclass Dog
        IL_0038: stloc.s 5
        IL_003a: ldloc.s 4
        IL_003c: callvirt instance class Animal Animal::GiveBirth()
        IL_0041: castclass Dog
        IL_0046: stloc.s 6
        IL_0048: ldloc.s 6
        IL_004a: callvirt instance class [mscorlib]System.Type [mscorlib]System.Object::GetType()
        IL_004f: pop
        IL_0050: ldloc.s 4
        IL_0052: stloc.s 7
        IL_0054: ldloc.s 7
        IL_0056: callvirt instance class Animal Animal::GiveBirth()
        IL_005b: stloc.s 8
        IL_005d: ldloc.s 8
        IL_005f: callvirt instance class [mscorlib]System.Type [mscorlib]System.Object::GetType()
        IL_0064: pop
        IL_0065: ret
    } // end of method Program::Main

    .method public hidebysig specialname rtspecialname 
        instance void .ctor () cil managed 
    {
        // Method begins at RVA 0x20c2
        // Code size 8 (0x8)
        .maxstack 8

        IL_0000: ldarg.0
        IL_0001: call instance void [mscorlib]System.Object::.ctor()
        IL_0006: nop
        IL_0007: ret
    } // end of method Program::.ctor

} // end of class Program

.class public auto ansi abstract beforefieldinit Animal
    extends [mscorlib]System.Object
{
    // Methods
    .method public hidebysig newslot virtual 
        instance class Animal GiveBirth () cil managed 
    {
        // Method begins at RVA 0x20cb
        // Code size 6 (0x6)
        .maxstack 8

        IL_0000: newobj instance void Cat::.ctor()
        IL_0005: ret
    } // end of method Animal::GiveBirth

    .method family hidebysig specialname rtspecialname 
        instance void .ctor () cil managed 
    {
        // Method begins at RVA 0x20c2
        // Code size 8 (0x8)
        .maxstack 8

        IL_0000: ldarg.0
        IL_0001: call instance void [mscorlib]System.Object::.ctor()
        IL_0006: nop
        IL_0007: ret
    } // end of method Animal::.ctor

} // end of class Animal

.class public auto ansi beforefieldinit Dog
    extends Animal
{
    // Methods
    .method public hidebysig virtual 
        instance class Animal GiveBirth () cil managed 
    {
        .custom instance void ReturnTypeAttribute::.ctor(class [mscorlib]System.Type) = (
            01 00 03 44 6f 67 00 00
        )
        // Method begins at RVA 0x20d2
        // Code size 6 (0x6)
        .maxstack 8

        IL_0000: newobj instance void Dog::.ctor()
        IL_0005: ret
    } // end of method Dog::GiveBirth

    .method public hidebysig specialname rtspecialname 
        instance void .ctor () cil managed 
    {
        // Method begins at RVA 0x20d9
        // Code size 8 (0x8)
        .maxstack 8

        IL_0000: ldarg.0
        IL_0001: call instance void Animal::.ctor()
        IL_0006: nop
        IL_0007: ret
    } // end of method Dog::.ctor

} // end of class Dog

.class public auto ansi beforefieldinit Cat
    extends Animal
{
    // Methods
    .method public hidebysig specialname rtspecialname 
        instance void .ctor () cil managed 
    {
        // Method begins at RVA 0x20d9
        // Code size 8 (0x8)
        .maxstack 8

        IL_0000: ldarg.0
        IL_0001: call instance void Animal::.ctor()
        IL_0006: nop
        IL_0007: ret
    } // end of method Cat::.ctor

} // end of class Cat
```

**case d**

```csharp
.assembly Covariant {}
.assembly extern mscorlib {}
.class private auto ansi beforefieldinit Program
    extends [mscorlib]System.Object
{
    // Methods
    .method private hidebysig static 
        void Main (
            string[] args
        ) cil managed 
    {
        // Method begins at RVA 0x2050
        // Code size 111 (0x6f)
	.entrypoint
        .maxstack 1
        .locals init (
            [0] class Cat,
            [1] class Cat,
            [2] class Cat,
            [3] class IAnimal,
            [4] class IAnimal,
            [5] class Dog,
            [6] class Dog,
            [7] class Dog,
            [8] class IAnimal,
            [9] class IAnimal
        )

        IL_0000: nop
        IL_0001: newobj instance void Cat::.ctor()
        IL_0006: stloc.0
        IL_0007: ldloc.0
        IL_0008: callvirt instance class Cat Cat::GiveBirth()
        IL_000d: stloc.1
        IL_000e: ldloc.1
        IL_000f: callvirt instance class [mscorlib]System.Type [mscorlib]System.Object::GetType()
        IL_0014: pop
        IL_0015: ldloc.0
        IL_0016: callvirt instance class Cat Cat::GiveBirth()
        IL_001b: stloc.2
        IL_001c: ldloc.0
        IL_001d: stloc.3
        IL_001e: ldloc.3
        IL_001f: callvirt instance class IAnimal IAnimal::GiveBirth()
        IL_0024: stloc.s 4
        IL_0026: ldloc.s 4
        IL_0028: callvirt instance class [mscorlib]System.Type [mscorlib]System.Object::GetType()
        IL_002d: pop
        IL_002e: newobj instance void Dog::.ctor()
        IL_0033: stloc.s 5
        IL_0035: ldloc.s 5
        IL_0037: callvirt instance class IAnimal Dog::GiveBirth()
        IL_003c: castclass Dog
        IL_0041: stloc.s 6
        IL_0043: ldloc.s 5
        IL_0045: callvirt instance class IAnimal Dog::GiveBirth()
        IL_004a: castclass Dog
        IL_004f: stloc.s 7
        IL_0051: ldloc.s 7
        IL_0053: callvirt instance class [mscorlib]System.Type [mscorlib]System.Object::GetType()
        IL_0058: pop
        IL_0059: ldloc.s 5
        IL_005b: stloc.s 8
        IL_005d: ldloc.s 8
        IL_005f: callvirt instance class IAnimal IAnimal::GiveBirth()
        IL_0064: stloc.s 9
        IL_0066: ldloc.s 9
        IL_0068: callvirt instance class [mscorlib]System.Type [mscorlib]System.Object::GetType()
        IL_006d: pop
        IL_006e: ret
    } // end of method Program::Main

    .method public hidebysig specialname rtspecialname 
        instance void .ctor () cil managed 
    {
        // Method begins at RVA 0x20cb
        // Code size 8 (0x8)
        .maxstack 8

        IL_0000: ldarg.0
        IL_0001: call instance void [mscorlib]System.Object::.ctor()
        IL_0006: nop
        IL_0007: ret
    } // end of method Program::.ctor

} // end of class Program

.class interface public auto ansi abstract IAnimal
{
    // Methods
    .method public hidebysig newslot abstract virtual 
        instance class IAnimal GiveBirth () cil managed 
    {
    } // end of method IAnimal::GiveBirth

} // end of class IAnimal

.class public auto ansi beforefieldinit Dog
    extends [mscorlib]System.Object
    implements IAnimal
{
    // Methods
    .method public final hidebysig newslot virtual 
        instance class IAnimal GiveBirth () cil managed 
    {
        .custom instance void ReturnTypeAttribute::.ctor(class [mscorlib]System.Type) = (
            01 00 03 44 6f 67 00 00
        )
        // Method begins at RVA 0x20d4
        // Code size 6 (0x6)
        .maxstack 8

        IL_0000: newobj instance void Dog::.ctor()
        IL_0005: ret
    } // end of method Dog::GiveBirth

    .method public hidebysig specialname rtspecialname 
        instance void .ctor () cil managed 
    {
        // Method begins at RVA 0x20cb
        // Code size 8 (0x8)
        .maxstack 8

        IL_0000: ldarg.0
        IL_0001: call instance void [mscorlib]System.Object::.ctor()
        IL_0006: nop
        IL_0007: ret
    } // end of method Dog::.ctor

} // end of class Dog

.class public auto ansi beforefieldinit Cat
    extends [mscorlib]System.Object
    implements IAnimal
{
    // Methods
    .method private final hidebysig newslot virtual 
        instance class IAnimal IAnimal.GiveBirth () cil managed 
    {
        .override method instance class IAnimal IAnimal::GiveBirth()
        // Method begins at RVA 0x20d4
        // Code size 6 (0x6)
        .maxstack 8

        IL_0000: newobj instance void Dog::.ctor()
        IL_0005: ret
    } // end of method Cat::IAnimal.GiveBirth

    .method public hidebysig 
        instance class Cat GiveBirth () cil managed 
    {
        // Method begins at RVA 0x20db
        // Code size 6 (0x6)
        .maxstack 8

        IL_0000: newobj instance void Cat::.ctor()
        IL_0005: ret
    } // end of method Cat::GiveBirth

    .method public hidebysig specialname rtspecialname 
        instance void .ctor () cil managed 
    {
        // Method begins at RVA 0x20cb
        // Code size 8 (0x8)
        .maxstack 8

        IL_0000: ldarg.0
        IL_0001: call instance void [mscorlib]System.Object::.ctor()
        IL_0006: nop
        IL_0007: ret
    } // end of method Cat::.ctor

} // end of class Cat

```

**case e**
```csharp
.assembly Covariant {}
.assembly extern mscorlib {}
.class private auto ansi beforefieldinit Program
    extends [mscorlib]System.Object
{
    // Methods
    .method private hidebysig static 
        void Main (
            string[] args
        ) cil managed 
    {
        // Method begins at RVA 0x2050
        // Code size 61 (0x3d)
        .entrypoint
        .maxstack 1
        .locals init (
            [0] class Retriever,
            [1] class Retriever,
            [2] class Dog,
            [3] class Dog,
            [4] class Animal,
            [5] class Animal
        )

        IL_0000: nop
        IL_0001: newobj instance void Retriever::.ctor()
        IL_0006: stloc.0
        IL_0007: ldloc.0
        IL_0008: callvirt instance class Animal Animal::GiveBirth()
        IL_000d: castclass Retriever
        IL_0012: stloc.1
        IL_0013: ldloc.0
        IL_0014: stloc.2
        IL_0015: ldloc.2
        IL_0016: callvirt instance class Animal Animal::GiveBirth()
        IL_001b: castclass Dog
        IL_0020: stloc.3
        IL_0021: ldloc.3
        IL_0022: callvirt instance class [mscorlib]System.Type [mscorlib]System.Object::GetType()
        IL_0027: pop
        IL_0028: ldloc.0
        IL_0029: stloc.s 4
        IL_002b: ldloc.s 4
        IL_002d: callvirt instance class Animal Animal::GiveBirth()
        IL_0032: stloc.s 5
        IL_0034: ldloc.s 5
        IL_0036: callvirt instance class [mscorlib]System.Type [mscorlib]System.Object::GetType()
        IL_003b: pop
        IL_003c: ret
    } // end of method Program::Main

    .method public hidebysig specialname rtspecialname 
        instance void .ctor () cil managed 
    {
        // Method begins at RVA 0x2099
        // Code size 8 (0x8)
        .maxstack 8

        IL_0000: ldarg.0
        IL_0001: call instance void [mscorlib]System.Object::.ctor()
        IL_0006: nop
        IL_0007: ret
    } // end of method Program::.ctor

} // end of class Program

.class public auto ansi beforefieldinit Animal
    extends [mscorlib]System.Object
{
    // Methods
    .method public hidebysig newslot virtual 
        instance class Animal GiveBirth () cil managed 
    {
        // Method begins at RVA 0x20a2
        // Code size 6 (0x6)
        .maxstack 8

        IL_0000: newobj instance void Animal::.ctor()
        IL_0005: ret
    } // end of method Animal::GiveBirth

    .method public hidebysig specialname rtspecialname 
        instance void .ctor () cil managed 
    {
        // Method begins at RVA 0x2099
        // Code size 8 (0x8)
        .maxstack 8

        IL_0000: ldarg.0
        IL_0001: call instance void [mscorlib]System.Object::.ctor()
        IL_0006: nop
        IL_0007: ret
    } // end of method Animal::.ctor

} // end of class Animal

.class public auto ansi beforefieldinit Dog
    extends Animal
{
    // Methods
    .method public hidebysig virtual 
        instance class Animal GiveBirth () cil managed 
    {
        .custom instance void ReturnTypeAttribute::.ctor(class [mscorlib]System.Type) = (
            01 00 03 44 6f 67 00 00
        )
        // Method begins at RVA 0x20a9
        // Code size 6 (0x6)
        .maxstack 8

        IL_0000: newobj instance void Dog::.ctor()
        IL_0005: ret
    } // end of method Dog::GiveBirth

    .method public hidebysig specialname rtspecialname 
        instance void .ctor () cil managed 
    {
        // Method begins at RVA 0x20b0
        // Code size 8 (0x8)
        .maxstack 8

        IL_0000: ldarg.0
        IL_0001: call instance void Animal::.ctor()
        IL_0006: nop
        IL_0007: ret
    } // end of method Dog::.ctor

} // end of class Dog

.class public auto ansi beforefieldinit Poodle
    extends Dog
{
    // Methods
    .method public hidebysig virtual 
        instance class Animal GiveBirth () cil managed 
    {
        .custom instance void ReturnTypeAttribute::.ctor(class [mscorlib]System.Type) = (
            01 00 03 44 6f 67 00 00
        )
        // Method begins at RVA 0x20b9
        // Code size 6 (0x6)
        .maxstack 8

        IL_0000: newobj instance void Poodle::.ctor()
        IL_0005: ret
    } // end of method Poodle::GiveBirth

    .method public hidebysig specialname rtspecialname 
        instance void .ctor () cil managed 
    {
        // Method begins at RVA 0x20c0
        // Code size 8 (0x8)
        .maxstack 8

        IL_0000: ldarg.0
        IL_0001: call instance void Dog::.ctor()
        IL_0006: nop
        IL_0007: ret
    } // end of method Poodle::.ctor

} // end of class Poodle

.class public auto ansi beforefieldinit Retriever
    extends Dog
{
    // Methods
    .method public hidebysig virtual 
        instance class Animal GiveBirth () cil managed 
    {
        .custom instance void ReturnTypeAttribute::.ctor(class [mscorlib]System.Type) = (
            01 00 09 52 65 74 72 69 65 76 65 72 00 00
        )
        // Method begins at RVA 0x20c9
        // Code size 6 (0x6)
        .maxstack 8

        IL_0000: newobj instance void Retriever::.ctor()
        IL_0005: ret
    } // end of method Retriever::GiveBirth

    .method public hidebysig specialname rtspecialname 
        instance void .ctor () cil managed 
    {
        // Method begins at RVA 0x20c0
        // Code size 8 (0x8)
        .maxstack 8

        IL_0000: ldarg.0
        IL_0001: call instance void Dog::.ctor()
        IL_0006: nop
        IL_0007: ret
    } // end of method Retriever::.ctor

} // end of class Retriever
```

**case f**
```csharp
.assembly Covariant {}
.assembly extern mscorlib {}
.class private auto ansi beforefieldinit Program
    extends [mscorlib]System.Object
{
    // Methods
    .method private hidebysig static 
        void Main (
            string[] args
        ) cil managed 
    {
        // Method begins at RVA 0x2050
        // Code size 61 (0x3d)
	.entrypoint
        .maxstack 1
        .locals init (
            [0] class Retriever,
            [1] class Retriever,
            [2] class Dog,
            [3] class Dog,
            [4] class Animal,
            [5] class Animal
        )

        IL_0000: nop
        IL_0001: newobj instance void Retriever::.ctor()
        IL_0006: stloc.0
        IL_0007: ldloc.0
        IL_0008: callvirt instance class Animal Animal::GiveBirth()
        IL_000d: castclass Retriever
        IL_0012: stloc.1
        IL_0013: ldloc.0
        IL_0014: stloc.2
        IL_0015: ldloc.2
        IL_0016: callvirt instance class Animal Animal::GiveBirth()
        IL_001b: castclass Dog
        IL_0020: stloc.3
        IL_0021: ldloc.3
        IL_0022: callvirt instance class [mscorlib]System.Type [mscorlib]System.Object::GetType()
        IL_0027: pop
        IL_0028: ldloc.0
        IL_0029: stloc.s 4
        IL_002b: ldloc.s 4
        IL_002d: callvirt instance class Animal Animal::GiveBirth()
        IL_0032: stloc.s 5
        IL_0034: ldloc.s 5
        IL_0036: callvirt instance class [mscorlib]System.Type [mscorlib]System.Object::GetType()
        IL_003b: pop
        IL_003c: ret
    } // end of method Program::Main

    .method public hidebysig specialname rtspecialname 
        instance void .ctor () cil managed 
    {
        // Method begins at RVA 0x2099
        // Code size 8 (0x8)
        .maxstack 8

        IL_0000: ldarg.0
        IL_0001: call instance void [mscorlib]System.Object::.ctor()
        IL_0006: nop
        IL_0007: ret
    } // end of method Program::.ctor

} // end of class Program

.class public auto ansi abstract beforefieldinit Animal
    extends [mscorlib]System.Object
{
    // Methods
    .method public hidebysig newslot abstract virtual 
        instance class Animal GiveBirth () cil managed 
    {
    } // end of method Animal::GiveBirth

    .method family hidebysig specialname rtspecialname 
        instance void .ctor () cil managed 
    {
        // Method begins at RVA 0x2099
        // Code size 8 (0x8)
        .maxstack 8

        IL_0000: ldarg.0
        IL_0001: call instance void [mscorlib]System.Object::.ctor()
        IL_0006: nop
        IL_0007: ret
    } // end of method Animal::.ctor

} // end of class Animal

.class public auto ansi abstract beforefieldinit Dog
    extends Animal
{
    // Methods
    .method public hidebysig abstract virtual 
        instance class Animal GiveBirth () cil managed 
    {
        .custom instance void ReturnTypeAttribute::.ctor(class [mscorlib]System.Type) = (
            01 00 03 44 6f 67 00 00
        )
    } // end of method Dog::GiveBirth

    .method family hidebysig specialname rtspecialname 
        instance void .ctor () cil managed 
    {
        // Method begins at RVA 0x20a2
        // Code size 8 (0x8)
        .maxstack 8

        IL_0000: ldarg.0
        IL_0001: call instance void Animal::.ctor()
        IL_0006: nop
        IL_0007: ret
    } // end of method Dog::.ctor

} // end of class Dog

.class public auto ansi beforefieldinit Poodle
    extends Dog
{
    // Methods
    .method public hidebysig virtual 
        instance class Animal GiveBirth () cil managed 
    {
        .custom instance void ReturnTypeAttribute::.ctor(class [mscorlib]System.Type) = (
            01 00 03 44 6f 67 00 00
        )
        // Method begins at RVA 0x20ab
        // Code size 6 (0x6)
        .maxstack 8

        IL_0000: newobj instance void Poodle::.ctor()
        IL_0005: ret
    } // end of method Poodle::GiveBirth

    .method public hidebysig specialname rtspecialname 
        instance void .ctor () cil managed 
    {
        // Method begins at RVA 0x20b2
        // Code size 8 (0x8)
        .maxstack 8

        IL_0000: ldarg.0
        IL_0001: call instance void Dog::.ctor()
        IL_0006: nop
        IL_0007: ret
    } // end of method Poodle::.ctor

} // end of class Poodle

.class public auto ansi beforefieldinit Retriever
    extends Dog
{
    // Methods
    .method public hidebysig virtual 
        instance class Animal GiveBirth () cil managed 
    {
        .custom instance void ReturnTypeAttribute::.ctor(class [mscorlib]System.Type) = (
            01 00 09 52 65 74 72 69 65 76 65 72 00 00
        )
        // Method begins at RVA 0x20bb
        // Code size 6 (0x6)
        .maxstack 8

        IL_0000: newobj instance void Retriever::.ctor()
        IL_0005: ret
    } // end of method Retriever::GiveBirth

    .method public hidebysig specialname rtspecialname 
        instance void .ctor () cil managed 
    {
        // Method begins at RVA 0x20b2
        // Code size 8 (0x8)
        .maxstack 8

        IL_0000: ldarg.0
        IL_0001: call instance void Dog::.ctor()
        IL_0006: nop
        IL_0007: ret
    } // end of method Retriever::.ctor

} // end of class Retriever
```

**case g**
```csharp
.assembly Hello {}
.assembly extern mscorlib {}
.class private auto ansi beforefieldinit Program
    extends [mscorlib]System.Object
{
    // Methods
    .method private hidebysig static 
        void Main (
            string[] args
        ) cil managed 
    {
        // Method begins at RVA 0x2050
        // Code size 36 (0x24)
	.entrypoint
        .maxstack 1
        .locals init (
            [0] class Dog,
            [1] class Dog,
            [2] class Animal,
            [3] class Animal
        )

        IL_0000: nop
        IL_0001: newobj instance void Dog::.ctor()
        IL_0006: stloc.0
        IL_0007: ldloc.0
        IL_0008: callvirt instance class Animal Animal::GiveBirth()
        IL_000d: castclass Dog
        IL_0012: stloc.1
        IL_0013: ldloc.0
        IL_0014: stloc.2
        IL_0015: ldloc.2
        IL_0016: callvirt instance class Animal Animal::GiveBirth()
        IL_001b: stloc.3
        IL_001c: ldloc.3
        IL_001d: callvirt instance class [mscorlib]System.Type [mscorlib]System.Object::GetType()
        IL_0022: pop
        IL_0023: ret
    } // end of method Program::Main

    .method public hidebysig specialname rtspecialname 
        instance void .ctor () cil managed 
    {
        // Method begins at RVA 0x2080
        // Code size 8 (0x8)
        .maxstack 8

        IL_0000: ldarg.0
        IL_0001: call instance void [mscorlib]System.Object::.ctor()
        IL_0006: nop
        IL_0007: ret
    } // end of method Program::.ctor

} // end of class Program

.class public auto ansi beforefieldinit Animal
    extends [mscorlib]System.Object
{
    // Methods
    .method public hidebysig newslot virtual 
        instance class Animal GiveBirth () cil managed 
    {
        // Method begins at RVA 0x2089
        // Code size 6 (0x6)
        .maxstack 8

        IL_0000: newobj instance void Animal::.ctor()
        IL_0005: ret
    } // end of method Animal::GiveBirth

    .method public hidebysig specialname rtspecialname 
        instance void .ctor () cil managed 
    {
        // Method begins at RVA 0x2080
        // Code size 8 (0x8)
        .maxstack 8

        IL_0000: ldarg.0
        IL_0001: call instance void [mscorlib]System.Object::.ctor()
        IL_0006: nop
        IL_0007: ret
    } // end of method Animal::.ctor

} // end of class Animal

.class public auto ansi beforefieldinit Dog
    extends Animal
{
    // Methods
    .method public final hidebysig virtual 
        instance class Animal GiveBirth () cil managed 
    {
        .custom instance void ReturnTypeAttribute::.ctor(class [mscorlib]System.Type) = (
            01 00 03 44 6f 67 00 00
        )
        // Method begins at RVA 0x2090
        // Code size 6 (0x6)
        .maxstack 8

        IL_0000: newobj instance void Dog::.ctor()
        IL_0005: ret
    } // end of method Dog::GiveBirth

    .method public hidebysig specialname rtspecialname 
        instance void .ctor () cil managed 
    {
        // Method begins at RVA 0x2097
        // Code size 8 (0x8)
        .maxstack 8

        IL_0000: ldarg.0
        IL_0001: call instance void Animal::.ctor()
        IL_0006: nop
        IL_0007: ret
    } // end of method Dog::.ctor

} // end of class Dog

.class public auto ansi beforefieldinit Cat
    extends Animal
{
    // Methods
    .method public final hidebysig virtual 
        instance class Animal GiveBirth () cil managed 
    {
        // Method begins at RVA 0x20a0
        // Code size 6 (0x6)
        .maxstack 8

        IL_0000: newobj instance void Cat::.ctor()
        IL_0005: ret
    } // end of method Cat::GiveBirth

    .method public hidebysig specialname rtspecialname 
        instance void .ctor () cil managed 
    {
        // Method begins at RVA 0x2097
        // Code size 8 (0x8)
        .maxstack 8

        IL_0000: ldarg.0
        IL_0001: call instance void Animal::.ctor()
        IL_0006: nop
        IL_0007: ret
    } // end of method Cat::.ctor

} // end of class Cat
```

**case h**
```csharp
.assembly Covariant {}
.assembly extern mscorlib {}
.class public auto ansi beforefieldinit InheritedAtrributeSingleInstance
    extends [mscorlib]System.Attribute
{
    .custom instance void [mscorlib]System.AttributeUsageAttribute::.ctor(valuetype [mscorlib]System.AttributeTargets) = (
        01 00 40 00 00 00 02 00 54 02 0d 41 6c 6c 6f 77
        4d 75 6c 74 69 70 6c 65 00 54 02 09 49 6e 68 65
        72 69 74 65 64 01
    )
    // Methods
    .method public hidebysig specialname rtspecialname 
        instance void .ctor (
            int32 id
        ) cil managed 
    {
        // Method begins at RVA 0x2050
        // Code size 9 (0x9)
        .maxstack 8

        IL_0000: ldarg.0
        IL_0001: call instance void [mscorlib]System.Attribute::.ctor()
        IL_0006: nop
        IL_0007: nop
        IL_0008: ret
    } // end of method InheritedAtrributeSingleInstance::.ctor

} // end of class InheritedAtrributeSingleInstance

.class public auto ansi beforefieldinit InheritedAtrributeMultipleInstance
    extends [mscorlib]System.Attribute
{
    .custom instance void [mscorlib]System.AttributeUsageAttribute::.ctor(valuetype [mscorlib]System.AttributeTargets) = (
        01 00 40 00 00 00 02 00 54 02 0d 41 6c 6c 6f 77
        4d 75 6c 74 69 70 6c 65 01 54 02 09 49 6e 68 65
        72 69 74 65 64 01
    )
    // Methods
    .method public hidebysig specialname rtspecialname 
        instance void .ctor (
            int32 id
        ) cil managed 
    {
        // Method begins at RVA 0x2050
        // Code size 9 (0x9)
        .maxstack 8

        IL_0000: ldarg.0
        IL_0001: call instance void [mscorlib]System.Attribute::.ctor()
        IL_0006: nop
        IL_0007: nop
        IL_0008: ret
    } // end of method InheritedAtrributeMultipleInstance::.ctor

} // end of class InheritedAtrributeMultipleInstance

.class public auto ansi beforefieldinit NonInheritedAtrributeSingleInstance
    extends [mscorlib]System.Attribute
{
    .custom instance void [mscorlib]System.AttributeUsageAttribute::.ctor(valuetype [mscorlib]System.AttributeTargets) = (
        01 00 40 00 00 00 02 00 54 02 0d 41 6c 6c 6f 77
        4d 75 6c 74 69 70 6c 65 00 54 02 09 49 6e 68 65
        72 69 74 65 64 00
    )
    // Methods
    .method public hidebysig specialname rtspecialname 
        instance void .ctor (
            int32 id
        ) cil managed 
    {
        // Method begins at RVA 0x2050
        // Code size 9 (0x9)
        .maxstack 8

        IL_0000: ldarg.0
        IL_0001: call instance void [mscorlib]System.Attribute::.ctor()
        IL_0006: nop
        IL_0007: nop
        IL_0008: ret
    } // end of method NonInheritedAtrributeSingleInstance::.ctor

} // end of class NonInheritedAtrributeSingleInstance

.class public auto ansi beforefieldinit Animal
    extends [mscorlib]System.Object
{
    // Methods
    .method public hidebysig newslot virtual 
        instance class Animal GiveBirth () cil managed 
    {
        .custom instance void InheritedAtrributeSingleInstance::.ctor(int32) = (
            01 00 00 00 00 00 00 00
        )
        .custom instance void InheritedAtrributeMultipleInstance::.ctor(int32) = (
            01 00 00 00 00 00 00 00
        )
        .custom instance void NonInheritedAtrributeSingleInstance::.ctor(int32) = (
            01 00 00 00 00 00 00 00
        )
        // Method begins at RVA 0x205a
        // Code size 6 (0x6)
        .maxstack 8

        IL_0000: newobj instance void Animal::.ctor()
        IL_0005: ret
    } // end of method Animal::GiveBirth

    .method public hidebysig specialname rtspecialname 
        instance void .ctor () cil managed 
    {
        // Method begins at RVA 0x2061
        // Code size 8 (0x8)
        .maxstack 8

        IL_0000: ldarg.0
        IL_0001: call instance void [mscorlib]System.Object::.ctor()
        IL_0006: nop
        IL_0007: ret
    } // end of method Animal::.ctor

} // end of class Animal

.class public auto ansi beforefieldinit Dog
    extends Animal
{
    // Methods
    .method public hidebysig virtual 
        instance class Animal GiveBirth () cil managed 
    {
        .custom instance void ReturnTypeAttribute::.ctor(class [mscorlib]System.Type) = (
            01 00 03 44 6f 67 00 00
        )
        // Method begins at RVA 0x206a
        // Code size 6 (0x6)
        .maxstack 8

        IL_0000: newobj instance void Dog::.ctor()
        IL_0005: ret
    } // end of method Dog::GiveBirth

    .method public hidebysig specialname rtspecialname 
        instance void .ctor () cil managed 
    {
        // Method begins at RVA 0x2071
        // Code size 8 (0x8)
        .maxstack 8

        IL_0000: ldarg.0
        IL_0001: call instance void Animal::.ctor()
        IL_0006: nop
        IL_0007: ret
    } // end of method Dog::.ctor

} // end of class Dog

.class public auto ansi beforefieldinit Poodle
    extends Dog
{
    // Methods
    .method public hidebysig virtual 
        instance class Animal GiveBirth () cil managed 
    {
        .custom instance void ReturnTypeAttribute::.ctor(class [mscorlib]System.Type) = (
            01 00 03 44 6f 67 00 00
        )
        // Method begins at RVA 0x207a
        // Code size 6 (0x6)
        .maxstack 8

        IL_0000: newobj instance void Poodle::.ctor()
        IL_0005: ret
    } // end of method Poodle::GiveBirth

    .method public hidebysig specialname rtspecialname 
        instance void .ctor () cil managed 
    {
        // Method begins at RVA 0x2081
        // Code size 8 (0x8)
        .maxstack 8

        IL_0000: ldarg.0
        IL_0001: call instance void Dog::.ctor()
        IL_0006: nop
        IL_0007: ret
    } // end of method Poodle::.ctor

} // end of class Poodle

.class public auto ansi beforefieldinit Retriever
    extends Dog
{
    // Methods
    .method public hidebysig virtual 
        instance class Animal GiveBirth () cil managed 
    {
        .custom instance void ReturnTypeAttribute::.ctor(class [mscorlib]System.Type) = (
            01 00 09 52 65 74 72 69 65 76 65 72 00 00
        )
        // Method begins at RVA 0x208a
        // Code size 6 (0x6)
        .maxstack 8

        IL_0000: newobj instance void Retriever::.ctor()
        IL_0005: ret
    } // end of method Retriever::GiveBirth

    .method public hidebysig specialname rtspecialname 
        instance void .ctor () cil managed 
    {
        // Method begins at RVA 0x2081
        // Code size 8 (0x8)
        .maxstack 8

        IL_0000: ldarg.0
        IL_0001: call instance void Dog::.ctor()
        IL_0006: nop
        IL_0007: ret
    } // end of method Retriever::.ctor

} // end of class Retriever

.class public auto ansi beforefieldinit StBernard
    extends Dog
{
    // Methods
    .method public hidebysig virtual 
        instance class Animal GiveBirth () cil managed 
    {
        .custom instance void InheritedAtrributeMultipleInstance::.ctor(int32) = (
            01 00 01 00 00 00 00 00
        )
        .custom instance void InheritedAtrributeSingleInstance::.ctor(int32) = (
            01 00 01 00 00 00 00 00
        )
        .custom instance void ReturnTypeAttribute::.ctor(class [mscorlib]System.Type) = (
            01 00 09 53 74 42 65 72 6e 61 72 64 00 00
        )
        // Method begins at RVA 0x2091
        // Code size 6 (0x6)
        .maxstack 8

        IL_0000: newobj instance void StBernard::.ctor()
        IL_0005: ret
    } // end of method StBernard::GiveBirth

    .method public hidebysig specialname rtspecialname 
        instance void .ctor () cil managed 
    {
        // Method begins at RVA 0x2081
        // Code size 8 (0x8)
        .maxstack 8

        IL_0000: ldarg.0
        IL_0001: call instance void Dog::.ctor()
        IL_0006: nop
        IL_0007: ret
    } // end of method StBernard::.ctor

} // end of class StBernard

.class public auto ansi beforefieldinit Collie
    extends Dog
{
    // Methods
    .method public hidebysig virtual 
        instance class Animal GiveBirth () cil managed 
    {
        .custom instance void InheritedAtrributeMultipleInstance::.ctor(int32) = (
            01 00 01 00 00 00 00 00
        )
        .custom instance void InheritedAtrributeSingleInstance::.ctor(int32) = (
            01 00 01 00 00 00 00 00
        )
        .custom instance void ReturnTypeAttribute::.ctor(class [mscorlib]System.Type) = (
            01 00 03 44 6f 67 00 00
        )
        // Method begins at RVA 0x2098
        // Code size 6 (0x6)
        .maxstack 8

        IL_0000: newobj instance void Collie::.ctor()
        IL_0005: ret
    } // end of method Collie::GiveBirth

    .method public hidebysig specialname rtspecialname 
        instance void .ctor () cil managed 
    {
        // Method begins at RVA 0x2081
        // Code size 8 (0x8)
        .maxstack 8

        IL_0000: ldarg.0
        IL_0001: call instance void Dog::.ctor()
        IL_0006: nop
        IL_0007: ret
    } // end of method Collie::.ctor

} // end of class Collie

.class public auto ansi beforefieldinit Cat
    extends Animal
{
    // Methods
    .method public hidebysig virtual 
        instance class Animal GiveBirth () cil managed 
    {
        .custom instance void InheritedAtrributeMultipleInstance::.ctor(int32) = (
            01 00 01 00 00 00 00 00
        )
        .custom instance void InheritedAtrributeSingleInstance::.ctor(int32) = (
            01 00 01 00 00 00 00 00
        )
        .custom instance void NonInheritedAtrributeSingleInstance::.ctor(int32) = (
            01 00 01 00 00 00 00 00
        )
        .custom instance void ReturnTypeAttribute::.ctor(class [mscorlib]System.Type) = (
            01 00 03 43 61 74 00 00
        )
        // Method begins at RVA 0x209f
        // Code size 6 (0x6)
        .maxstack 8

        IL_0000: newobj instance void Cat::.ctor()
        IL_0005: ret
    } // end of method Cat::GiveBirth

    .method public hidebysig specialname rtspecialname 
        instance void .ctor () cil managed 
    {
        // Method begins at RVA 0x2071
        // Code size 8 (0x8)
        .maxstack 8

        IL_0000: ldarg.0
        IL_0001: call instance void Animal::.ctor()
        IL_0006: nop
        IL_0007: ret
    } // end of method Cat::.ctor

} // end of class Cat

.class public auto ansi beforefieldinit Tiger
    extends Cat
{
    // Methods
    .method public hidebysig virtual 
        instance class Animal GiveBirth () cil managed 
    {
        .custom instance void ReturnTypeAttribute::.ctor(class [mscorlib]System.Type) = (
            01 00 05 54 69 67 65 72 00 00
        )
        // Method begins at RVA 0x20a6
        // Code size 6 (0x6)
        .maxstack 8

        IL_0000: newobj instance void Tiger::.ctor()
        IL_0005: ret
    } // end of method Tiger::GiveBirth

    .method public hidebysig specialname rtspecialname 
        instance void .ctor () cil managed 
    {
        // Method begins at RVA 0x20ad
        // Code size 8 (0x8)
        .maxstack 8

        IL_0000: ldarg.0
        IL_0001: call instance void Cat::.ctor()
        IL_0006: nop
        IL_0007: ret
    } // end of method Tiger::.ctor

} // end of class Tiger

.class public auto ansi beforefieldinit Leopard
    extends Cat
{
    // Methods
    .method public hidebysig virtual 
        instance class Animal GiveBirth () cil managed 
    {
        .custom instance void ReturnTypeAttribute::.ctor(class [mscorlib]System.Type) = (
            01 00 03 43 61 74 00 00
        )
        // Method begins at RVA 0x20b6
        // Code size 6 (0x6)
        .maxstack 8

        IL_0000: newobj instance void Leopard::.ctor()
        IL_0005: ret
    } // end of method Leopard::GiveBirth

    .method public hidebysig specialname rtspecialname 
        instance void .ctor () cil managed 
    {
        // Method begins at RVA 0x20ad
        // Code size 8 (0x8)
        .maxstack 8

        IL_0000: ldarg.0
        IL_0001: call instance void Cat::.ctor()
        IL_0006: nop
        IL_0007: ret
    } // end of method Leopard::.ctor

} // end of class Leopard

.class public auto ansi beforefieldinit Cheetah
    extends Cat
{
    // Methods
    .method public hidebysig virtual 
        instance class Animal GiveBirth () cil managed 
    {
        .custom instance void InheritedAtrributeMultipleInstance::.ctor(int32) = (
            01 00 02 00 00 00 00 00
        )
        .custom instance void InheritedAtrributeSingleInstance::.ctor(int32) = (
            01 00 02 00 00 00 00 00
        )
        .custom instance void ReturnTypeAttribute::.ctor(class [mscorlib]System.Type) = (
            01 00 07 43 68 65 65 74 61 68 00 00
        )
        // Method begins at RVA 0x20bd
        // Code size 6 (0x6)
        .maxstack 8

        IL_0000: newobj instance void Cheetah::.ctor()
        IL_0005: ret
    } // end of method Cheetah::GiveBirth

    .method public hidebysig specialname rtspecialname 
        instance void .ctor () cil managed 
    {
        // Method begins at RVA 0x20ad
        // Code size 8 (0x8)
        .maxstack 8

        IL_0000: ldarg.0
        IL_0001: call instance void Cat::.ctor()
        IL_0006: nop
        IL_0007: ret
    } // end of method Cheetah::.ctor

} // end of class Cheetah

.class public auto ansi beforefieldinit Jaguar
    extends Cat
{
    // Methods
    .method public hidebysig virtual 
        instance class Animal GiveBirth () cil managed 
    {
        .custom instance void InheritedAtrributeMultipleInstance::.ctor(int32) = (
            01 00 02 00 00 00 00 00
        )
        .custom instance void InheritedAtrributeSingleInstance::.ctor(int32) = (
            01 00 02 00 00 00 00 00
        )
        .custom instance void ReturnTypeAttribute::.ctor(class [mscorlib]System.Type) = (
            01 00 03 43 61 74 00 00
        )
        // Method begins at RVA 0x20c4
        // Code size 6 (0x6)
        .maxstack 8

        IL_0000: newobj instance void Jaguar::.ctor()
        IL_0005: ret
    } // end of method Jaguar::GiveBirth

    .method public hidebysig specialname rtspecialname 
        instance void .ctor () cil managed 
    {
        // Method begins at RVA 0x20ad
        // Code size 8 (0x8)
        .maxstack 8

        IL_0000: ldarg.0
        IL_0001: call instance void Cat::.ctor()
        IL_0006: nop
        IL_0007: ret
    } // end of method Jaguar::.ctor

} // end of class Jaguar
```

**case i**

This case is essentially translated to the equavalent C#:
```csharp
class Program
{
    static void Main(string[] args)
    {
        var dog = new Dog();
        Func<Dog> dogFunc = () => (Dog)dog.GiveBirth(); //should compile
        var babyDog = FunctionApplier(() => (Dog)dog.GiveBirth()); //type of var should be Dog
    }

    static T FunctionApplier<T>(Func<T> func) => func();
}

public class Animal
{
    public virtual Animal GiveBirth() => new Animal();
}

public class Dog : Animal
{
    [ReturnType(typeof(Dog))]
    public override Animal GiveBirth() => new Dog();
}
```
Note that a new delegate has to be created wrapping the GiveBirth method to make sure that the delegate is of the right type.

In theory this ahould also be optimised, so that when the delegate return type is of the base return type, no extra delegate is ontroduced. For example;

```csharp
[ReturnType(typeof(int)]
public object Method() => default(int);

...

Func<int> a = Method; \\ converted to Func<int> a = () => (int)Method();

Func<ValueType> b = Method; \\ converted to Func<ValueType> b = () => (ValueType)Method();

Func<object> c = Method; \\ converted to Func<object> c = Method;
```

Here is the emitted IL for this test case:

```csharp
.assembly Covariant {}
.assembly extern mscorlib {}
.class private auto ansi beforefieldinit Program
    extends [mscorlib]System.Object
{
    // Nested Types
    .class nested private auto ansi sealed beforefieldinit '<>c__DisplayClass0_0'
        extends [mscorlib]System.Object
    {
        .custom instance void [mscorlib]System.Runtime.CompilerServices.CompilerGeneratedAttribute::.ctor() = (
            01 00 00 00
        )
        // Fields
        .field public class Dog dog

        // Methods
        .method public hidebysig specialname rtspecialname 
            instance void .ctor () cil managed 
        {
            // Method begins at RVA 0x2096
            // Code size 8 (0x8)
            .maxstack 8

            IL_0000: ldarg.0
            IL_0001: call instance void [mscorlib]System.Object::.ctor()
            IL_0006: nop
            IL_0007: ret
        } // end of method '<>c__DisplayClass0_0'::.ctor

        .method assembly hidebysig 
            instance class Dog '<Main>b__0' () cil managed 
        {
            // Method begins at RVA 0x20cf
            // Code size 17 (0x11)
            .maxstack 8

            IL_0000: ldarg.0
            IL_0001: ldfld class Dog Program/'<>c__DisplayClass0_0'::dog
            IL_0006: callvirt instance class Animal Animal::GiveBirth()
            IL_000b: castclass Dog
            IL_0010: ret
        } // end of method '<>c__DisplayClass0_0'::'<Main>b__0'

        .method assembly hidebysig 
            instance class Dog '<Main>b__1' () cil managed 
        {
            // Method begins at RVA 0x20cf
            // Code size 17 (0x11)
            .maxstack 8

            IL_0000: ldarg.0
            IL_0001: ldfld class Dog Program/'<>c__DisplayClass0_0'::dog
            IL_0006: callvirt instance class Animal Animal::GiveBirth()
            IL_000b: castclass Dog
            IL_0010: ret
        } // end of method '<>c__DisplayClass0_0'::'<Main>b__1'

    } // end of class <>c__DisplayClass0_0


    // Methods
    .method private hidebysig static 
        void Main (
            string[] args
        ) cil managed 
    {
        // Method begins at RVA 0x2050
        // Code size 50 (0x32)
	.entrypoint
        .maxstack 2
        .locals init (
            [0] class Program/'<>c__DisplayClass0_0',
            [1] class [mscorlib]System.Func`1<class Dog>,
            [2] class Dog
        )

        // sequence point: hidden
        IL_0000: newobj instance void Program/'<>c__DisplayClass0_0'::.ctor()
        IL_0005: stloc.0
        IL_0006: nop
        IL_0007: ldloc.0
        IL_0008: newobj instance void Dog::.ctor()
        IL_000d: stfld class Dog Program/'<>c__DisplayClass0_0'::dog
        IL_0012: ldloc.0
        IL_0013: ldftn instance class Dog Program/'<>c__DisplayClass0_0'::'<Main>b__0'()
        IL_0019: newobj instance void class [mscorlib]System.Func`1<class Dog>::.ctor(object, native int)
        IL_001e: stloc.1
        IL_001f: ldloc.0
        IL_0020: ldftn instance class Dog Program/'<>c__DisplayClass0_0'::'<Main>b__1'()
        IL_0026: newobj instance void class [mscorlib]System.Func`1<class Dog>::.ctor(object, native int)
        IL_002b: call !!0 Program::FunctionApplier<class Dog>(class [mscorlib]System.Func`1<!!0>)
        IL_0030: stloc.2
        IL_0031: ret
    } // end of method Program::Main

    .method private hidebysig static 
        !!T FunctionApplier<T> (
            class [mscorlib]System.Func`1<!!T> func
        ) cil managed 
    {
        // Method begins at RVA 0x208e
        // Code size 7 (0x7)
        .maxstack 8

        IL_0000: ldarg.0
        IL_0001: callvirt instance !0 class [mscorlib]System.Func`1<!!T>::Invoke()
        IL_0006: ret
    } // end of method Program::FunctionApplier

    .method public hidebysig specialname rtspecialname 
        instance void .ctor () cil managed 
    {
        // Method begins at RVA 0x2096
        // Code size 8 (0x8)
        .maxstack 8

        IL_0000: ldarg.0
        IL_0001: call instance void [mscorlib]System.Object::.ctor()
        IL_0006: nop
        IL_0007: ret
    } // end of method Program::.ctor

} // end of class Program

.class public auto ansi beforefieldinit Animal
    extends [mscorlib]System.Object
{
    // Methods
    .method public hidebysig newslot virtual 
        instance class Animal GiveBirth () cil managed 
    {
        // Method begins at RVA 0x209f
        // Code size 6 (0x6)
        .maxstack 8

        IL_0000: newobj instance void Animal::.ctor()
        IL_0005: ret
    } // end of method Animal::GiveBirth

    .method public hidebysig specialname rtspecialname 
        instance void .ctor () cil managed 
    {
        // Method begins at RVA 0x2096
        // Code size 8 (0x8)
        .maxstack 8

        IL_0000: ldarg.0
        IL_0001: call instance void [mscorlib]System.Object::.ctor()
        IL_0006: nop
        IL_0007: ret
    } // end of method Animal::.ctor

} // end of class Animal

.class public auto ansi beforefieldinit Dog
    extends Animal
{
    // Methods
    .method public hidebysig virtual 
        instance class Animal GiveBirth () cil managed 
    {
        .custom instance void ReturnTypeAttribute::.ctor(class [mscorlib]System.Type) = (
            01 00 03 44 6f 67 00 00
        )
        // Method begins at RVA 0x20a6
        // Code size 6 (0x6)
        .maxstack 8

        IL_0000: newobj instance void Dog::.ctor()
        IL_0005: ret
    } // end of method Dog::GiveBirth

    .method public hidebysig specialname rtspecialname 
        instance void .ctor () cil managed 
    {
        // Method begins at RVA 0x20ad
        // Code size 8 (0x8)
        .maxstack 8

        IL_0000: ldarg.0
        IL_0001: call instance void Animal::.ctor()
        IL_0006: nop
        IL_0007: ret
    } // end of method Dog::.ctor

} // end of class Dog
```

**case j**

At the moment attributes cannot use Type parameters. This makes the following code impossible to write in C#:

```csharp
public abstract class Factory<T>
{
    public abstract T Create();
}

public abstract class DerivedFactory<TDerived, TBase> : Factory<TBase> where TDerived : TBase
{
    [ReturnType(typeof(TDerived))]
    public abstract override TBase Create(); //Should Compile
}
```

However, in CIL it is theoretically possible for attributes to use type parameters. However I believe there may be issues when using it in practice - see https://github.com/dotnet/csharplang/blob/master/meetings/2017/LDM-2017-02-21.md#generic-attributes. I don't know whether this would effect us here, since this attribute is only designed to be used at compile time. However given that this is only a proposal I will not investigate this further for now.

Either way, there are many other ways to indicate to the compiler the return type to cast to, so we can deal with this case as and when we come to it.

The second example in this test case is:

```csharp
public abstract class Factory<T>
{
    public abstract T Create();
}

public class Animal
{
}

public class Dog : Animal
{
}

public class DogFactory : Factory<Animal>
{
    public override Dog Create() => new Dog(); //should compile
}
```

This works absolutely fine, generating the following IL:

```csharp
.assembly Covariant {}
.assembly extern mscorlib {}
.class public auto ansi abstract beforefieldinit Factory`1<T>
    extends [mscorlib]System.Object
{
    // Methods
    .method public hidebysig newslot abstract virtual 
        instance !T Create () cil managed 
    {
    } // end of method Factory`1::Create

    .method family hidebysig specialname rtspecialname 
        instance void .ctor () cil managed 
    {
        // Method begins at RVA 0x2050
        // Code size 8 (0x8)
        .maxstack 8

        IL_0000: ldarg.0
        IL_0001: call instance void [mscorlib]System.Object::.ctor()
        IL_0006: nop
        IL_0007: ret
    } // end of method Factory`1::.ctor

} // end of class Factory`1

.class public auto ansi beforefieldinit Animal
    extends [mscorlib]System.Object
{
    // Methods
    .method public hidebysig specialname rtspecialname 
        instance void .ctor () cil managed 
    {
        // Method begins at RVA 0x2050
        // Code size 8 (0x8)
        .maxstack 8

        IL_0000: ldarg.0
        IL_0001: call instance void [mscorlib]System.Object::.ctor()
        IL_0006: nop
        IL_0007: ret
    } // end of method Animal::.ctor

} // end of class Animal

.class public auto ansi beforefieldinit Dog
    extends Animal
{
    // Methods
    .method public hidebysig specialname rtspecialname 
        instance void .ctor () cil managed 
    {
        // Method begins at RVA 0x2059
        // Code size 8 (0x8)
        .maxstack 8

        IL_0000: ldarg.0
        IL_0001: call instance void Animal::.ctor()
        IL_0006: nop
        IL_0007: ret
    } // end of method Dog::.ctor

} // end of class Dog

.class public auto ansi beforefieldinit DogFactory
    extends class Factory`1<class Animal>
{
    // Methods
    .method public hidebysig virtual 
        instance class Animal Create () cil managed 
    {
        .custom instance void ReturnTypeAttribute::.ctor(class [mscorlib]System.Type) = (
            01 00 03 44 6f 67 00 00
        )
        // Method begins at RVA 0x2062
        // Code size 6 (0x6)
        .maxstack 8

        IL_0000: newobj instance void Dog::.ctor()
        IL_0005: ret
    } // end of method DogFactory::Create

    .method public hidebysig specialname rtspecialname 
        instance void .ctor () cil managed 
    {
        // Method begins at RVA 0x2069
        // Code size 8 (0x8)
        .maxstack 8

        IL_0000: ldarg.0
        IL_0001: call instance void class Factory`1<class Animal>::.ctor()
        IL_0006: nop
        IL_0007: ret
    } // end of method DogFactory::.ctor

} // end of class DogFactory
```

### 7.  How Design2 plays with other .Net code

Since this Design uses type erasure, and the types are put back in using the attribute, any code written in other .Net languages which do not support the attribute, or in older versions of C# will be oblivious to the covariant return type. Instead they will see the original return type.

For example, let us assume that Design2 is implemented in C# X.0

The following library is then written:

```csharp
\\C# X.0
public class Animal
{
    public virtual Animal GiveBirth() => new Animal();
}

public class Dog : Animal
{
    public override Dog GiveBirth() => new Dog();
}
```

Then the library is consumed from some C# 7.0. The consuming code would see the following public API:
```csharp
\\ API as percieved by C# 7.0
public class Animal
{
    public virtual Animal GiveBirth();
}

public class Dog : Animal
{
    public override Animal GiveBirth();
}
```

Thus any casts whivh are automattically inserted in C# x.0, will have to be manually inserted in C# 7.0.

This in and of itself is not a problem. It does however makes this design less attractive than Design1, where covariant return types would be able to be consumed identically from C# 7.0 and other .Net languages as from C# X.0.

However this is not the only problem with backwards compatability. Unfortunately, by mixing C# X.0 and C# 7.0 it is actually possible to cause runtime type errors which should in theory be prevented. For example:

```csharp
\\Assembly1: C# X.0
public class Animal
{
    public virtual Animal GiveBirth() => new Animal();
}

public class Dog : Animal
{
    public override Dog GiveBirth() => new Dog();
}

public static class BabyGetter
{
    public static Dog GetBaby(Dog dog) => dog.GiveBirth();
}

...
\\Assembly2: C# 7.0

public class Retriever : Dog
{
     public override Animal GiveBirth() => new Animal(); \\This compiles in C# 7.0, even though in C# X.0 it would be disallowed.
}

class Program
{
    static void Main(string[] args)
    {
         var retriever = new Retriever();
	 var dog = BabyGetter.GetBaby(retriever); //Throws InvalidCastException
    }
}
```

There are two issues with the code above. Firstly, perfectly legal code containing no explicit casts is throwing an InvalidCastException. Given that one of the cornerstones of C# is type safety, this is bad news.

Secondly, perfectly legal C# 7.0 code is no longer legal in C# X.0. We do not like to make breaking changes between different language versions.

As a result backwards compatability, and compatability with other .Net code is a serious issue for Design2.

### 8. Design2 Advantages/Disadvantages

#### Advantages

**1. No extra method call required**

Unlike Design1, No extra method call is required when a covariant return type is used. This increases performance, and avoids altering the stack trace.

**2. Easy to generate the code for the covariant return type**

The code generated for the covariant return type is almost identical to the code generated for a non covariant return type, but with an attribute added. This should be easy to implement.

**3. The method is a genuine override of the parent method**

This means that attributes and reflection work exactly as one would expect.

#### Disadvantages

**1. Uses Type Erasure**

One lesson that can be learned from Java Generics, is that use of type erasure can lead to enourmous complexity and numerous issues. All the disadvantages listed here stem from the use of type erasure.

**2. Not Backwards Compatible**

See the above section "How Design2 plays with other .Net code".

**3. Hides Boxing**

Consider the following code:

```csharp
public class Producer
{
    public virtual object Produce() => new object();
}

public class Int32Producer : Producer
{
    public override int Produce() => 42;
}

class Program
{
    static void Main(string[] args)
    {
         int number = new Int32Producer().Produce();
    }
}
```

Whilst it appears as though the int 42 is not being boxed in this example, the actual generated IL looks something like this:

```csharp
public class Producer
{
    public virtual object Produce() => new object();
}

public class Int32Producer : Producer
{
    [ReturnType(int)]
    public override object Produce() => 42;
}

class Program
{
    static void Main(string[] args)
    {
         int number = (int) new Int32Producer().Produce();
    }
}
```

In fact 42 is being both unboxed and boxed here, but that fact is invisible to anyone who doesn't  know the internals of this Design.

This may lead to unexpected performance issues.

**4. This requires changing emited IL wherever a covariant return type is consumed**

Wherever a method with the ReturnTypeAttribute is consumed, casts will have to be added to the IL.

This makes this a far larger change to Roslyn than Design1, which only required local changes. Instead this Design would presumably require changing Roslyn in numerous places to make sure the cast is inserted. This is unlikely to happen for a design which already has many weaknesses.

**5. Added complexity when it comes to creating delegates**

See the emited IL for test case J. In order for a delegate to have the desired return type, instead of creating a delegate directly from the Method, it is neccessary to create a lambda which calls the method and then casts the returned value.

Not only does this add complexity to the compiler, but it also will cause performance issues by adding an extra function call.

### 9. Design3 (explicit virtual method overrides)

In my opinion, both Design1 and Design2 suffer from an issue which Anders Hejlsberg called Simplexity. They attempt to do something inherently complex, and then provide a simple abstraction over the behaviour which hides the complexity. Unfortunately neither abstraction is perfect, and this can lead to unexpected behaviour - whether missing attributes in Design1, or hidden boxing in Design2.

The aim in this design is not to provide an abstraction which simulates Covariant return types, but to provide a mechanism by which programmers can simulate it themselves.

As we have already seen, it is possible to simulate Covariant return types when implementing an interface using Explicit Interface Implementations. 

For example:

```csharp
public interface IAnimal
{
    IAnimal GiveBirth();
}

public class Dog : IAnimal
{
    IAnimal IAnimal.GiveBirth() => GiveBirth();
    
    public Dog GiveBirth() => new Dog();
}
```

Here when calling `IAnimal.GiveBirth()` an `IAnimal` is returned, but when calling `Dog.GiveBirth()` a `Dog` is returned. However they both end up calling the same Method. Thus the public API 'looks' exactly the same as it would if the CLR supported true Covariant Overrides.

This is the same technique as Design1 uses. However since Design1 does this all under the hood, it can lead to unexpected behaviour such as missing attributes, and unexpected function calls. Here however it is perfectly clear to both the writer and the consumer what is going on, and they can work with that.

The proposal for this design is based off https://github.com/dotnet/csharplang/issues/1618, and aims to add the ability to explicitly overide virtual methods:

```csharp
public class Animal
{
    public virtual Animal GiveBirth() => new Animal();
}

public class Dog : Animal
{
    Animal base.GiveBirth() => GiveBirth();
    
    public virtual Dog GiveBirth() => new Dog();
}
```

This syntax is identical to that of Explicit Interface Implementation, except that the base keyword is used instead.

The issue with this syntax is that it may be in some cases be ambigous which virtual method is being overriden. For example:

```csharp
public class Animal
{
    public virtual Animal GiveBirth() => new Animal();
}

public class Dog : Animal
{
    public new virtual Animal GiveBirth() => new Dog();
}

public class Poodle : Dog
{
    Animal base.GiveBirth() => new Poodle();
}
```

Their are a number of alternatives here.

One is to say that the method being overriden is the same method as would be overriden if an implicit override had been used - i.e.

```csharp
public class Poodle : Dog
{
    public override Animal GiveBirth() => new Poodle(); //overrides Dog.GiveBirth
}
```

This is perfectly consistent with the language, but is less flexible. It also doesn't give the ability to mitigate performance issues when their are multiple levels of covariant overrides, as was done in the IL for test cases e and f in Design 1.

The second alternative is to use identical syntax to explicit interface implementation. i.e.

```csharp
public class Animal
{
    public virtual Animal GiveBirth() => new Animal();
}

public class Dog : Animal
{
    public new virtual Animal GiveBirth() => new Dog();
}

public class Poodle : Dog
{
    Animal Dog.GiveBirth() => new Poodle();
    Animal Animal.GiveBirth() => new Poodle();
}
```

This is again perfectly consistent with language, and it can be noticed that this is not an explicit interface implementation due to the convention that interface names start with 'I'.

The third is to use the suggested syntax for default interface methods of `base(type)`:

```csharp
public class Animal
{
    public virtual Animal GiveBirth() => new Animal();
}

public class Dog : Animal
{
    public new virtual Animal GiveBirth() => new Dog();
}

public class Poodle : Dog
{
    Animal base(Dog).GiveBirth() => new Poodle();
    Animal base(Animal).GiveBirth() => new Poodle();
}
```

I feel that this overkill though. My preffered syntax is the second, and I will be going with it for the remainder of this proposal.

In terms of what this gets compiled down to, the emmited method is a private final method named `BaseType.Method` which uses the `.override` keyword to specify which Method it is overriding.

Using this new feature it is possible to generate the IL given for all the test cases in Design1. In order not to repeat myself, I will just give the first example.

```csharp
class Program
{
    static void Main(string[] args)
    {
        Animal animal = new Animal();
        var babyAnimal = animal.GiveBirth(); //type of var should be Animal
        babyAnimal.GetType(); // should be Animal
        Dog dog = new Dog();
        Dog babyDog = dog.GiveBirth(); // should compile and run
        var babyDog2 = dog.GiveBirth(); // type of var should be Dog
        babyDog2.GetType(); // should be Dog
        Animal animal2 = dog;
        var babyAnimal2 = animal2.GiveBirth(); //type of var should be Animal
        babyAnimal2.GetType(); // should be Dog
    }
}

public class Animal
{
    public virtual Animal GiveBirth() => new Animal();
}

public class Dog : Animal
{
    Animal Animal.GiveBirth() => GiveBirth();
    public virtual Dog GiveBirth() => new Dog();
}
```

Emits the IL

```csharp
.assembly Covariant {}
.assembly extern mscorlib {}
.class private auto ansi beforefieldinit Program
    extends [mscorlib]System.Object
{
    // Methods
    .method private hidebysig static 
        void Main (
            string[] args
        ) cil managed 
    {
        // Method begins at RVA 0x2050
        // Code size 71 (0x47)
        .entrypoint
        .maxstack 1
        .locals init (
            [0] class Animal,
            [1] class Animal,
            [2] class Dog,
            [3] class Dog,
            [4] class Dog,
            [5] class Animal,
            [6] class Animal
        )

        IL_0000: nop
        IL_0001: newobj instance void Animal::.ctor()
        IL_0006: stloc.0
        IL_0007: ldloc.0
        IL_0008: callvirt instance class Animal Animal::GiveBirth()
        IL_000d: stloc.1
        IL_000e: ldloc.1
        IL_000f: callvirt instance class [mscorlib]System.Type [mscorlib]System.Object::GetType()
        IL_0014: pop
        IL_0015: newobj instance void Dog::.ctor()
        IL_001a: stloc.2
        IL_001b: ldloc.2
        IL_001c: callvirt instance class Dog Dog::GiveBirth()
        IL_0021: stloc.3
        IL_0022: ldloc.2
        IL_0023: callvirt instance class Dog Dog::GiveBirth()
        IL_0028: stloc.s 4
        IL_002a: ldloc.s 4
        IL_002c: callvirt instance class [mscorlib]System.Type [mscorlib]System.Object::GetType()
        IL_0031: pop
        IL_0032: ldloc.2
        IL_0033: stloc.s 5
        IL_0035: ldloc.s 5
        IL_0037: callvirt instance class Animal Animal::GiveBirth()
        IL_003c: stloc.s 6
        IL_003e: ldloc.s 6
        IL_0040: callvirt instance class [mscorlib]System.Type [mscorlib]System.Object::GetType()
        IL_0045: pop
        IL_0046: ret
    } // end of method Program::Main

    .method public hidebysig specialname rtspecialname 
        instance void .ctor () cil managed 
    {
        // Method begins at RVA 0x20a3
        // Code size 8 (0x8)
        .maxstack 8

        IL_0000: ldarg.0
        IL_0001: call instance void [mscorlib]System.Object::.ctor()
        IL_0006: nop
        IL_0007: ret
    } // end of method Program::.ctor

} // end of class Program


.class public auto ansi beforefieldinit Animal
    extends [mscorlib]System.Object
{
    // Methods
    .method public hidebysig newslot virtual 
        instance class Animal GiveBirth () cil managed 
    {
        // Method begins at RVA 0x2050
        // Code size 6 (0x6)
        .maxstack 8

        IL_0000: newobj instance void Animal::.ctor()
        IL_0005: ret
    } // end of method Animal::GiveBirth

    .method public hidebysig specialname rtspecialname 
        instance void .ctor () cil managed 
    {
        // Method begins at RVA 0x2057
        // Code size 8 (0x8)
        .maxstack 8

        IL_0000: ldarg.0
        IL_0001: call instance void [mscorlib]System.Object::.ctor()
        IL_0006: nop
        IL_0007: ret
    } // end of method Animal::.ctor

} // end of class Animal

.class public auto ansi beforefieldinit Dog
    extends Animal
{
    // Methods
    .method private final hidebysig virtual 
        instance class Animal Animal.GiveBirth () cil managed 
    {
    
        .override Animal::GiveBirth
        // Method begins at RVA 0x2058
        // Code size 7 (0x7)
        .maxstack  8
        .locals init (object V_0)

        IL_0000:  nop
        IL_0001:  ldarg.0
        IL_0002:  tail.
        IL_0004:  callvirt   instance class Dog Dog::GiveBirth()
        IL_0009:  ret
    } // end of method Dog::Animal.GiveBirth

    .method public hidebysig newslot virtual 
        instance class Dog GiveBirth() cil managed
    {
        // Method begins at RVA 0x2060
        // Code size 6 (0x6)
        .maxstack 8

        IL_0000: newobj instance void Dog::.ctor()
        IL_0005: ret
    }// end of method Dog::GiveBirth

    .method public hidebysig specialname rtspecialname 
        instance void .ctor () cil managed 
    {
        // Method begins at RVA 0x2067
        // Code size 8 (0x8)
        .maxstack 8

        IL_0000: ldarg.0
        IL_0001: call instance void Animal::.ctor()
        IL_0006: nop
        IL_0007: ret
    } // end of method Dog::.ctor

} // end of class Dog
```

#### Precise Specification

Given a class 'TBase', and a class 'TDerived' which is derived from TBase, and TBase defines a virtual or abstract method 'SomeMethod', and there is no class 'TInBetween' such that TDerived is derived from TInBetween, and TInBetween is derived from TBase, and TInBetween provides a non-explicit override of SomeMethod which marks SomeMethod as sealed .

Then TDerived may define an ExplicitMethodOverride

```
ExplicitMethodOverride :
	ReturnType BaseType '.' MethodName '(' TypeArguments* ')' MethodBody
```

Where ReturnType must be the same as the Return Type of SomeMethod, and BaseType must be 'TBase', and MethodName must be 'SomeMethod'. The rules for the Type Arguments and the MethodBody are the same as for any other override of a virtual/abstract method.

Then a method will be generated in IL wich is marked as private final, and has the name 'BaseType.MethodName', and uses the '.override' syntax to override TBase::SomeMethod, and whose body corresponds to that which should be generated for MethodBody.

An ExplicitMethodOverride cannnot be marked as private, public, internal, protected, abstract, virtual or sealed. The grammer given above is the only valid grammar.

Furthermore, if an explicit method override is provided, no warning should occur if another method is provided which hides SomeMethod and does not use the new keyword. 

It will be a compile time error to provide two explicit overrides of the same method, or an explicit and implicit override of the same method, but it is not an error to provide an explicit override of a method, and an explicit or implicit override of another method which overrides that first method.

If TDerived provides an explicit override of 'TBase::SomeMethod', and TDerived has no method with at least the same visibility as 'TBase::SomeMethod', the same name as SomeMethod and the same type arguments as SomeMethod, the method `TBase::SomeMethod` will not be hidden on an instance of `TDerived` - eg. it is legal C# to write `new TDerived().SomeMethod()`. There are ways to make this code illegal in future C# versions, but no way to make it illegal in old C# versions. In order to preserve backwards compatibility, it thus seems that we will have to make this legal in future versions as well.

As such I suggest it will be a compile time error for TDerived to provide an explicit override of 'TBase::SomeMethod', unless TDerived has a method with at least the same visibility as 'TBase::SomeMethod', the same name as SomeMethod and the same type arguments as SomeMethod. I will be going with this for the remainder of this proposal. However this point could be debated.

#### TestCases

**test case a**
```csharp
public class A
{
	public virtual void M() => Console.WriteLine("A::M");
}

/// Should not compile => no method with same signature as A::M()
//public class B : A
//{
//    void A.M() => Console.WriteLine("B::A.M");
//}

/// Should not compile => no method with same signature as A::M() and equal or greater visibility
//public class C : A
//{
//    void A.M() => Console.WriteLine("C::A.M");
//    private void M()=> Console.WriteLine("C::M");
//}

/// Should not compile => no method with same signature as A::M() and equal or greater visibility
//public class C : A
//{
//    void A.M() => Console.WriteLine("C::A.M");
//    protected void M()=> Console.WriteLine("C::M");
//}

/// Should not compile => no method with same signature as A::M() and equal or greater visibility
//public class D : A
//{
//    void A.M() => Console.WriteLine("D::A.M");
//    internal void M()=> Console.WriteLine("D::M");
//}

/// Should not compile => no method with same signature as A::M() and equal or greater visibility
//public class E : A
//{
//    void A.M() => Console.WriteLine("E::A.M");
//    protected internal void M()=> Console.WriteLine("E::M");
//}

/// Should not compile => F::A.M() does not have the same return type as A::M()
//public class F : A
//{
//	  int A.M()
//	  {
//		  Console.WriteLine("F::M");
//		  return 0;
//	  }
//	  public void M() => Console.WriteLine("F::M");
//}

/// Should not compile => no method with same signature as A::M()
//public class G : A
//{
//    void A.M() => Console.WriteLine("G::A.M");
//    public void M(int x)=> Console.WriteLine("G::M");
//}

/// Should not compile => multiple overrides of A::M()
//public class H : A
//{
//    void A.M() => Console.WriteLine("H::A.M");
//    public override void M()=> Console.WriteLine("H::M");
//}

/// Should not compile => multiple overrides of A::M()
//public class I : A
//{
//    void A.M() => Console.WriteLine("I::A.M");
//    void A.M() => Console.WriteLine("I::A.M");
//}

/// Should not compile => J::A.M() does not have same type parameters as A::M() 
//public class J : A
//{
//    void A.M(int x) => Console.WriteLine("J::A.M");
//    public void M() => Console.WriteLine("J::M");
//}

/// Should not compile => There is no method A::N()
//public class K : A
//{
//    void A.N() => Console.WriteLine("K::A.N");
//    public void N() => Console.WriteLine("K::N");
//}

/// Should Compile, without warning that L::M() hides A::M() 
public class L : A
{
	void A.M() => Console.WriteLine("L::A.M");

	public void M() => Console.WriteLine("L::M");
}

/// Should Compile, without warning that N::M() hides A::M()
public class N : A
{
	void A.M() => Console.WriteLine("N::A.M");

	public int M()
	{
		Console.WriteLine("N::M");
		return 0;
	}
}

/// Should Compile
public class O : A
{
	void A.M() => Console.WriteLine("O::A.M");

	public new void M() => Console.WriteLine("O::M");
}

/// Should Compile
public class P : A
{
	void A.M() => Console.WriteLine("P::A.M");

	public new int M()
	{
		Console.WriteLine("P::M");
		return 0;
	}
}
```

**test case b**

```csharp
public class A
{
	internal virtual void M() => Console.WriteLine("A::M");
}

/// Should not compile => no method with same signature as A::M() and equal or greater visibility
//public class C : A
//{
//    void A.M() => Console.WriteLine("C::A.M");
//    private void M()=> Console.WriteLine("C::M");
//}

/// Should not compile => no method with same signature as A::M() and equal or greater visibility
//public class C : A
//{
//    void A.M() => Console.WriteLine("C::A.M");
//    protected void M()=> Console.WriteLine("C::M");
//}

/// Should Compile
public class D : A
{
	void A.M() => Console.WriteLine("D::A.M");
	internal new void M() => Console.WriteLine("D::M");
}

/// Should Compile
public class E : A
{
	void A.M() => Console.WriteLine("E::A.M");
	protected internal new void M() => Console.WriteLine("E::M");
}

/// Should Compile
public class F : A
{
	void A.M() => Console.WriteLine("F::A.M");

	public new void M() => Console.WriteLine("F::M");
}
```
