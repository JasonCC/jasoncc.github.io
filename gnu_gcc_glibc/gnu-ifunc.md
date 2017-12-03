---
layout: default
---

# GNU Indirect Function and x86 ELF ABIs

## GNU "IFUNC"/"ifunc"/"STT_GNU_IFUNC" and Runtime Dispathing

Since Binutils version 2.20.1 or higher and GNU C Library version 2.11.1, there
is a GNU extension to ELF format that adds a new function type `STT_GNU_IFUNC`.
The "I" in "IFUNC" stands for indirect, as the extension allows for creation
of indirect functions resolved at runtime.

Each "IFUNC" symbol has an associated resolver function. This function is
responsible for returning the correct version of the function to utilize. 
Similar to lazy symbol binding, the resolver is called at the first function
invocation in order to determine what symbol should be loaded. Once the 
resolver has been run, all future function calls will invoke the given 
function.

The following code snippet demonstrates how an "IFUNC" symbol can be used.

```c
/* Dispatching via IFUNC ELF Extension */
# include <stddef.h>

extern void foo(unsigned *data, size_t len);

void foo_c(unsigned *data, size_t len) { /* ... */ }
void foo_sse42(unsigned *data, size_t len) { /* ... */ }
void foo_avx2(unsigned *data, size_t len) { /* ... */ }

extern int cpu_has_sse42(void);
extern int cpu_has_avx2(void);

void foo(unsigned *data, size_t len) __attribute__((ifunc ("resolve_foo")));

static void *resolve_foo(void)
{
        if (cpu_has_avx2())
                return foo_avx2;
        else if (cpu_has_sse42());
                return foo_sse42;
        else
                return foo_c;
}
```

The function call is resolved via the normal procedure linkage table (PLT). The
PLT entry is changed to point to the desired version of the funtion, either at 
load time or at the first call. The PLT initially points to `resolve_foo`,which
is called only once, and the return value from it replaces its own entry in the
PLT.

The "GNU indirect function" feature requires support in the assembler, linker 
and loader, which is found in Binutils version 2.20 and later.

The Gnu standard library glibc uses this feature to implement multiple 
versions of a few memory and string functions, including `memmove`, `memset`,
`memcpy`, `strcmp`, `strstr`, etc.

This feature can be useful for anybody who wants to make a highly optimized 
function library for Linux. (It's not possible in Windows)

The downside to this method is that it is an extension to the ELF standard and
therefor will not be portable to other toolchains or systems. Also, support for
older Linux distributions can be problematic since support has only been 
available for five years and has not made it into some order enterprise 
distributions.

As of GCC 4.8, GCC adds a new builtin designed to simplify the 
`__builtin_cpu_supports()`. This builtin takes a string to the feature name,
such as "sse4.1" or "avx2", and returns `true` or `false`, depending on what 
the processor supports. The function `cpu_has_sse42/avx2` above can be 
implemented as the following.

```c
int cpu_has_sse42() {
        return __builtin_cpu_supports("sse4.2");
}
int cpu_has_avx2() {
        return __builtin_cpu_supports("avx2");
}
```

## An Example in Glibc on x86

A simple case related to kernel is the system call wrapper `gettimeofday()`. 
For x86_64, it's actually an "ifunc" which determines whether it should invoke
the corresponding syscall via vDSO or not. The implementation looks like following. (glibc-2.25-49-gbc5ace67fe)

```c
# undef INIT_ARCH
# define INIT_ARCH() PREPARE_VERSION_KNOWN (linux26, LINUX_2_6)
/* If the vDSO is not available we fall back to syscall.  */
libc_ifunc_hidden (__gettimeofday_type, __gettimeofday,
                   (_dl_vdso_vsym ("__vdso_gettimeofday", &linux26)
                    ?: &__gettimeofday_syscall))
libc_hidden_def (__gettimeofday)

weak_alias (__gettimeofday, gettimeofday)
libc_hidden_weak (gettimeofday)
```

Macro `libc_ifunc_hidden` is defined in file `include/libc-symbols.h` as the
following. In case of that GCC supports attribute `ifunc`, it will be used 
for implmenting indirect functions, otherwise, use the old behavior, 
`%gnu_indirect_function`.

```c
/* Helper / base  macros for indirect function symbols.  */
#define __ifunc_resolver(type_name, name, expr, arg, init, classifier)  \
  classifier inhibit_stack_protector void *name##_ifunc (arg)           \
  {                                                                     \
    init ();                                                            \
    __typeof (type_name) *res = expr;                                   \
    return res;                                                         \
  }

#ifdef HAVE_GCC_IFUNC
# define __ifunc(type_name, name, expr, arg, init)                      \
  extern __typeof (type_name) name __attribute__                        \
                              ((ifunc (#name "_ifunc")));               \
  __ifunc_resolver (type_name, name, expr, arg, init, static)

# define __ifunc_hidden(type_name, name, expr, arg, init)       \
  __ifunc (type_name, name, expr, arg, init)
#else
/* Gcc does not support __attribute__ ((ifunc (...))).  Use the old behaviour
   as fallback.  But keep in mind that the debug information for the ifunc
   resolver functions is not correct.  It contains the ifunc'ed function as
   DW_AT_linkage_name.  E.g. lldb uses this field and an inferior function
   call of the ifunc'ed function will fail due to "no matching function for
   call to ..." because the ifunc'ed function and the resolver function have
   different signatures.  (Gcc support is disabled at least on a ppc64le
   Ubuntu 14.04 system.)  */
# define __ifunc(type_name, name, expr, arg, init)                      \
  extern __typeof (type_name) name;                                     \
  void *name##_ifunc (arg) __asm__ (#name);                             \
  __ifunc_resolver (type_name, name, expr, arg, init,)                  \
 __asm__ (".type " #name ", %gnu_indirect_function");

# define __ifunc_hidden(type_name, name, expr, arg, init)               \
  extern __typeof (type_name) __libc_##name;                            \
  __ifunc (type_name, __libc_##name, expr, arg, init)                   \
  strong_alias (__libc_##name, name);
#endif /* !HAVE_GCC_IFUNC  */

#define libc_ifunc_hidden(redirected_name, name, expr)                  \
  __ifunc_hidden (redirected_name, name, expr, void, INIT_ARCH)
```

## ELF and x86/x86_64 ABIs

In Linux, the `execve()` system call is used to laod and execute a stored 
program in the current process. To accommodate this, the stored program must
convey information to the kernel regarding how the associated code should be
organized in memory, among other bits of information. This information is
communicated through the various pieces of the executable file format, which
the kernel must thus parse and handle accordingly.

Linux supports parsing executing mulitple file formats. Internally, Linux 
abstracts the implementations for each of these file format handlers via the
`linux_binfmt` descriptor. Each handler is responsible for parsing the 
associated file format, creating the necessary state, and then starting the
execution. For example, the reason that it's possible to invoke `execve()` on 
shell scripts beginning with the (hash bang/shebang) line `#! <prog> <args>` is
because there is an explicit handler, `binfmt_script`, dedicated to parsing 
files with that prefix, executing the program, and then passing the script to
that program. That recursion almost always ends with the invocation of an ELF
binary program.

These `linux_binfmt` handler implementations, along with common `execve()`, can
be found in `${KSRC}/fs/binfmt_*.c` and `${KSRC}/fs/exec.c`

While Linux supports multiple executable file formats, the most commonly used 
format on Linux is the Executable and Linkable Format (ELF). The ELF standard
was published by the UNIX System Laboratories, and was intended to provid a
single executable format that would "extend across multiple operating 
environments" (UNIX System Laboratories, 2001).

ELF defines three different object types: **relocatable files**, **executable files**, and **shared objects**. Relocatable files are designed to be combined
with other object files by the linker to produce executable or shared object 
files. Executable files are designed to be invoked via `execve()`. Shared 
object files are designed to be linked, either by the linker at build time, or 
at load time by the dynamic linker. Shared object files allow for code to be 
referenced via shared libraries.

The ELF format is comprised of two complementary views.

**The first view, linking view**, is comprised of the program sections.
Sections group various aspects such as code and data into congiuous regions 
within the the ELF file. The linker is responsible for concatenating duplicate 
sections when linking multiple object files. For example, if object file 
`foo.o` defines a funcion `foo()` and a variable `x`, and object file `bar.o` 
defines a function `bar()` and a variable `y`, the linker would produce an 
object file whose code section contained functions `foo()` and `bar()` and 
whose data section contained variables `x` and `y`.

The ELF specification defines special sections that are allocated for specific
tasks, such as holding code or read-only data. These special sections can be
identified by leading dot in their name. Developers are also free to define 
their own custom sections.

A sampling of some common special sections and their usage is describled below.

- `.text` Holds executable instructions.

- `.bss` Holds uninitialized, i.e. initialized to zero, data. Since all of the
  variables in this section have the same value, zero, they aren't physically
  stored in the file. When the file is loaded, the corresponding memory region
  is zeroed.

- `.data` Holds initialized data.

- `.rodata` Holds read-only data.

- `.strtab` A string table containing a NULL-terminated string of each symbol
  or section name. The first and last entries are NULL bytes.

- `.symtab` A symbol table containing entries for locating symbols. Includes 
  the symbol name, as an index into `.strtab` sections, the symbol type, the
  symbol size, etc.

In GAS, the GNU assembler, the current section is selected via `.section`
pseudo-op. Some popular special sections can also be selected through their
corresponding pseudo-ops, e.g., `.data`, or `.text`, which switches to the
`.data` or `.text` sections, respectively. 

**The second view, execution view**, is comprised of segments. Unlike sections,
which represent the file layout, segments represent the virtual memory segments
during execution. These segments are described in the Program Header table, 
which contains the information needed by the kernel to struct the code in 
memory. This table consists of the Load Memory Address (LMA) and Virtual Memory
Address (VMA) for each memory segments, along with other information, such as
the alignment requirements and size in memory. To accommodate on-demand loading
of virtual memory segments, the ELF format stipulates that a segment's virtual
address and file offset must be congruent modulo the page size.

### Relocations and PIC

One important aspect of building and loading an executable object is the 
handling of external symbols, i.e., symbols exposed in other ELF objects. 
There are two differnt techniques for dealing with external symbols. The first
technique, static linking, occurs completely at link time while the seconds
technique, dynamic linking, occurs partially at link time and partially at load
time.

Statically linking an executable resolves all external symbols at link time. To
accomplish this, the external dependencies are explicitly copied from their ELF
objects into the executable. In other words, the executable includes all of the
user space code needed for it to run. While this alleviates the need for 
resolving symbols at load time, it comes at the cost of increased executable 
size, memory usage, and update effort.

The increased executable size is caused by the inclusion of the program's 
dependencies. For instance, if an executable relies on functions from the C
runtime, such as `printf`, the full `printf` implementation from libc must be
copied into the executable. The exact size increase depends, since modern 
linkers only copy the needed bits, ensuring that the code size only pays for 
what the software uses.

The increased memory usage is caused by the redundant copies of functions 
between executables. For instance in the previous example, each statically 
linked that uses `printf` must have its own copy of `printf` implementation,
whereas otherwise all of the executables can share one implementation. Some
people might argue that static linking actually reduces memory usage, since
unused objects from libraries aren't loaded into the process image. While it's
true that with dynamic linking the full library, including unused objects, will
be mapped into virtual address space of the process, that doesn't necessarily
translate to that full library being loaded into physical memory. Each page of 
the library will only loaded from disk into memory once it has been accessed, 
and once these pages have been loaded they can be shared between multiple 
executables.

The increased update effort is caused by the fact that the only way to update 
the external dependencies used by an executable is to recompile it. This also
makes it more difficult to determine what version of a library a specific
executable is using. This can obviously be a serious issue when determine 
whether an executable is susceptible to a published security vulnerability in 
one of its dependencies. With dynamic linking, only the shared library needs to
be recompiled.

Unlike with static linking, dynamic linking doesn't fully resolve the location 
of external dependencies until runtime. At link time, the linker creates 
special sections, describing the external dependencies that need to be 
resolved at runtime. When the object is loaded, this information is used by the
dynamic linker to finish resolving the external symbols enumerated within the 
object. This process is referred to as symbol *binding*. Once this is complete,
control is transferred to the executable.

The linker creates an `PT_INTERP` entry into the object's Program Header. This
entry contains a NULL-terminated filesystem path to the dynamic linker, 
normally `ld.so` or `ld-linux.so`. This is the dyanmic linker that will be 
invoked to load the necessary symbols.

The linker also creates a `.dynamic` section that includes a 
sentinel-entry-terminated arrary of dynamic properties. Each of these entries 
is composed of a type and value. A full list of supported types and their 
subsequent meanings can be found within the ELF specification. A few of the 
important types are described below.

- `DT_NEEDED` Array elements marked with this type represent the names of 
  shared libraries required for external symbol resolution. The names can 
  either be the "SONAME" or the filesystem path, relative or absolute. The
  value associated with this type is an offset into the file's string table.
  A list of values of this type can be obtained either by reading the section,
  with a command like `readelf -d`, or at execution time via setting the 
  `LD_TRACE_LOADED_OBJECTS` environment variable. For convenience, `ldd` is an
  utility that will set the environment variable and execute the program.

- `DT_SONAME` The shared object name of the file. The value associated with 
  this type is an offset into the file's string table. This value is set via 
  the linker with `-soname=LDFLAG`.

- `DT_RPATH` The library search path. The value associated with this type is an
  offset into the file's string table. This value is set via the linker with
  `-rpath=LDFLAG`.

- `DT_HASH` The location of the symbol hash table. The value associated with 
  this type is the table address.

- `DT_TEXTREL` Whether the relocations will update read-only code.

Each of the entries in the `.dynamic` section marded `DT_NEEDED` are loaded by 
the dynamic linker and then searched for the necessary symbols. At link time, 
the linker creates a `.hash` section, which consists of a hash table of the 
symbol names exported within the current file. This hash table is used to 
accelerate the search process at runtime.

Unlike executables, which are always loaded at a fixed address known as build 
time, shared objects can be loaded at any address. Otherwise, it would be 
possible for conflicts to arise where two objects expect to be loaded at the 
same address.

In order to handle this, the symbols in shared objects must be *relocatable*. 
The ELF format supports many differnt types of relocations; a full description 
of each kind can be found within ELF specification. While the programmer isn't
required to manually handle these relocations, it's necessary to understand 
what happens behind the scenes, because it impacts both performance and 
security. 

When building relocatable code, the linker uses a dummy base address, 
typically zero, with each symbol value set to the appropriate offset from the
base address of the symbol's section. Each time one of these addrsses is used
in the code, the linker creates an entry in that section's corresponding 
relocation section. This entry contains the relocation type, that determines 
how the real address should be calculated, and the location of the code that
needs to be updated with real address. The relocation section takes the form of
`.rel.section_name` or `.rela.section_name`, where `section_name` is the 
section containing that relocation.

At runtime, the dynamic linker iterates the relocation section. For each 
entry, the real address is calculated and then corresponding code is patched 
with the resulting address. This has three important ramifications. First, a 
significant number of relocations can hurt application load performance. 
Second, the code pages in memory must be writable, since the dynamic linker 
needs to update them with the relocations, and thus security is reduced. Third,
since the code pages are modifies by the relocations, the can't be shared 
between processes, since each process will have different addresses, and thus
this leads to increased memory usage.

Obviously binding a large number of symbols before execution begins can be 
devastating to the application's load time. Even worse, there is no guarantee
that all of these resolved symbols will actually be needed during execution. In
order to alleviate this issue, ELF supports **lazy binding**, that is, where 
symbols are resolved the first time they are actually used. To accommodate 
this, function calls occur indirectly through the **Procedure Linkage Table** (
**PLT**) and **Gobal Offset Table** (**GOT**). The GOT contains the calculated 
addresses of the relocated symbols, whilie PLT is used as a *trampoline* to 
that address for function function calls.

Each entry in the PLT, except the first, corresponds to a special function. 
Rather than directly calling the relocated symbol, the code jumps to the 
function's PLT entry. The first instruction in a function's PTL entry jumps
to the address has been bound, the GOT address points back to the next 
instruction in the PLT entry, that is, the next instruction after the PLT's
first jump instruction. This next instruction pushes the symbol's relocation
offset onto the stack. Then the next instruction jumps to the first PLT entry,
which invokes the dynamic linker to resolve the relocation offset pushed onto
the stack. Once the dynamic linker has calculated the relocated address, it
updates the relevant entry in the GOT and them jumps to the relocated function.

For future invocations of that function, the code will still jump to the 
function's PLT entry. The first instruction in the PLT will jump to the address
stored in the GOT, which will now point to the relocated address. In other 
words, the cost of binding the symbols occurs the first time the function is 
used. After that, the only additional cost is the indirect jump into the PLT.

Another problem with the dynamic linker patching code at runtime is that the 
code pages must be writable and dirtied, that is, modified. Writable code pages
are a security rick, since those pages also marked as executable. Additionally,
dirty pages can't be shared between multiple processes and must be committed 
back to disk when swapped out. To solve this problem, the relocations need to
be shifted from the code pages to the data pages, since each process will 
require its own copy of those anyway. This is the key insight of Position 
Independent code (PIC).

The PLT is never modified by relocations, so it's marked read-only and shared
between processes. On the other hand, each process will require its own GOT,
containing the specific relocations for that process. In PIC, the PLT can only
indirectly access the GOT.

The method for addressing the GOT in the PLT works slightly differently between
the 32- and 64-bit ELF formats. In both specifications, the address of the GOT
is encoded relative to the instruction pointer. For 64-bit PIC, access to the 
GOT is encoded as an offset relative to the current instruction pointer. This
straightforward approach is possible because of *RIP-relative addressing*, 
which is only available in 64-bit mode. For 32-bit PIC, ELF ABI reserves the 
*EBX* register for holding the base address of the GOT.

Since the instruction pointer isn't directly accessible in 32-bit mode, a 
special trick must be employed. The `CALL` instruction pushes the address of 
the next instruction, the first instruction after the function returns, onto 
the stack so that `RET` instruction can resume execution ther after the 
function call is complete. Leveraging this, a simple function can read that 
saved address from the stack, and then return. In the GNU libc implementation,
this type of function is typically called `__i686.get_pc_thunk.bx`, which loads
the program counter, i.e., the instruction pointer, into the *EBX* register.
Once the instruction pointer is loaded into *EBX*, the offset of the GOT is 
added.

## Quesetions

- What is IFUNC? (and the problems it wants to revolve)

- Does it populate PLT?

- What are the cases which are suitable for using IFUNC?

- How is it used when implementing some syscalls?

- How does Glibc use `elf_ifunc_invoke` to implement runtime dispatching?

## References

### Processing ELF

- [How programs get run: ELF binaries](https://lwn.net/Articles/631631/)

### IFUNC

- [Gnu support for CPU dispatching](http://www.agner.org/optimize/blog/read.php?i=167)
- [glibc's use of gnu_indirect_function feature](https://sourceware.org/ml/libc-help/2011-08/msg00006.html)
- [12.3.2 Runtime Dispatching via IFUNC ELF Extension](https://books.google.com/books?id=X-WcBAAAQBAJ&pg=PA226&lpg=PA226&dq=glibc+indirect+function&source=bl&ots=FmF2LrOuRw&sig=IsQDYJjVVjlGJonCdyC9nyacGJQ&hl=en&sa=X&ved=0ahUKEwjc6smHx-rXAhUU02MKHfOwB4o4ChDoAQhNMAg#v=onepage&q=glibc%20indirect%20function&f=false)
- [Using GNU indirect functions](https://willnewton.name/uncategorized/using-gnu-indirect-functions/)

### MFV

- [Function multi-versioning in GCC 6](https://lwn.net/Articles/691932/)

### GLIBC Tunables

- [Tunables story continued - glibc 2.26](https://siddhesh.in/posts/tunables-story-continued-glibc-2-26.html)

### GLIBC Categories

- [https://siddhesh.in/categories/glibc.html](https://siddhesh.in/categories/glibc.html)

### Books

- [Jim Kukunas-2015, Power and Performance](https://www.amazon.com/Power-Performance-Software-Analysis-Optimization-ebook/dp/B00WZ1AX6S)

[bask](../)
