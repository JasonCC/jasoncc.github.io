---
layout: default
---

# Kernel Mode Process Stack on x86-64 bit

## Linux-4.4 ~ 4.10

`THREAD_SIZE_ORDER` is used to control kernel mode process stack. It's value
is 2 by default. And it means that the kernel stack is expanded to 4 page frames
(16KB).

From [`kernel/fork.c`](http://lxr.free-electrons.com/source/kernel/fork.c#L172)

```
172 static unsigned long *alloc_thread_stack_node(struct task_struct *tsk, int node)
173 {
174 #ifdef CONFIG_VMAP_STACK
175         void *stack;
176         int i;
177
178         local_irq_disable();
179         for (i = 0; i < NR_CACHED_STACKS; i++) {
180                 struct vm_struct *s = this_cpu_read(cached_stacks[i]);
181
182                 if (!s)
183                         continue;
184                 this_cpu_write(cached_stacks[i], NULL);
185
186                 tsk->stack_vm_area = s;
187                 local_irq_enable();
188                 return s->addr;
189         }
190         local_irq_enable();
191
192         stack = __vmalloc_node_range(THREAD_SIZE, THREAD_SIZE,
193                                      VMALLOC_START, VMALLOC_END,
194                                      THREADINFO_GFP | __GFP_HIGHMEM,
195                                      PAGE_KERNEL,
196                                      0, node, __builtin_return_address(0));
197
198         /*
199          * We can't call find_vm_area() in interrupt context, and
200          * free_thread_stack() can be called in interrupt context,
201          * so cache the vm_struct.
202          */
203         if (stack)
204                 tsk->stack_vm_area = find_vm_area(stack);
205         return stack;
206 #else
207         struct page *page = alloc_pages_node(node, THREADINFO_GFP,
208                                              THREAD_SIZE_ORDER);
209                                              ^~~~~~~~~~~~~~~~~~
210         return page ? page_address(page) : NULL;
211 #endif
212 })}
```

And [`Linux/arch/x86/include/asm/page_64_types.h`](http://lxr.free-electrons.com/source/arch/x86/include/asm/page_64_types.h#L14)

```
1 #ifndef _ASM_X86_PAGE_64_DEFS_H
2 #define _ASM_X86_PAGE_64_DEFS_H
3
4 #ifndef __ASSEMBLY__
5 #include <asm/kaslr.h>
6 #endif
7
8 #ifdef CONFIG_KASAN
9 #define KASAN_STACK_ORDER 1
10 #else
11 #define KASAN_STACK_ORDER 0
12 #endif
13
14 #define THREAD_SIZE_ORDER       (2 + KASAN_STACK_ORDER)
           ^~~~~~~~~~~~~~~~~        ^~~
15 #define THREAD_SIZE  (PAGE_SIZE << THREAD_SIZE_ORDER)
16 #define CURRENT_MASK (~(THREAD_SIZE - 1))
17
18 #define EXCEPTION_STACK_ORDER (0 + KASAN_STACK_ORDER)
19 #define EXCEPTION_STKSZ (PAGE_SIZE << EXCEPTION_STACK_ORDER)
20
21 #define DEBUG_STACK_ORDER (EXCEPTION_STACK_ORDER + 1)
22 #define DEBUG_STKSZ (PAGE_SIZE << DEBUG_STACK_ORDER)
23
24 #define IRQ_STACK_ORDER (2 + KASAN_STACK_ORDER)
25 #define IRQ_STACK_SIZE (PAGE_SIZE << IRQ_STACK_ORDER)
26
27 #define DOUBLEFAULT_STACK 1
28 #define NMI_STACK 2
29 #define DEBUG_STACK 3
30 #define MCE_STACK 4
31 #define N_EXCEPTION_STACKS 4  /* hw limit: 7 */
```

## Linux-3.2.y

`THREAD_ORDER` is used to control kernel mode process stack.

For x86_64, its value is 1 and it is defined in
`linux-3.2/arch/x86/include/asm/page_64_types.h`. That means the kernel mode
process stack is 2 page frames (8,192 bytes).

```
1 #ifndef _ASM_X86_PAGE_64_DEFS_H
2 #define _ASM_X86_PAGE_64_DEFS_H
3
4 #define THREAD_ORDER	1
5 #define THREAD_SIZE  (PAGE_SIZE << THREAD_ORDER)
6 #define CURRENT_MASK (~(THREAD_SIZE - 1))
```

And `arch/x86/include/asm/thread_info.h`.

```
3 #define alloc_thread_info_node(tsk, node)                               \
2 ({                                                                      \
1         struct page *page = alloc_pages_node(node, THREAD_FLAGS,        \
167                                            THREAD_ORDER);             \
1         struct thread_info *ret = page ? page_address(page) : NULL;     \
2                                                                         \
3         ret;                                                            \
4 })
```

## References

- [Handing stack overflows: linux kernel stack design highlights](http://ru.kernelnewbies.org/node/44)
- [x86_64: expand kernel stack to 16K](https://lwn.net/Articles/600645/)
- [Expanding the kernel stack](https://lwn.net/Articles/600644/)
- [4K stacks by default?](https://lwn.net/Articles/279229/)
- [Re: x86_64: expand kernel stack to 16K](https://lwn.net/Articles/600647/)
- [linus Re: [RFC 2/2] x86_64: expand kernel stack to 16K](https://lwn.net/Articles/600649/)
- [Kernel stack](https://lwn.net/Kernel/Index/#Kernel_stack)
- [Kernel stacks on x86-64 bit](https://www.kernel.org/doc/Documentation/x86/kernel-stacks)


[back](../)

