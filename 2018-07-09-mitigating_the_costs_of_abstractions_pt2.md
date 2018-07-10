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
