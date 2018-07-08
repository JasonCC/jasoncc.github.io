---
layout: default
---

# Syscalls

## Sources (linux-3.2.33)

```c
arch/x86/include/asm/unistd_64.h 	// This file contains the system call numbers
arch/x86/kernel/entry_64.S 		// It contains the system-call and fault low-level handling routines.
arch/x86/kernel/syscall_64.c 		// System call table for x86-64
include/linux/syscall.h 		// Linux syscall interfaces (non-arch-specific)
arch/x86/kernel/process.c 		// sys_clone()
```

## Sources (linux-4.4 and later)

```c
arch/x86/entry/syscalls/syscall_64.tbl 	// common entry for x86_64 and x32
arch/x86/entry/syscalls/syscall_32.tbl 	// for i386 entries
```

## Adding a New System Call

### Generic System Call Implementation

1. `CONFIG` option for the new function, normally in `init/Kconfig`.
2. `SYSCALL_DEFINEn(xyzzy, ...)` for the entry pooint
3. corresponding prototype in `include/linux/syscalls.h`
4. generic table entry in `include/uapi/asm-generic/unistd.h` (this step can be ingored in x86 case.)
5. fallback stub in `kernel/sys_ni.c`

### x86 System Call Implementation

### Practice

1. Add `CONFIG_` option for the new function in `init/Kconfig`.
2. Wrapper new function code with `#ifdef CONFIG_XXX`

## Futexes

- [A futex overview and update](https://lwn.net/Articles/360699/)
- [locklessinc. Mutexes and Condition Variables using Futexes](https://locklessinc.com/articles/mutex_cv_futex/)
- [Rusty Russel. Fuss, Futexes and Furwocks: Fast Userlevel Locking in Linux](https://www.kernel.org/doc/ols/2002/ols2002-pages-479-495.pdf)
- [Ulrich Drepper. Futexes Are Tricky](https://www.akkadia.org/drepper/futex.pdf)
- [Requeue PI: Making Glibc Condvars PI-Aware](http://static.lwn.net/images/conf/rtlws11/papers/paper.10.html)
- [FUTEX + rwsem = SNAFU](https://lwn.net/Articles/124747/)
- [lwn Kernel: Futex](https://lwn.net/Kernel/Index/#Futex)
- [DESIGN-rwloc](https://github.com/lattera/glibc/blob/master/nptl/DESIGN-rwlock.txt)
- [robust-futexes](https://www.kernel.org/doc/Documentation/robust-futexes.txt)

### Glibc Bugs

- [Bug 14958 - Concurrent reader deadlock in pthread_rwlock_rdlock()](https://sourceware.org/bugzilla/show_bug.cgi?id=14958)

## Glibc Wrappers

- [Glibc wrappers for (nearly all) Linux system calls](https://lwn.net/Articles/655028/)

## Tools

- **ausyscall(8)**
> mapping syscall names and nubmers

```
$ ausyscall i386 open
open               5
mq_open            277
openat             295
perf_event_open    336
open_by_handle_at  342

$ ausyscall x86_64 open
open               2
mq_open            240
openat             257
perf_event_open    298
open_by_handle_at  304

$ ausyscall --dump | head -n10
Using x86_64 syscall table:
0       read
1       write
2       open
3       close
4       stat
5       fstat
6       lstat
7       poll
8       lseek
```

# References

- [kernel entries in entry_64.S](https://www.kernel.org/doc/Documentation/x86/entry_64.txt)
- [Adding a New System Call](https://www.kernel.org/doc/html/latest/process/adding-syscalls.html?highlight=syscall)
- [audit-userspace](https://github.com/linux-audit/audit-userspace)

[back](../)

