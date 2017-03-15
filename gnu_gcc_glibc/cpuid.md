---
layout: default
---

# Proper Use of x86/x86_64 CPUID Instruction with Extended Assembler

The CPUID instruction always uses EAX/EBX/ECX/EDX, even on 64-bit platforms.
On 64-bit platforms, the CPUID instruction sets the high 32-bit words of
RAX/RBX/RCX/RDX equal to 0.

This is a generally useful C implementation that works on 32- and 64-bit systems:

```
#include <stdio.h>

int main() {
	int i;
	unsigned int index = 0;
	unsigned int regs[4];
	int sum;
	__asm__ __volatile__(
#if defined(__x86_64__) || defined(_M_AMD64) || defined (_M_X64)
	"pushq %%rbx     \n\t" /* save %rbx */
#else
	"pushl %%ebx     \n\t" /* save %ebx */
#endif
	"cpuid            \n\t"
	"movl %%ebx ,%[ebx]  \n\t" /* write the result into output var */
#if defined(__x86_64__) || defined(_M_AMD64) || defined (_M_X64)
	"popq %%rbx \n\t"
#else
	"popl %%ebx \n\t"
#endif
	: "=a"(regs[0]), [ebx] "=r"(regs[1]), "=c"(regs[2]), "=d"(regs[3])
	: "a"(index));
	for (i=4; i<8; i++) {
		printf("%c" ,((char *)regs)[i]);
	}
	for (i=12; i<16; i++) {
		printf("%c" ,((char *)regs)[i]);
	}
	for (i=8; i<12; i++) {
		printf("%c" ,((char *)regs)[i]);
	}
	printf("\n");
}
```

# References

- [CPUID](https://en.wikipedia.org/wiki/CPUID)
- [Proper use of x86/x86_64 CPUID instruction with extended assembler](https://gcc.gnu.org/ml/gcc-help/2015-08/msg00088.html)
- [Function multi-versioning in GCC 6](https://lwn.net/Articles/691932/)
- [Function Multiversioning](https://gcc.gnu.org/wiki/FunctionMultiVersioning)
- [CPUID usage for interaction between Hypervisors and Linux](https://lwn.net/Articles/301888/)
- [GCC's cpuid.h](https://github.com/gcc-mirror/gcc/blob/master/gcc/config/i386/cpuid.h)



[back](../)




