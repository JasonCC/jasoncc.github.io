# I-cache coherency and pipeline flushing on modern x86/x64 CPUs

## Acronyms

SMC     Self Modifying Code
SMC     Shared Map Cache

## Intel SDM

- 8.1.3 Handling Self- and Cross-Modifying Code
> Chapter 8 Multiple-processor Management
- 11.6  SELF-MODIFYING CODE
> Chapter 11 Memory Cache Control

A write to a memory location in a code segment that is currently cached in the processor causes the associated cache line (or lines) to be invalidated. This check is based on the physical address of the instruction. In addition, the P6 family and Pentium processors check whether a write to a code segment may modify an instruction that has been prefetched for execution. **If the write affects a prefetched instruction, the prefetch queue is invalidated.** This latter check is based on the linear address of the instruction. For the Pentium 4 and Intel Xeon processors, a write or a snoop of an instruction in a code segment, where the target instruction is already decoded and resident in the trace cache, invalidates the entire trace cache. The latter behavior means that programs that self-modify code can cause severe degradation of performance when run on the Pentium 4 and Intel Xeon processors.

In practice, the check on linear addresses should not create compatibility problems among IA-32 processors. Appli- cations that include self-modifying code use the same linear address for modifying and fetching the instruction. Systems software, such as a debugger, that might possibly modify an instruction using a different linear address than that used to fetch the instruction, will execute a serializing operation, such as a CPUID instruction, before the modified instruction is executed, which will automatically resynchronize the instruction cache and prefetch queue. (See Section 8.1.3, “Handling Self- and Cross-Modifying Code,” for more information about the use of self-modi- fying code.)

## References

- [I-cache coherency and pipeline flushing on modern x86/x64 CPUs](https://groups.google.com/forum/#!topic/comp.arch/k8tKb2TzufM)
- [Using self modifying code under Linux](http://asm.sourceforge.net/articles/smc.html)
- [Coherence Domain Restriction on Large Scale Systems](https://parallel.princeton.edu/papers/yfu_micro15.pdf)
- [How is x86 instruction cache synchronized?](https://stackoverflow.com/questions/10989403/how-is-x86-instruction-cache-synchronized)

[Back](../)
