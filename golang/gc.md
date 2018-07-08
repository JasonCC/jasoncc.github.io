---
layout: default
---

# The Go Gargabe Collector (GC)

key points:
- Go's concurrent GC can work concurrently with the program and uses a write barrier.
- Algorithm: tricolor mark-and-sweep algorithm
- CMS: Concurrent Mark/Sweep collector

## The generational hypothesis

It has been known since 1984 [7] that most allocations "die young" i.e.
become garbage very soon after being allocated. This observation is called
the generational hypothesis and is one of the strongest empirical observations
in the entire PL engineering space. It has been consistently true across very
different kinds of programming languages and across decades of change in the
software industry.

Discovering this fact about programs was useful because it meant GC algorithms
could be designed to take advantage of it. These new *generational collectors*
has lots of improvements over the old stop-mark-sweep style:

- GC throughput
- Allocation performance
- Program throughput
- Pause time

They also introduce some downsides:

- Compability (there are no generational collectors for C++)
- Heap overhead
- Pause distribution
- Tuning
- Warmup time

Still, the benefits are so huge that basically all modern GC algorithms are
generational.

## The Go Concurrent Collector

As Go is a relatively ordinary imperative language with value types, its
memory access patterns are probably comparable to C# where the generational
hypothesis holds and thus .NET uses a generational collector.

Go programs exhibit strongly generational behavior, and the Go team are
exploring potentially expoliting that in future with something they call the
"request oriented collector"[8]. It has been observed that this is essentially
a renamed generational GC with a tweaked tenuring policy.

Despite that, Go's current GC is not generational. It just runs a plain old
mark/sweep in the background.

- GC throughput: The time needed to clear the heap of garbage scales with
  the size of heap.

- Compaction: No compaction. (tcmalloc style based memory allocator suitable
  for multi-threaded program)

- Program throughput: as the GC has to do a lot of work for every cycle, that
  steals CPU time from the program itself, slowing it down.

- Pause distribution: runtime has no choice but to stop your program entirely
  and wait for the GC cycle to complete. Thus when Go claims GC pauses are
  very low, this claim can only be true for the case where the GC has
  sufficient CPU time and headroom to outrun the main program. 
  So distribution is a bit unpredictable.

- Heap overhead: because collecting the heap via mark/sweep is very slow,
  you need lots of spare space to ensure you don't suffer a "concurrent
  mode failure". Go defaults to a heap overhead of 100% ... it doubles the
  amount of memory your program needs.

Go optimises for pause times as the expense of throughput to such an extent
that it seems willing to slow down your program by almost any amount in order
to get event just slightly faster pauses. (But it's still too expensive to
a program working as an OS kernel)

## Conclusion from Mike Hearn

But if you take one thing away, let it be this: garbage collection is a hard problem, really hard, one that has been studied by an army of computer scientists for decades. So be very suspicious of supposed breakthroughs that everyone else missed. They are more likely to just be strange or unusual tradeoffs in disguise, avoided by others for reasons that may only become apparent later.

## References

1. [Pusher: GC in theory and practice](https://making.pusher.com/golangs-real-time-gc-in-theory-and-practice/index.html)
2. [The Go Garbage Collector (GC)](http://www.mtsoukalos.eu/Go-Garbage-Collector)
3. [Go GC: Prioritizing low latency and simplicity](https://blog.golang.org/go15gc)
4. [(Java)GC Algorithms: Implementations](https://plumbr.io/handbook/garbage-collection-algorithms-implementations)
5. [Why golang garbage-collector not implement Generational and Compact gc?](https://groups.google.com/forum/#!msg/golang-nuts/KJiyv2mV2pU/wdBUH1mHCAAJ)
6. [Modern garbage collection--A look at the Go GC strategy](https://blog.plan99.net/modern-garbage-collection-911ef4f8bd8e)
7. [Generation Scavenging: A Non-disruptlve High Perfornm Storage Reclamation Algorithm](http://citeseerx.ist.psu.edu/viewdoc/download?doi=10.1.1.122.4295&rep=rep1&type=pdf)
8. [(ROC)Request Oriented Collector](https://docs.google.com/document/d/1gCsFxXamW8RRvOe5hECz98Ftk-tcRRJcDFANj2VwCB0/edit)
9. [Go GC:Latency Problem Solved](https://talks.golang.org/2015/go-gc.pdf)
10. [Andrey Sibiryov, Uber, Golang’s Garbage](https://www.usenix.org/sites/default/files/conference/protected-files/srecon17asia_slides_sibiryov.pdf)

[back](../)
