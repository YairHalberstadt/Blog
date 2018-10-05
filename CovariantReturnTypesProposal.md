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
TODO: Complete
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

//TODO: carry on from here

.class public auto ansi beforefieldinit Leopard
    extends Cat
{
    // Methods
    .method public hidebysig virtual 
        instance class Cat GiveBirth2 () cil managed 
    {
        // Method begins at RVA 0x20de
        // Code size 6 (0x6)
        .maxstack 8

        IL_0000: newobj instance void Leopard::.ctor()
        IL_0005: ret
    } // end of method Leopard::GiveBirth2

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
    .method public final hidebysig virtual 
        instance class Cat GiveBirth2 () cil managed 
    {
        .custom instance void InheritedAtrributeMultipleInstance::.ctor(int32) = (
            01 00 02 00 00 00 00 00
        )
        .custom instance void InheritedAtrributeSingleInstance::.ctor(int32) = (
            01 00 02 00 00 00 00 00
        )
        // Method begins at RVA 0x20e5
        // Code size 7 (0x7)
        .maxstack 8

        IL_0000: ldarg.0
        IL_0001: callvirt instance class Cheetah Cheetah::GiveBirth3()
        IL_0006: ret
    } // end of method Cheetah::GiveBirth2

    .method public hidebysig newslot virtual 
        instance class Cheetah GiveBirth3 () cil managed 
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
    } // end of method Cheetah::GiveBirth3

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
        instance class Cat GiveBirth2 () cil managed 
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
    } // end of method Jaguar::GiveBirth2

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
```
