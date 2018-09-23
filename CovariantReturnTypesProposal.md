## Design Proposals For Adding Covariant Return Types in C#

### Contents

1. Description
2. Test Cases
3. Design1 (creating a new method as well as a bridging overriding method)
4. How Design1 Plays with other .Net code
5. Design1 Advantages/Disadvantages
6. Design2 (using an attribute to indicate the desired return type)
7. How Design2 Plays with other .Net code
8. Design2 Advantages/Disadvantages
9. Personal Conclusions

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
 

An implicit reference conversion covers all the inheritance-related conversions. It seems OK to me, even if it may be advisable to further restrict this rule by explicitly listing the conversions we want to allow and support. For example we may choose to restrict this to only allow identity conversions. Both designs given in this proposal would work in either case.

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

public class Poodle : Dog
{
    //public override Dog GiveBirth() => new Poodle(); // Should not compile
}

public class Retriever : Dog
{
    //public override Retriever GiveBirth() => new Retriever(); // Should not Compile
}

public class Cat : Animal
{
    public sealed override Animal GiveBirth() => new Cat();
}

public class Tiger : Cat
{
    //public override Tiger GiveBirth() => new Tiger(); // Should not compile
}
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
    public sealed override Cat GiveBirth() => new Cat();
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
    .method private hidebysig virtual final
        instance class Animal Animal_GiveBirth () cil managed 
    {
        .override Animal::GiveBirth
        // Method begins at RVA 0x2094
        // Code size 7 (0x7)
        .maxstack 8

        IL_0000: ldarg.0
        IL_0001: call instance class Dog Dog::GiveBirth()
        IL_0006: ret
    } // end of method Dog::Animal_GiveBirth

    .method public hidebysig 
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
