---
Title: CLR Internals. Runtime Type Information For Structs
---

Objects will often exhibit different behaviour at runtime depending on their runtime type. For example, two different implementations of the same interface will have different function calls when the same interface method is called on them

The necessary information to access the runtime type of an object is stored in its object header. This is 4 bytes that every reference type has before it's fields begin.

You can access the runtime type by calling object.GetType();

Now structs don't have headers, so how can they exhibit type specific behaviour.

For example, how does 1.ToString(), a virtual function, know to return "1", and not "System.Int32", as the default implementation of object.ToString() does?

Similarly a struct can implement an interface. How then do we know which interface method to call on a struct? And how does struct.GetType() work?

All structs are sealed. As such when we see something like:

``` csharp

public static string GetString(int i) => i.ToString();

```

We know that i will always be an int, and not a more derived type, so we can use a non-virtual function call to int.ToString();

That's well and fine in most cases, but what if we cast our struct to a object or an interface, and call object.ToString() on it?

Well then boxing will occur. Essentially, we wrap the struct in an implicitly created reference type, whose only field is the struct. As a reference type, the boxed struct has a header, so runtime type information is available.

Since at the moment boxing occurs, we are creating a reference type based on an instance of a struct, we know based on compile time information the type of the struct, and can create the reference with the appropriate header accordingly.

We could achieve something similar even if structs could not be boxed by using a Reference<T> class.

``` csharp

public sealed class Reference<T>  where T : Value type
{
    public T Value {get;}
    public Reference<T>(T value) => Value = value;
    public override string ToString() => Value.ToString();
    public override int GetHashCode() => Value.GetHashCode();
    public override bool Equals(object obj) => Value.Equals(obj);

    public static implicit operator Reference<T>(T value) => new Reference<T>(value);
    public static implicit operator T(Reference<T> ref) => ref.Value;
} 

// System.ValueType defines an implicit cast from ValueType to object by returning a reference<T>.
//System.Object defines an explicit cast from object to T where T is a struct, by checking if object is a Reference<T> and if so returning the Value.


public static void SomeMethod( object obj);

SomeMethod(22); // This calls the implicit cast to an object by creating a Reference<int>
```

To support interfaces we could have the compiler automatically create a class inheriting from Reference<T> every time a struct is defined, and have that class implement all the interfaces.

Very little runtime support is absolutely necessary for this. Most of the features boxing a struct provides can be achieved via compile time tricks in theory.

In practice the CLR tends to provide more support than it needs to for many language features. This is as a result of it's aim to be used across languages. By implementing more in the CLR and not leaving it up to the languages to implement all the features themselves, it means that the languages on top of the CLR are likely to be more generally inter-compatable.

A final note about GetType()

Whilst in theory Struct.GetType() could be turned into a compile time constant, at the moment it instead boxes the type and calls GetType on that.

Since struct.GetType() will always return the same as typeof(struct) for an unboxed struct, one might as well use typeof since it is a compile time constant and avoids boxing the struct.
