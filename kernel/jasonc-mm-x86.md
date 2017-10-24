---
layout : default
---

# Notes on x86_64 Linux Memory Management Part 1: Memory Addressing

## x86 System Architecture Operating Modes and Features

From chapter 2 of Intel SDM manual volume 3. IA-32 architecture (beginning
with Intel386 processor family) provides extensive support for operating
system and system-development software. This support offers multiple modes of
operation, which include Read mode, protected mode, virtual 8086 mode, and
system management mode. These are sometimes referred to legacy modes.

Intel 64 architecture supports almost all the system programming facilities
available in IA-32 architecture and extends them to a new operating mode (
IA-32e mode) that supports a 64-bit programming environment. IA-32e mode allows
software to operate in one of two sub-modes:

- 64-bit mode supports 64-bit OS and 64-bit applications
- Compatibility mode allows most legacy software to run; it co-exist with
  64-bit applications under 64-bit OS.

The IA-32 system-level architecutre includes features to assist in the
following operatins:

- Memory management
- Protection of software modules
- Multitasking
- Exception and interrupt handling
- Multiprocessing
- Cache Management
- Hardware resource and power management
- Debugging and performance monitoring

All Intel 64 and IA-32 processors enter real-address mode following a power-up
or reset. Software then initiates the swith from real-address mode to
protected mode. If IA-32e mode operations is desired, software also initiates
a switch from protected mode to IA-32e mode. (see Chapter 9, "Processor
Management and Initialization", SDM-vol-3).

## Memory Addressing

### Memory Address

Three kinds of addresses:

- **Logical address**, included in the machine language instructions to specify the address of an operand or of an instruction. Each logical address consists of a *segment* and *offset* (or *displacement*) that denotes the distance from the start of the segment to the actual address.

- **Linear address** (also known as **virtual address**), a signal 32/64-bit
unsigned integer that can be used to address whole address space.

- **Physical address**, used to address memory cells in memory chips. They
correspond to the electrical signals along the address pins of the
microprocessor to the memory bus.

The Memory Management Unit (MMU) transforms a logical address into a linear
address by means of a hardware circuit called a *segmentation unit*;
subsequently, a second hardware circuit called a *paging unit* transforms the
linear address into physical address. ("L->L->P" Model)

### Segmentation in Hardewre

#### Segment Selector and Segmentation Registers

A logical address consists of two parts: a segment identifier and an offset.
The segment identifier is a 16-bit field called the *Segment Selector*, while
the offset is a 32-bit field (64-bit field in IA-32e mode?).

To make it easy to retrieve segment selectors quickly, the processor provides
*segmentation registers* whose only purpose is to hold Segment Selectors; these
registers are cs, ds, es, ss, fs, gs. Although there are only six of them, a
program can resue the same segmentation register for different purpose by
saving its content in memory and them restoring it later.

The cs register has another important function: it includes a 2-bit field that
specifies Current Privilege Level (CPL) of the CPU. The value 0 denotes the
highest privilege level (a.k.a. Ring 0), while the value 3 denotes the lowest
one (a.k.a. Ring 3). Linux uses only levels 0 and 3, which are respectively
called **Kernel Mode** and **User Mode**.

#### Segmentation Descriptor

Each segment is represented by an 8-byte **Segment Descriptor** that describes
the segment characteristics. Segment Descriptors are stored either in the
**Global Descriptor Table (GDT)** or **Local Descriptor Table (LDT)**.

There are several types of segments, and thus serveral types of Segment
Descriptors. The following list shows the types that are widely used in Linux.

*Code Segment Descriptor*
>    Indicates that the Segment Descriptor refers to a code segment; it may be
>    included in either GDT or LDT. The descriptor has the S flag (non-system
>    segment).

*Data Segment Descriptor*
>    Indicates that the Segment Descriptor refers to a data segment; it may be
>    included in either GDT or LDT. It has S flag set. Stack segments are
>    implemented by means of generic data segments.

*Task State Segment Descriptor (TSSD)*
>    Indicates that the segment Descriptor refers to a Task State Segment
>    (TSS), that is, a segment used to save the contents of the processor
>    registers; it can appear only in GDT. The S flag is set to 0.

*Local Descriptor Table Descriptor (LDTD)*
>    Indicates that the Segment Descriptor refers to a segment containing an
>    LDT; it can appear only in GDT. The S flag of such descriptor is set to 0.

#### Fast Access to Segment Descriptors

To speed up the translation of logical addresses into linear addresses, x86
processor provides an additional nonprogrammable register for each of the six
segmentation register. Each nonprogrammable register (a.k.a *"shadow register"*
or *"descriptor cache"*) contains 8-byte Segment Descriptor specified by the
Segment Selector contained in the corresponding segmentation register. Every
time a Segment Selector is loaded in a segmentation register, the corresponding
Segment Descriptor is loaded from memory into the CPU shadow register. From
then on, translating of logical addresses referring to that segment can be
performed without accessing the GDT or LDT stored in main memory.

#### Segmentation Unit

Logical address is `<selector = { Index : TI }> : <offset>`

TI is to determine whether to get base linear address from *gdtr* or *ldrt*.

Linear address = `{ (Index * 8 + *gdtr/ldtr).base + offset }`

#### Segmentation in IA-32e Mode (32584-sdm-vol3: 3.2.4)

In IA-32e mode of Intel 64 architecture, the effects of segmentation depend on
whether the processor is running in compatibility mode or 64-bit mode. In
compatibility mode, segmentation functions just as it does using legacy 16-bit
or 32-bit protected mode sematics.

In 64-bit mode, segmentation is generally (but not completely) disabled,
creating a flat 64-bit linear-address space. The processor treats the
segmentation base of CS, DS, ES, SS as zero, creating a linear address that is
equal to the effective address. The FS and GS segments are exceptions. These
segment registers (which hold segment base) can be used as additional base
registers in linear address calculations. They facilitate addressing local
data and certain operating system data structures.

Note that the processor does not perform segment limit checks at runtime in
64-bit mode.

##### Segment Selector

A segment seletor is 16-bit identifier for a segment.

```
15     3  2 1 0
+-------+--+---+  Index: Select one of the 8192 descriptors in the GDT or LDT.
| Index |TI|RPL|  Table Indicator (TI) :{0=GDT, 1=LDT};
+-------+--+---+  Requested Privilege Level (RPL)
```

Segment selectors are visible to application programs as part of a pointer variable,
but the values of selectors are usually assigned or modified by link editors or
linking loaders, not application programs.

#### Segment Descriptor Tables in IA-32e Mode

In IA-32e mode, a segment descriptor table can contain up to 8192 (2^13) 8-byte
descriptors. An entry in the segment descriptor table can be 8 bytes. System
segment descriptors are expaned to 16 bytes (occupying the space of the two
entries).

GDTR and LDTR registers are expanded to hold 64-bit base addres. The corresponding pseudo-descriptor is 80 bits (64 bits "Base Address" + 16 bits "Limit").

The following system descriptors expand to 16 bytes:

- Call gate descriptor (see 5.8.3.1, "IA-32e Mode Call Gates")
- IDT gate descriptor (see 6.14.1, "64-Bit Mode IDT")
- LDT and TSS descriptors (see 7.2.3, "TSS Descriptor in 64-bit mode")

### Segmentation in Linux

Linux prefers paging to segmentation for the following reasons:

- Memory management is simpler when all processes use the same segment
  register value, that is, when the share the same set of linear addresses.

- One of the design objectives of Linux is portability to a wide range of
  architecture. RISC architectures in particular have limited support for
  segmentation.

In Linux, all base addresses of user mode code/data segments and kernel code/data segments are set to 0.

The corresponding Segment Selectors are defined by the macros `__USER_CS`,
`__USER_DS`, `__KERNEL_CS`, and `__KERNEL_DS`, respectively. To address the
kernel core segment, for instance, the kernel just loads the value yielded by
the `__KERNEL_CS` macro into the cs segmentation register.

The *Current Privilege Level* of the CPU indicates whether the process is in
User Mode or Kernel Mode and is specified by the *RPL* field of the Segment
Selector stored in the *cs* register.

When saving a pointer to an instruction or to a data structure, the kernel
does not need to store the Segment Selector component of the logical address,
because the *ss* register contains the Segment Selector. As an example, when
the kernel invokes a function, it executes a `call` assembly language
instruction specifiying just the *Offset* component of its logical address;
the Segment Selector is implicitly selected as the one referred to by the *cs*
register. Because there is just one segment of type "executable in Kernel
Mode", namely the code segment identified by `__KERNEL_CS`, it sufficient to
load `__KERNEL_CS` into *cs* whenever the CPU switched to Kernel Mode. The
same argument goes for pointers to kernel data structures (implicitly using
the *ds* register), as well as for pointers to user data structures (the
kernel explicitly uses the *es* register).

#### Linux GDT

In uniprocessor systems there is only one GDT, while in multiprocessor systems
there is one GDT for every CPU in the system.

`init_per_cpu_gdt_page` if `CONFIG_X86_64_SMP` is set or `gtd_page` otherwise.

see label `early_gdt_descr` and `early_gdt_descr_base` in `head_64.S`.

- Four user and kernel code and data segments

- Task State Segment (TSS), this is different from each processor in the
system. The linear space corresponding to a TSS is a small subset of the
linear address space corresponding to the kernel data segment. The Task
State Segments are sequentially stored in the per-CPU `init_tss` array on
SMP system. (it's `cpu_tss` since 4.x kernel). See how Linux uses TSSs in
chapter 3 in ULK3, and `__switch_to` in the source.

- A segment including the default Local Descriptor Table (LDT), usually shared
by all processes.

- Three *Thread Local Storage (TLS)* segments. see `set_thread_area(2)` and
`get_thread_area(2)`.

- Three segments related to Advanced Power Management (APM).

- Five segments related to Plug and Play (PnP) BIOS services.

- A special TSS segment used by kernel to handle "Double fault" execptions. (see Chapter 4 in ULK3).

### Paging in Hardware

The paging unit translates linear addresses into physical ones. One key task in the unit is to check the requested access type against the access rights of the linear address. If the memory access is not valid, it generates a **Page Fault** exceptions (e.g. COW, SIGSEGV, SIGBUS?, Swap Cache/area).

For the sake of efficiency, linear addresses are grouped in fixed-length intervals called *pages*; contiguous linear addresses within a page are mapped into contigous physical addresses. We use term *page* to refer both to a set of addresses and to the data contained in this group of addresses.

The paging unit thinks of all RAM as partitioned into fixed-length *page fromes* (a.k.a *physical pages*). Each page frames contains a page, that is, the length of a page frame coincides with that of a page. A page frame is a constituent of main memory, and hence is a storage area. *It is important to distinguish a page from a page frame; the former is just a block of data, which may be stored in any page frame or on disk*.

The data structure that maps linear to physical addresses are called *page tables*; they are stored in main memroy and must be properly initialized by the kernel before enabling paging unit.

Paging unit is enabled by setting the *PG* flag of a control register named *cr0*. When `PG = 0` (disabled), linear addresses are interpreted as physical addresses.

#### Regular Paging

Fields in 32bit linear address space:

    - 2 level page tables
    - Directory (10 bits), Table (10 bits), Offset (12 bits)
    -

Fields in 64bit linear address space:

    - 4 level page tables
    - ...

The entries of Page Directories and Page Tables have the same structure. Each entry includes the following fields:

- **Present flag**
- **Field containing the 20 most significant bits of a page frame physical address**
- **Accessed flag**: set each time the paging unit addresses the corresponding page frame. (e.g. PFRA)
- **Dirty flag**
- **Read/Write flag**: contains the access right (Read/Write or Read) of page or of the page table.
- **User/Supervisor flag**: contains privilege level required to access the page or of the page table.
- **PCD and PWT flag**: contains the way the page or Page Table is handled by the hardware cache.
    - *PCD*: page-level cache disable
    - *PWT*: page-level write-through
- **Page Size flag**: applies only to Page Directory entries. If it's set, the entry refers to a 2MB, or 4MB long page frame.
- **Global flag**: applies only to Page Table entries. This flag as introduced to prevent frequently used pages from being flushed from *TLB cache*. It works only if the Page Global Enable (PGE) flag of register *CR4* is set.


#### Extended/PAE Paging (Large Pages)

PAE stands for Physical Address Extension Paging Mechanism

#### Paging for 64-bit Architecture (x86_64)

| Page size                   | 4KB                |
| --------------------------- | ------------------ |
| Number of address bits used | 48                 |
| Number of paging leveles    | 4                  |
| Linear address splitting    | 9 + 9 + 9 + 9 + 12 |

#### Hardware Cache

Hardware cache memory were introduced to reduce the speed mismatch between CPU and DRAM. They are based on the well-known *locality principle*, which holds both for program and data structure. For this purpose, a new unit called *line* was introduced to the architecture. It consists of a few dozen contigous bytes that are transfered in burst mode between the slow DRAM and the fast on-chip static RAM (SRAM) used to implement caches.

The cache is subdivided into subsets of *lines*. And most caches are to some degree N-way set associative. where any line of main memory can be stored in any one of N lines of the cache.

The bits of the memory's physical address are usually split into three groups: the most significant ones correspond to the **tag**, the middle ones to the cache controller **subset index**, and the least significant ones to the **offset** within the *line*.

physical address ==> {Tag:Index:Offset}

Linux clears the *PCD* and *PWT* flags of all Page Diectory and Page Table entries; as a result, caching is enabled for all page frames, and the write-back strategy is always adopted for writing.

#### Translation Lookaside Buffers (TLB)

Besides general-purpose hardware caches, x86 processor include another cache called *Translation Lookaside Buffers (TLB)* to speed up linear address translation. When a linear address is used for the first time, the corresponding physical address is computed through slow accesses to the Page Table in RAM. The physical address is then stored in a TLB entry so that further references to the same linear address can be quickly translated.

In a multiprocessor system, each CPU has its own TLB, called the *local TLB* of the CPU.

When the *cr3* control register of a CPU is modified, the hardware automatically invalidates all entries of the local TLB, because a new set of page tables is in use and the TLBs are pointing to the old data.

### Paging in Linux

Linux adopts a common paging model that fits both 32-bit and 64-bit architectures. Starting with version 2.6.11, a four-level paging model has been adopted.

- Page Global Directory
- Page Upper Directory
- Page Middel Directory
- Page Table

(Regular) Linear address:

```
         9             9          9            9        12
    [ Global DIR | Upper DIR | Middle DIR |  Table  | Offset ]
         |             |          |            |        |
cr3  +   v     +       v     +    v     +      v    +   v ==> physical address
                                                    |
                                                    `==> page frame
```

Each process has its own Page Global Directory and its own set of Page Tables.
When a process switch occurs, the *cr3* is switched accordingly. Thus, when the new process resumes its execution on the CPU, the paging unit refers to the correct set of Page Tables.


Starting with v4.12, Intel's 5-level paging support is enabled if `CONFIG_X86_5LEVEL` is set. See the following links to find more.

- [Intel's 5-Level Paging Support Being Prepped For Linux 4.12](https://www.phoronix.com/scan.php?page=news_item&px=Intel-5-LVL-Paging-Linux-4.12)
- [x86: 5-level paging enabling for v4.12, Part 3](http://lkml.iu.edu/hypermail/linux/kernel/1703.3/01979.html)
- [5-Level Paging and 5-Level EPT](https://software.intel.com/sites/default/files/managed/2b/80/5-level_paging_white_paper.pdf)

#### The Linear Address Fields

Macros that simplify Page Table handling.

- `PAGE_SHIFT`
- `PMD_SHIFT`
- `PUD_SHIFT`
- `PGDIR_SHIFT`
- `PTRS_PER_PTE, PTRS_PER_PMD, PTRS_PER_PUD, PTRS_PER_PGD`

#### Page Table Handling

`(pte|pmd|pud|pgd)_t` describe the format of, respectively, a Page Table, a Page Middle Directoy, a Page Upper Directoy, and a Page Global Directoy entry. They are 64-bit data teyps when PAE is enabled and 32-bit data types otherwise. (PS: They are the same type `unsigned long` on x86_64)

Five type-conversion macros cast an unsigned integer into the required type:
`__(pte|pmd|pud|pgd)`. Five other type-conversion macros do the revser casting: `(pte|pmd|pud|pgd)_val`.

Kernel also provides several macros and functions to read or modify page tables
entries.

```
- (pte|pmd|pud|pgd)_none
- (pte|pmd|pud|pgd)_clear, ptep_get_and_clear()
- set_(pte|pmd|pud|pgd)
- pte_same(a,b)
- pmd_large(e) return 1 if the Page Middle Directoy entry e refers to a large 
  page (2MB or 4MB), 0 otherwise.
- pmd_bad yields 1 if the entry points to a bad Page Table, that is at least
  one of the following conditions applies: (see `_KERNPG_TABLE`)
        - The page is not in main memory (*Present* flag cleared).
        - The page allows only Read access (*Read/Write* flags cleared).
        - Either *Accessed* or *Dirty* is cleared (Linu always forces these
          flags to be set for every Page Table).
- (pud|pgd)_bad
- (pte|pmd|pud|pgd)_present
```

**Page flag reading functions**:

`pte_(user|read|write|exec|dirty|young|file)`

**Page flag setting functions**:

`pte_(mkhuge|clrhuge|wrprotect|rdprotect|mkread|mkwrite|mkexec|mkdirty|...)`

**Page allocation functions**:

```c
- pgd_alloc(mm), pgd_free(pgd)
- pud_alloc(mm, pgd, addr), pud_free(x)
- pmd_alloc(mm, pud, addr), pmd_free(x)
- pte_alloc_map|kernel(mm, pmd, addr), pte_free(pte)
- pte_free(pte), pte_free_kernel(pte)
- clear_page_range(mmu, start, end)
```

The `index` field of the page descriptor used as a pgd page is pointing to the
corresponding memory descriptor `mm_struct`. (see `pgd_set_mm`)

Global variable `pgd_list` holds the doubly linked list of pgd(s) by linking `page->lru` field of that pgd page. (see `pgd_list_add`)

Common kernel code path: `execve()`

```
do_execve_common @fs/exec.c
    bprm_mm_init
        mm_alloc @kernel/fork.c
            allocate_mm
            mm_init
                mm_alloc_pgd
                    pgd_alloc @arch/x86/mm/pgtable.c
                        // 1. allocates a new page via zone allocator
                        // 2. stores the new pgd page in the corresponding
                        //    memory descriptor
                        // 3. requires pgd_lock spinlock
                        // 5. invokes pgd_ctor to
                        //      a. copy/clone master kernel page tables
                        //      b. stores the corresonping memory descriptor in
                        //         the index field of the page descriptor of
                        //         this pgd page.
                        //      c. add pgd page to the global doubly linked
                        //         list pgd_list by linking page->lru.
```

#### Virtual Memory Layout

See Documentation/x86/x86_64/mm.txt in the kernel to find the latest
description of the memory map.

The following is grabbed from v3.2.

```
<previous description obsolete, deleted>

Virtual memory map with 4 level page tables:

0000000000000000 - 00007fffffffffff (=47 bits) user space, different per mm
hole caused by [48:63] sign extension
ffff800000000000 - ffff80ffffffffff (=40 bits) guard hole
ffff880000000000 - ffffc7ffffffffff (=64 TB) direct mapping of all phys. memory
ffffc80000000000 - ffffc8ffffffffff (=40 bits) hole
ffffc90000000000 - ffffe8ffffffffff (=45 bits) vmalloc/ioremap space
ffffe90000000000 - ffffe9ffffffffff (=40 bits) hole
ffffea0000000000 - ffffeaffffffffff (=40 bits) virtual memory map (1TB)
... unused hole ...
ffffffff80000000 - ffffffffa0000000 (=512 MB)  kernel text mapping, from phys 0
ffffffffa0000000 - fffffffffff00000 (=1536 MB) module mapping space

The direct mapping covers all memory in the system up to the highest
memory address (this means in some cases it can also include PCI memory
holes).

vmalloc space is lazily synchronized into the different PML4 pages of
the processes using the page fault handler, with init_level4_pgt as
reference.

Current X86-64 implementations only support 40 bits of address space,
but we support up to 46 bits. This expands into MBZ space in the page tables.

-Andi Kleen, Jul 2004
```

#### A little interesting `page_address()` implemenation on x86_64

A linear address can be calculated by `__va(PFN_PHYS(page_to_pfn(page)))`,
which is equivalent to: `(page - vmemmap) / 64 * 4096`. Where

- page is address of a page descriptor
- vmemmap is `0xffffea00_00000000`
- `64` is size of page descriptor, i.e. `sizeof(struct page)`
- `4096` is `PAGE_SIZE`.

As 2's complement subtraction is the binary addition of the minuend to the 2's
complement of the subtrahend (adding a negivative number is the same as
subtracting a positive one). The assembly code doing that calculation looks
like the following. Note that `0x1600_00000000` is the corresponding 2's
complement, and by "x86_64-abi-0.98", which requires compiler to use `movabs`
in this case.

```asm
    movabs $0x160000000000,%rax
    add    %rax,%rdx
    movabs $0xffff880000000000,%rax
    sar    $0x6,%rdx
    shl    $0xc,%rdx
    add    %rdx,%rax
```

(see `__get_free_pages`, `virt_to_page`, and `lowmem_page_address`).

#### Memory Model Kernel Configuration

By default, [SPARSEMEM](https://lwn.net/Articles/134804/) and [Sparse Virtual Memmap](https://lwn.net/Articles/229670/).

```
CONFIG_SPARSEMEM=y
CONFIG_SPARSEMEM_VMEMMAP=y
```

#### Physical Memory Layout

During the initialization phase the kernel must build a *physical address map*
that specifies which physical address ranges are usable by the kernel and which
are unavaiable.

The kernel considers the following page frames as *reserved*:

- Those falling in the unavaiable physical address ranges
- Those containing the kernel's code and initialization data structures.

A page contained in a reserved page frame can never be dynamically assigned 
or swapped out to disk.

In the early stage of the boot sequence (*TBD*), the kernel queries the BIOS
and learns the size of the physical memory. In recent computers, kernel also
invoks a BIOS procedure to build a list of physical address ranges and their
corresponding memory types.

Later, the kernel executes the `default_machine_specific_memory_setup`
function, which builds the physical addresses map. (see `setup_memory_map`,
`setup_arch`)

Variables describing the kernel's physical memory layout

- `max_pfn`, `max_low_pfn`
- `num_physpages`
- `high_memory`
- `_text`, `_etext`, `_edata`, `_end`

The BIOS-provides physical memory map is registered at `/sys/firmware/memmap`.

#### Process Page Table

>*TBD*

#### Kernel Page Table

The kernel maintains a set of page tables for its own use, rooted at a
so-called *master kernel Page Global Directory*. After system initialization,
this set of page tables is never directly used by any process or kernel
thread; rather, the highest entries of of the master kernel Page Global
Directory are the reference model for the corresponding entries of the Page
Global Directories of every regular process in the system.

##### Provisional Kernel Page Table

A provisional Page Global Directory is initialized statically during kernel
compilation, while it is initialized by the `startup_64` assembly language
function defined in `arch/x86/kernel/head_64.S`.

The provisional Page Global Directory is contained in the `swapper_pg_dir`
(`init_level4_pgt`).

##### Final Kernel Page Table

The master kernel Page Global Directory is still stored in `swapper_pg_dir`.
It is initialized by `kernel_physical_mapping_init` which is invoked by
`init_memory_mapping`.

After finalizing the page tables, `setup_arch` invokes `paging_init()` which
invokes `free_area_init_nodes` to initialise all `pg_data_t` and zone data.

The following is the corresponding call-graph using linux-3.2.

```
setup_arch                            // @arch/x86/kernel/setup.c
    |
    `init_gbpages
    |
    `init_memory_mapping              // @arch/x86/mm/init.c
    |   |
    |   `kernel_physical_mapping_init // @arch/x86/mm/init_64.c
    |   |   |
    |   |   `sync_global_pgds
    |   |   `__flush_tlb_all          // @arch/x86/include/asm/tlbflush.h
    |   |
    |   `__flush_tlb_all
    |
    `initmem_init                     // @arch/x86/mm/numa_64.c            
    |   |                             // CONFIG_NUMA=y
    |   `x86_numa_init                // @arch/x86/mm/numa.c
    |       |
    |       `numa_init(x86_acpi_numa_init) // CONFIG_ACPI_NUMA=y
    |       `numa_init(dummy_numa_init)    // Fallback dummy NUMA init.
    |           |
    |           `dummy_numa_init      // Faking a NUMA node (0) descritpors
    |           `numa_register_memblks
    |               |
    |               `setup_node_data  // Allocate and initialize *NODE_DATA*
    |                                 // for a node on the local memory.
    |
    `paging_init                      // @arch/x86/mm/init_64.c
        |
        `free_area_init_nodes         // @mm/page_alloc.c
            |
            `free_area_init_node
                |
                `free_area_init_core  // *Set up the zone data structures*
```

#### Fix-Mapped Linear Addresses

To associate a physical address with a fix-mapped linear address, the kernel
uses the following:

- `fix_to_virt(idx)`
- `set_fixmap(idx,phys)`, `set_fixmap_nocache(idx,phys)`
- `clear_fixmap(idx)`

#### Handling the Hardware Cache and the TLB

##### Handling the Hardware Cache

Hardware chaces are addressed by cache lines. The `L1_CACHE_BYTES` macro
yields the size of a cache line in bytes. On recently Intel models, it yields
value 64.

`CONFIG_X86_L1_CACHE_SHIFT=6`

Cache synchronization is performed automatically by the x86 microprocessors,
thus the Linux kernel for this kind of processor does not perform any hardware
cache flushing.

##### Handling the TLB

Processors can't synchronize their own TLB cache automatically because it is
the kernel, and not the hardware, that decides when a mapping between a linear
and a physical address is no longer valid.

**TLB flushing**: (see `arch/x86/include/asm/tlbflush.h`)

| func                               | desc                     |
| ---------------------------------- | ------------------------ |
| `flush_tlb()`                      | flushes the current mm struct TLBs |
| `flush_tlb_all()`                  | flushes all processes TLBs |
| `flush_tlb_mm(mm)`                 | flushes the specified mm context TLB's |
| `flush_tlb_page(vma, vmaddr)`      | flushes one page |
| `flush_tlb_range(vma, start, end)` | flushes a range of pages |
| `flush_tlb_kernel_range(start, end)` | flushes a range of kernel pages |
| `flush_tlb_others(cpumask, mm, va)` | flushes TLBs on other cpus |

x86-64 can only flush individual pages or full VMs. For a range flush
we always do the full VM. Might be worth trying if for a small range a
few `INVLPG`(s) in a row are a win.

**Avoiding Flushing TLB**:

As a general rule, any process switch implies changing the set of active page
tables. Local TLB entries relative to the old page talbes must be flushed;
this is done when kernel writes the address of the new PGD/PML4 into the *cr3*
control register. The kernel succeeds, however, in avoiding TLB flushes in the
following cases:

- When performing a process switch between two regular processes that uses the same set of page tables.
- When performing a process switch between a regular process and a kernel thread.

##### Lazy TLB Handling

The per-CPU variable `cpu_tlbstate` is used for implementing lazy TLB mode.
Furthermore, each memory descriptor includes a `cpu_vm_mask_var` field that
stores the indices of the CPUs that should receive *Interprocessor Interrupts*
related to TLB flushing. This field is meaningful only when the memory
descriptor belongs to a process currently in execution.

```
struct tlb_state {
        struct mm_struct *active_mm;
        int state;
};
DECLARE_PER_CPU_SHARED_ALIGNED(struct tlb_state, cpu_tlbstate);
```

(see `mm_cpumask`, `cpumask_set_cpu`, `swtich_mm`)

#### Referenced APIs and Source

##### Common in 3.2 and 4.4

| file                                | desc |
| ----------------------------------- | ----------- |
| `arch/x86/include/asm/page_types.h`    | common macros of x86 page table |
| `arch/x86/include/asm/pgtable_64_types.h`| macros of x86_64 4-level paging |
| `arch/x86/include/asm/pgtable.h` | page table handling |
| `arch/x86/include/asm/pgtable_types.h` | page table handling |
| `arch/x86/mm/pgtable.c` | Page allocation functions |
| `arch/x86/kernel/setup.c` | Architecture-specific boot-time initializations |
| `arch/x86/kernel/e820.c` | BIOS-provided memory map |

##### Linux 3.2

| file                                | desc |
| ----------------------------------- | ----------- |
| `arch/x86/include/asm/segment.h` | segment layout and definitions |
| `arch/x86/kernel/head_64.S`      | start in 32bit and switch to 64bit. |
| `arch/x86/kernel/init_task.c`    | initial task and per-CPU TSS segments |
| `arch/x86/kernel/process_64.c`   | process handling |
| `arch/x86/include/asm/processor.h` | x86 `tss_struct` and per-CPU `init_tss`|

##### Linux 4.4

| file                        | desc |
| --------------------------- | ----------- |
| `arch/x86/kernel/process.c` | per-CPU TSS segments. |

## References

- [What every programmer should know about memory, Part 1](https://lwn.net/Articles/250967/)
- [Memory part 2: CPU caches](https://lwn.net/Articles/252125/)
- [Memory part 3: Virtual Memory](https://lwn.net/Articles/253361/)

[back](../)


