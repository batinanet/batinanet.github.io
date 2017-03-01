---
layout: post
title: Proper Way to Benchmark Code Execution Time in C#
tags: [c#, perfomance]
---

To properly benchmark code execution time we need to minimize interference from outside factors by taking into consideration the following items that can affect our measurement:

- Garbage Collector
- JIT Compilation
- Timing Precision
- Build Configuration
						
### Garbage Collector

When we fill the managed heap by creating new objects then the .Net Framework's garbage collector kick in and frees the memory by deleting objects that are no longer used. While the garbage collector is running all other managed threads are suspended. That will greatly impact our timing.

So, to prevent the garbage collector from running in the middle of our benchmark we need to manually trigger garbage collection before starting the benchmark. This can be accomplished with the following code:

```csharp
// Perform garbage collection.
GC.Collect();
GC.WaitForPendingFinalizers();
```

### JIT Compilation

The .Net applications are compiled into IL code. Before the first run IL code needs to be translated into native machine code. This process is known as Just-In-Time (JIT) compilation.

We need to make sure that the method we want to benchmark is already compiled into native machine code by simply invoking the method before the benchmark run.

```csharp
// Force JIT compilation of the method.
action.Invoke();
```

### Timing Precision

Timing needs to be precise as possible, so we can rule out DateTime and Timer objects because they are not precise enough for our needs. Luckily, .Net framework includes a StopWatch class which represents a high-resolution timer with a precision of &lt;1us. So that's what we will use.

### Build Configuration

Benchmarking must be done in a Release build outside of the debugger because the Debug configuration emmits debugging informations into application file and turns off code optimizations which make it run slower.

### Final Solution

Taking into account all of the above items here is the finished generic benchmarking class that you can use for your measurements.

```csharp
using System;
using System.Diagnostics;

public static class Benchmark
{
    public static void Run(string name, int iterations, Action action)
    {
        try
        {
            Console.Write($"Running benchmark '{name}' for {iterations} iterations... ");

            // Perform garbage collection.
            GC.Collect();
            GC.WaitForPendingFinalizers();

            // Force JIT compilation of the method.
            action.Invoke();

            // Run the benchmark.
            Stopwatch watch = Stopwatch.StartNew();
            for (int i = 0; i < iterations; i++)
            {
                action.Invoke();
            }
            watch.Stop();
            
            // Output results.
            Console.WriteLine($"Elapsed time {watch.ElapsedMilliseconds / iterations} ms.");
        }
        catch (OutOfMemoryException)
        {
            Console.WriteLine($"Out of memory!");
        }
    }
}
```

Usage example:

```csharp
Benchmark.Run("StreamReader.ReadToEnd", 1000, () =>
{
    using (StreamReader reader = new StreamReader(testFile))
    {
        string output = reader.ReadToEnd();
    }
});
```

Please leave a comment below. Thanks for reading.
