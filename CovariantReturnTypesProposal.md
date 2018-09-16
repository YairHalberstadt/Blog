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

### 2. Test Cases

**case a - overriding a virtual method**
```
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
```
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
```
class Program
{
    static void Main(string[] args)
    {
        Cat cat = new Cat();
        var babyCat = cat.GiveBirth(); // type of var should be Animal
        babyCat.GetType(); // should be Cat
        Cat babyCat2 = cat.GiveBirth(); // should not compile
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
```
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
    Cat IAnimal.GiveBirth() => new Cat(); // Should not Compile

    IAnimal IAnimal.GiveBirth() => new Dog(); // Should Compile

    public Cat GiveBirth() => new Cat(); // Should Compile
}
```

**case e - overriding a covarient override**
```
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

public class StBernard : Dog
{
    public override Animal GiveBirth() => new StBernard(); // Should not Compile
}
```

**case f - overriding a covarient abstract override**
```
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

public class StBernard : Dog
{
    public override Animal GiveBirth() => new StBernard(); // Should not Compile
}
```

**case g - sealed overrides**

```
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
    public override Dog GiveBirth() => new Poodle(); // Should not compile
}

public class Retriever : Dog
{
    public override Retriever GiveBirth() => new Retriever(); // Should not Compile
}

public class Cat : Animal
{
    public sealed override Animal GiveBirth() => new Cat();
}

public class Tiger : Cat
{
    public override Tiger GiveBirth() => new Tiger(); // Should not compile
}
```

**case h - attribute inheritance**

```
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
