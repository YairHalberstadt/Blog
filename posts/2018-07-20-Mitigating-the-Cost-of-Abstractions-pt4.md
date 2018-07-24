---
Title: Mitigating the Cost of Abstractions pt4. Benchmarking Caches in Loops
---

### Mitigating the Cost of Abstractions pt4. Benchmarking Caches in Loops

Consider the 4 following Add functions:

``` csharp

    private static int[] _values;

		private static int _length;

		private static int Length => _values.Length;

		public static void Run( int[] values)
		{
			_values = values;
			_length = _values.Length;
      AddProperty();
      AddField();
      AddNormal();
      AddCache();
		}
    
		public static void AddProperty()
		{
			var result = 0;
			for (int i = 0; i < Length; i++)
			{
				result += _values[i];
			}

			return result;
		}

		public static int AddField()
		{
			var result = 0;
			for (int i = 0; i < _length; i++)
			{
				result += _values[i];
			}

			return result;
		}

		public static int AddNormal()
		{
			var result = 0;
			for (int i = 0; i < _values.Length; i++)
			{
				result += _values[i];
			}

			return result;
		}

		public static int AddCache()
		{
			var result = 0;
			var length = _values.Length;
			for (int i = 0; i < length; i++)
			{
				result += _values[i];
			}

			return result;
		}

```

All 4 use a a loop with the same code in the loop to add the array. Where they differ is what they check the loop against.

The first check the index against a property which gets the length of the array. The second checks against a field where we've cached the length. The third checks directly against the length of the array. The fourth checks against a local variable where we've cached the length.

Which will be most performant?

On the one hand those where we cache the result avoid calling array.Length so often. Since array.Length is not in fact a simple property, but actually calls into unmanaged code, this might improve performance.

On the other hand maybe the compiler/jitter is clever enough to do the caching for us. And maybe checking against the length of the array directly allows the Jitter to skip the bounds check on the array.

In short we can't know without benchmarking. We as it happens, is exactly what I've done.

Like last time I've used Jon Skeets benchmarking framework, in release mode, on an x86 architecture, with the -runtwice command line parameter.

All results were consistent across multiple repeats, and across varying sizes of the array used.

Here is the code I used.

``` csharp

using System;
using System.Collections.Generic;
using System.Collections.Immutable;
using System.Text;
using System.Runtime.CompilerServices;

namespace performanceTests
{
	class LoopTests
	{
		private static int[] _values = new int[1000000000];

		private static int _result;

		private static int _length;

		private static int Length => _values.Length;

		public static void Init(string[] args)
		{
			for (int i = 0; i < _values.Length; i++)
			{
				_values[i] = i;
			}
			_length = _values.Length;
		}

		public static void Check()
		{
			Console.WriteLine(_result);
		}

		[Benchmark]
		public static void AddProperty()
		{
			var result = 0;
			for (int i = 0; i < Length; i++)
			{
				result += _values[i];
			}

			_result = result;
		}

		[Benchmark]
		public static void AddField()
		{
			var result = 0;
			for (int i = 0; i < _length; i++)
			{
				result += _values[i];
			}

			_result = result;
		}

		[Benchmark]
		public static void AddNormal()
		{
			var result = 0;
			for (int i = 0; i < _values.Length; i++)
			{
				result += _values[i];
			}

			_result = result;
		}

		[Benchmark]
		public static void AddCache()
		{
			var result = 0;
			var length = _values.Length;
			for (int i = 0; i < length; i++)
			{
				result += _values[i];
			}

			_result = result;
		}
	}
}

```

And here are the results.

```

Run #1
  AddProperty          00:00:00.5335864
  AddField             00:00:00.6083947
  AddNormal            00:00:00.5314011
  AddCache             00:00:00.5779477
Run #2
  AddProperty          00:00:00.5333801
  AddField             00:00:00.6139761
  AddNormal            00:00:00.5312850
  AddCache             00:00:00.5739332
  
```

Not a huge amount in it. Using the property or just straightforward access to the array do seem to be slightly faster, and were consistently so, but not by enough to make much a difference on most codebases - remember, this is less than 0.1 seconds across a billion loops. Unless the stuff happening inside the loop is very fast indeed, that's not going to make much of a difference.

What about though if we store the values in a list, not an array?

```

Run #1
  AddProperty          00:00:00.9148230
  AddField             00:00:00.7792866
  AddNormal            00:00:00.9058388
  AddCache             00:00:00.6283438
Run #2
  AddProperty          00:00:00.8993353
  AddField             00:00:00.7766070
  AddNormal            00:00:00.9050327
  AddCache             00:00:00.6279809
  
```

Woah. Suddenly checking against the count of the list is slower than caching the value by about 40 to 50 percent. Again it's not much in the grand scheme of things, but it does seem surprising.

I imagine this is because the length of a list can change, so the Jitter doesn't know it can cache the count.

So let's try to make arrays more difficult for the Jitter. Let's call instead of adding the integers directly, let's make a function `int Add(int a, int b) => a + b` and use that to perform addition inside the loops. Let's make this Add function virtual so that the Jitter doesn't know what happens inside it.

Then lets make _values public. That way the Jitter can't know for sure that when Add is called inside the loop it doesn't replace _values with another array, and so maybe it will no longer be able to cache _value.Length.

Let's see the results:

```

Run #1
  AddProperty          00:00:01.8455796
  AddField             00:00:01.5908046
  AddNormal            00:00:01.5638944
  AddCache             00:00:01.5416786
Run #2
  AddProperty          00:00:01.8052111
  AddField             00:00:01.5521521
  AddNormal            00:00:01.5575613
  AddCache             00:00:01.5555272

```

Woah!

AddProperty is suddenly far slower than everything else!

But interestingly enough AddNormal is no slower, which suggests something else is going on. Let's see what happens when we reorder the benchmarks:

```

 Run #1
  AddField             00:00:01.8848794
  AddNormal            00:00:01.6330310
  AddProperty          00:00:01.5610518
  AddCache             00:00:01.5592312
Run #2
  AddField             00:00:01.8052472
  AddNormal            00:00:01.5768331
  AddProperty          00:00:01.5634327
  AddCache             00:00:01.5566960

```

Aha.

So it just seems the first benchmark to run is slower in this case. I don't know why, but it's probably something to do with the virtual function call, and nothing to do with arrays.

The other benchmarks are all pretty much identically fast.

#### Conclusion

I think the conclusion here is not anything specific about loops, or caching. It's a more general lesson about optimising.

There is no point trying to second guess the Jitter, and assume that if we organise our code this way it will be faster than the other way.

Firstly because this will make our code trickier to read, and secondly because it is just as likely to be wrong as right.

A thousand different factors can effect the exact performance of your code, and it's impossible to estimate upfront what effect your code has.

Instead when writing our code we should do so in the most idiomatic, clearest way. Afterwards, we can profile in a **real world scenario** , and see what works best.

Micro-benchmarks are useful in specific cases. But they cannot tell us what the real impact of our code change will be, especially in something as subtle as caching. As we've seen, the slightest change to the code being benchmarked can completely alter the results.

Only a real world scenario test can tell us whether such changes our truly helpful*.

One final point.

The Jitter is different for every architecture, and every runtime. What's faster now for you maybe not be faster for someone else right now, or you in a year.

Whereas if you write clean code, that stays forever.

*Of course this does depend on what you're doing. Linq will always be slower than the equavelent foreach loop, since it's simply got more work to do. But changing linq to foreach is an algorithmic change to your code that the compiler and Jitter don't have the authority to do. Whereas caching they are perfectly able to work out for themselves, and often better than you at deciding whether to. Same goes for things like inlining and calling the garbage collector.

