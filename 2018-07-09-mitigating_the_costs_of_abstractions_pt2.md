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

I ran this in release mode on an x86 architecture with the -runtwice commandline parameter set. That way the jitter had a chance to work its magic. I did so a number of times, varying the order in which each function ran, and the results were consistent throughout.

Here are my results:

>Run #1
>  AddStandard          00:00:00.5807577
>  AddFunc              00:00:01.8229438
>  AddClass             00:00:00.5325112
>  AddInterface         00:00:02.0599220
>  AddGeneric           00:00:02.0681714
>Run #2
>  AddStandard          00:00:00.5684420
>  AddFunc              00:00:01.8062895
>  AddClass             00:00:00.5387222
>  AddInterface         00:00:02.0576619
>  AddGeneric           00:00:02.0390805

It's not surprising that the generic add had similar performance to the interface add - once the instance of TAdder was created, the rest of the code is identical, and so the performance should be pretty much equivalent, given the loop ran for 1,000,000,000 iterations.

They were both slightly slower than the Func Add. Although a lot of work takes place during delegate instantiation, once a delegate is instantiated the first time it's called, the result is cached and reused on any future calls. This seems to be enough to improve it's performance beyond that of a virtual function call in large loops, at least on the architecture I was using. Of course we can't know exactly why that is without seeing the machine code the jitter output.

The straightforward add function is about 4 times as fast as the interface add. The jitter is able to emit a single machine op to add two integers, and so the addition is probably the quickest part of the loop. Given that that's the case, the standard addition is likely many more than 4 times faster than the interface add, as much of that we are measuring is the cost of accessing the array, iterating i, and comparing it to the array length.

This is because calling an interface method uses a virtual method call. This involves a vtable lookup to find the correct method for this instance. That method must then be called, and only then can we emit the single machine op to add the two integers.

Usually the overhead of a virtual method is irrelevant compared to the cost of the method itself, but for such a small method, the cost is huge.

### The Problem

So why is this relevant?


