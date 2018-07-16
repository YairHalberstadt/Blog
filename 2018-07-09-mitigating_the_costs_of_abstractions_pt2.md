---
Title: Mitigating The Costs Of Abstractions. Using Functions for Simple Arithmetic
---

## Mitigating The Costs Of Abstractions. Using Functions for Simple Arithmetic

Consider the following 5 implementations of a simple Add Function:

``` csharp

public static int Add(int[] values)
{
	var result = 0;
	for (int i = 0; i < values.Length; i++)
	{
		result += values[i];
	}

	return result;
}

public static int Add(int[] values, Func<int, int, int> addFunc)
{
	var result = 0;
	for (int i = 0; i < values.Length; i++)
	{
		result = addFunc(result, values[i]);
	}

	return result;
}

public static int Add(int[] values, Adder adder)
{
	var result = 0;
	for (int i = 0; i < values.Length; i++)
	{
		result = adder.Add(result, values[i]);
	}

	return result;
}

public static int Add(int[] values, IAdder adder)
{
	var result = 0;
	for (int i = 0; i < values.Length; i++)
	{
		result = adder.Add(result, values[i]);
	}

	return result;
}

public static int Add<TAdder>(int[] values) where TAdder : IAdder, new()
{
	var adder = new TAdder();
	var result = 0;
	for (int i = 0; i < values.Length; i++)
	{
		result = adder.Add(result, values[i]);
	}

	return result;
}

```

How does their performance compare to each other?

I can guess but I can't know. Like always I'll have to profile. I've used Jon Skeets simple benchmarking framework (available at http://jonskeet.uk/csharp/benchmark.html) to check. 

Here is the code I used:

<details>
  <summary>Click to view code</summary>
	
``` csharp

using System;
using System.Collections.Generic;
using System.Text;

namespace performanceTests
{
	class function_tests
	{
		private static int[] _values = new int[1000000000];
		private static Func<int, int, int> _addFunc;
		private static IAdder _iadder;
		private static Adder _adder;

		private static int _result;

		public static void Init(string[] args)
		{
			for (int i = 0; i < _values.Length; i++)
			{
				_values[i] = i;
			}

			_addFunc = (a, b) => a + b;
			_iadder = _adder = new Adder();
		}

		public static void Check()
		{
			Console.WriteLine(_result);
		}

		[Benchmark]
		public static void AddStandard()
		{
			_result = Add(_values);
		}

		[Benchmark]
		public static void AddFunc()
		{
			_result = Add(_values, _addFunc);
		}

		[Benchmark]
		public static void AddClass()
		{
			_result = Add(_values, _adder);
		}

		[Benchmark]
		public static void AddInterface()
		{
			_result = Add(_values, _iadder);
		}

		[Benchmark]
		public static void AddGeneric()
		{
			_result = Add<Adder>(_values);
		}

		public static int Add(int[] values)
		{
			var result = 0;
			for (int i = 0; i < values.Length; i++)
			{
				result += values[i];
			}

			return result;
		}

		public static int Add(int[] values, Func<int, int, int> addFunc)
		{
			var result = 0;
			for (int i = 0; i < values.Length; i++)
			{
				result = addFunc(result, values[i]);
			}

			return result;
		}

		public static int Add(int[] values, Adder adder)
		{
			var result = 0;
			for (int i = 0; i < values.Length; i++)
			{
				result = adder.Add(result, values[i]);
			}

			return result;
		}

		public static int Add(int[] values, IAdder adder)
		{
			var result = 0;
			for (int i = 0; i < values.Length; i++)
			{
				result = adder.Add(result, values[i]);
			}

			return result;
		}

		public static int Add<TAdder>(int[] values) where TAdder : IAdder, new()
		{
			var adder = new TAdder();
			var result = 0;
			for (int i = 0; i < values.Length; i++)
			{
				result = adder.Add(result, values[i]);
			}

			return result;
		}
	}

	internal interface IAdder
	{
		int Add(int a, int b);
	}

	class Adder : IAdder
	{
		public int Add(int a, int b) => a + b;
	}
}

```
</details>
<br/>
I ran this in release mode on an x86 architecture with the -runtwice commandline parameter set. That way the jitter had a chance to work its magic. I did so a number of times, varying the order in which each function ran, and the results were consistent throughout.

Here are my results:


```
Run #1
  AddStandard          00:00:00.5807577
  AddFunc              00:00:01.8229438
  AddClass             00:00:00.5325112
  AddInterface         00:00:02.0599220
  AddGeneric           00:00:02.0681714
Run #2
  AddStandard          00:00:00.5684420
  AddFunc              00:00:01.8062895
  AddClass             00:00:00.5387222
  AddInterface         00:00:02.0576619
  AddGeneric           00:00:02.0390805
```
It's not surprising that the generic add had similar performance to the interface add - once the instance of TAdder was created, the rest of the code is identical, and so the performance should be pretty much equivalent, given the loop ran for 1,000,000,000 iterations.

They were both slightly slower than the Func Add. Although a lot of work takes place during delegate instantiation, once a delegate is instantiated the first time it's called, the result is cached and reused on any future calls. This seems to be enough to improve it's performance beyond that of a virtual function call in large loops, at least on the architecture I was using. Of course we can't know exactly why that is without seeing the machine code the jitter output.

The straightforward add function is about 4 times as fast as the interface add. The jitter is able to emit a single machine op to add two integers, and so the addition is probably the quickest part of the loop. Given that that's the case, the standard addition is likely many more than 4 times faster than the interface add, as much of that we are measuring is the cost of accessing the array, iterating i, and comparing it to the array length.

This is because calling an interface method uses a virtual method call. This involves a vtable lookup to find the correct method for this instance. That method must then be called, and only then can we emit the single machine op to add the two integers.

Usually the overhead of a virtual method is irrelevant compared to the cost of the method itself, but for such a small method, the cost is huge.

What's most interesting though is that addclass where we call a non-virtual method to add, suffers no performance costs compared to adding directly. In fact it was slightly faster on every repeat I did (I have no idea why this is. Maybe it's just a fluke?). 

This is because the Jitter inlines the function call, so effectively addclass and addstandard run the same code.

With virtual function calls though, the Jitter isn't yet clever enough to inline the function call, as the function changes depending on the type that's passed in.

Thus using a virtual function as opposed to a non - virtual one degrades our performance by 4 times.

Virtual functions are beautiful things, but in performance critical code, weigh up every single one before you use it, and always seal functions of they don't need to be further inherited.

### The Problem

So why would I every provide an add function through a virtual method in the first place?

I'm currently implementing a linear algebra library.

One of the core constructs in linear algebra is a vector. A vector is essentially a list of items. Each item must have addition, subtraction, multiplication and a couple of other things defined for it.

Adding one vector to another involves returning a new vector with each of its values the sum of the two values in the vectors being added.

If that's not clear enough, here is a simple implementation of Vector<int>.Add()

``` csharp

public class IntVector : IVector<int>
{
    private int[] _items;
    
   // constructor and other code goes here. IVector implements IEnumerable.

    public IVector<int> Add(IVector<int> addend)
    {
    var result = this.Zip(addend, (a,b) => a+b).ToArray();

    return new IntVector(result);
    }
}


```

Obviously that could be improved significantly by using a for loop instead of IEnumerable.Zip() but I'm not focusing on that for now.

This looks all fine and dandy, but that only works for an int. I want to define a Vector<T> that will work for all values of T.

I could do this by injecting a function or interface into    the constructor, and use that to define addition on T. But then I could instantiate two instances of Vector<T> which have different definitions for Add(). What would happen if I added them?

Really I want the definition of addition to be part of the type signature.

To do this I need to do the following:

``` csharp

public class Vector<TDataType, TAdder>: IVector<TDataType, TAdder> where TAdder : IAdder<TDataType>, new()
{
    // Constructor, items, properties, accessors etc. go here.

    public IVector<TDataType, TAdder> Add(IVector<TDataType, TAdder> addend)
    {
        var result = new TDataType[Length];
        var adder = new TAdder();
        for(int i = 0; I < Length; i++)
        {
            result[i] = adder.Add(this[i], addend[i]);
        }
        return new Vector<TDataType, TAdder>(result);
    }
}

```

Now this is overengineered, but functional. It works. But we are now calling a virtual method to do addition, with all the overhead that entails.

So we are stuck between a rock and a hard place: either we make Vector slow, or we have to define it separately for each DataType.

Code generation could help with the second problem, but that's not a neat solution.

In the end the decision I made was that we couldn't compromise either. It is unacceptable for a vector library to be slow on basic types, but at the same time, it would be ugly if it couldn't extend to other types. Besides fast vector libraries already exists for such things as doubles. One of the unique selling points of my library is it's extensibility, and it's fealty to mathematical definitions and concepts. As such I could not justify reducing a vector to something that just acts on numbers.

The trick is actually extremely simple but requires a little in-depth knowledge about c#:

What happens if we replace Adder with a struct, instead of an interface, and redo our performance tests?

```
Run #1
  AddStandard          00:00:00.5878752
  AddFunc              00:00:02.0918007
  AddClass             00:00:00.4395688
  AddInterface         00:00:02.8117917
  AddGeneric           00:00:00.4299338
Run #2
  AddStandard          00:00:00.5643042
  AddFunc              00:00:02.1003003
  AddClass             00:00:00.4203489
  AddInterface         00:00:02.7876626
  AddGeneric           00:00:00.4175348
```

Wow! AddGeneric now performs even faster than the normal add function!

Why is this?

I can't be sure why it's faster, but I do know why it's as fast. Part of the guarantee of generics in C# is that they will not cause boxing of structs. To avoid this, a new version of the generic type is instantiated for each struct type that's used.

As a result the Jitter knows that Add<Adder> is only going to be used with an Adder, and so can aggressively optimise the function by inlining Adder.Add().

So all we have to do is implement our adders as structs and we gain the same performance as a non-generic library!

Of course this comes at a cost.  Instantiating a new generic type for each struct used comes with a significant memory overhead, and can also effect things like cache performance - that _might_ be why both AddInterface and AddFunc are now slower than before. But seeing as int, double, complex etc. are all structs anyway, this would occur whatever happened. The extra overhead would only occur if you instantiated Vector on a non struct datatype, or used more than one adder with an struct datatype. Then you'll have to decide whether to use a class or a struct, but even then in 99% of case I imagine a struct is appropriate - as ever profile and see what works best.



