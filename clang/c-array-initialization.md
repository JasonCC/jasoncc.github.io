---
layout default
---

# Array Initialization

## How does the compiler fill values in `char array[100] = {0};`? What's the magic behind it?

I wanted to know how internally compiler initializes.

It's not magic.

The behavior of this code in C is described in section 6.7.8.21 of the C specification (online draft of C spec): for the elements that don't have a specified value, the compiler initializes pointers to NULL and arithmetic types to zero (and recursively applies this to aggregates).

The behavior of this code in C++ is described in section 8.5.1.7 of the C++ specification (online draft of C++ spec): the compiler aggregate-initializes the elements that don't have a specified value.

## Simple Test

Here is a simple code to test how GCC generates the assembly codes.

```
#include <string.h>

extern int bar(char *a, char *b);

void foo(void)
{
        char a[9];
        char b[10] = { 0 };

        memset(a, 0, sizeof(a));

        bar(a, b);
}
```

Optimization level `-O1`.

`gcc -Wall -O1 -c array-init.c`
`objdump -d array-init.o`

```
Disassembly of section .text:

0000000000000000 <foo>:
   0:   48 83 ec 28             sub    $0x28,%rsp
   4:   48 c7 04 24 00 00 00    movq   $0x0,(%rsp)
   b:   00
   c:   66 c7 44 24 08 00 00    movw   $0x0,0x8(%rsp)
  13:   48 c7 44 24 10 00 00    movq   $0x0,0x10(%rsp)
  1a:   00 00
  1c:   c6 44 24 18 00          movb   $0x0,0x18(%rsp)
  21:   48 89 e6                mov    %rsp,%rsi
  24:   48 8d 7c 24 10          lea    0x10(%rsp),%rdi
  29:   e8 00 00 00 00          callq  2e <foo+0x2e>
  2e:   48 83 c4 28             add    $0x28,%rsp
  32:   c3                      retq
```

Optimization `-O2`

`gcc -Wall -O2 -c array-init.c`
`objdump -d array-init.o`

Assembly codes get more optimization to leverage "Instruction-Level Parallelism".

```
Disassembly of section .text:

0000000000000000 <foo>:
   0:   48 83 ec 28             sub    $0x28,%rsp
   4:   31 c0                   xor    %eax,%eax
   6:   48 8d 74 24 10          lea    0x10(%rsp),%rsi
   b:   48 89 e7                mov    %rsp,%rdi
   e:   48 c7 44 24 10 00 00    movq   $0x0,0x10(%rsp)
  15:   00 00
  17:   66 89 44 24 18          mov    %ax,0x18(%rsp)
  1c:   48 c7 04 24 00 00 00    movq   $0x0,(%rsp)
  23:   00
  24:   c6 44 24 08 00          movb   $0x0,0x8(%rsp)
  29:   e8 00 00 00 00          callq  2e <foo+0x2e>
  2e:   48 83 c4 28             add    $0x28,%rsp
  32:   c3                      retq
```

P.S.: Note that memset will not be optimized when the optimization level is `O0`.
	  And the codes will look like the following.

```
00000000004004f6 <main>:
  4004f6:       55                      push   %rbp
  4004f7:       48 89 e5                mov    %rsp,%rbp
  4004fa:       48 83 ec 20             sub    $0x20,%rsp
  4004fe:       48 c7 45 e0 00 00 00    movq   $0x0,-0x20(%rbp)
  400505:       00
  400506:       66 c7 45 e8 00 00       movw   $0x0,-0x18(%rbp)
  40050c:       48 8d 45 f0             lea    -0x10(%rbp),%rax
  400510:       ba 09 00 00 00          mov    $0x9,%edx
  400515:       be 00 00 00 00          mov    $0x0,%esi
  40051a:       48 89 c7                mov    %rax,%rdi
  40051d:       e8 ce fe ff ff          callq  4003f0 <memset@plt>
  ^~~~~~~~~~~~~~^~~~~~~~~~~~~~~~~~~~~~~~^~~~~~~^~~~~~~~^~~~~~~~~~~
  400522:       b8 00 00 00 00          mov    $0x0,%eax
  400527:       c9                      leaveq
  400528:       c3                      retq
  400529:       0f 1f 80 00 00 00 00    nopl   0x0(%rax)
```

Also, note that in C++ (but not C), you can use an empty initializer list, causing the compiler to aggregate-initialize all of the elements of the array.

- [how does array[100] = {0} set the entire array to 0?](http://stackoverflow.com/questions/629017/how-does-array100-0-set-the-entire-array-to-0)
- [C Standard](http://www.open-std.org/jtc1/sc22/wg14/www/docs/n1124.pdf)
- [Strange assembly from array 0-initialization](http://stackoverflow.com/questions/531477/strange-assembly-from-array-0-initialization/531490)
- [6.27 Designated Initializers](https://gcc.gnu.org/onlinedocs/gcc-7.1.0/gcc/Designated-Inits.html#Designated-Inits)


[back](../)

