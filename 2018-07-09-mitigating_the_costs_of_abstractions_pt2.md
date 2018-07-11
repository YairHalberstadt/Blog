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

  public static int Add(int[] values, Func<int,int,int> addFunc)
	 {
		 var result = 0;
		 for (int i = 0; i < values.Length; i++)
		 {
			 result = addFunc(result, values[i]);
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

   public static int Add(int[] values, Adder adder)
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



I ran this in release mode with the -runtwice commandline parameter set. That way the jitter had a chance to work its magic.

It's not surprising that the generic add had similar performance to the interface add - once the instance of TAdder was created, the rest of the code is identical, and so the performance should be pretty much equivalent, given the loop ran for 1,000,000,000 iterations.

They were both slightly slower than the Func Add. Although a lot of work takes place during delegate instantiation, once a delegate is instantiated the first time it's called, the result is cached and reused on any future calls. This seems to be enough to improve it's performance beyond that of a virtual function call in large loops, at least on the architecture I was using. Of course we can't know exactly why that is without seeing the machine code the jitter output.

The straightforward add function is about 4 times as fast as the interface add. The jitter is able to emit a single machine op to add two integers, and so the addition is probably the quickest part of the loop. Given that that's the case, the standard addition is likely many more than 4 times faster than the interface add, as much of that we are measuring is the cost of accessing the array, iterating i, and comparing it to the array length.

This is because calling an interface method uses a virtual method call. This involves a vtable lookup to find the correct method for this instance. That method must then be called, and only then can we emit the single machine op to add the two integers.

Usually the overhead of a virtual method is irrelevant compared to the cost of the method itself, but for such a small method, the cost is huge.

### The Problem

So why is this relevant?


