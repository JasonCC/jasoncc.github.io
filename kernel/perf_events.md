# A Study of perf_events Subsystem

### `perf_events` Subsystem

HPC-based performance measurements can be conduct in two difference modes:

- Counting/Aggrigation, which is suitable for gathering statistics of a
  specific process or the entrie system in a particular time interval.
  In this mode, `perf` simply aggregates the occurrence of the events and
  either reports them at the end of the execution of the running process, or
  when the user sends a `SIGINT` signal to interrupt the target process.

- Sampling/Profiling, in which the results not onlly contain the number of
  total events count, but also has the program execution profile. Access to a
  program execution profile allows one to easily identify the time-consuming
  function(s)/hotspot(s) of the program that is the main goal of almost every
  performance analysis project.

#### `perf_event` Interface (Ways to Collection Data)

The `perf_event` subsystem only added one system call to the Linux kernel,
the `perf_event_open`. It returns a file descriptor to identify the configured
event(s). It manages the event(s) independently through file descriptor that
allows one to configure and count events with different configurations in a
single session.

There is no glibc wrapper for this system call. Therefore, it has be called
using the `syscall` function with the `__NR_perf_event_open` as the first
parameter.

see `perf_event_open(2)` man-page and `tools/perf/design.txt` in kernel source
to find more detail.

#### Counting Mode

In counting mode, the user can control the performance counter with the three
basic operations: reset, enalbe, and disable. These operations can be performed
by calling `ioctl` system call with the file descriptor previously returned by
`perf_event_open(2)`.

#### Sampling Mode

Unlike counting mode, `perf_event` configuration in sampling mode is more
complicated in both kernel and user levels. First of all, the perf_event
subsystem needs to configure the PMU (using `wrmsr` on x86) to overflow when
the hardware event count reaches to a specific value called "sampling period".
By invoking `perf_event_open(2)` in sample mode, the kernel would automatically
adjust the sampling period based on the frequency of hardware event generation
and the sampling frequency that provided via the `perf_event_attr.sample_freq`
attribute. After each PMU overflow interrupt, the kernel readjusts the sampling
period using the formula as long as we did not specify a fixed sampling period
at the time of creating event.

```
new_sampling_period = (last_sample_period * 10^9) / (elapsed_time * sample_freq)
```

After configuring an event for counting in sampling mode, we need to map the
file descriptor that returned from the `perf_event_open` to the user address
space. The following snippet shows how we can use `mmap` function to map the
file descriptor to the program address space that can provide a direct access
to the taken samples from the user space.

```c
char *mmap_addr = mmap(0, NR_PAGES*PAGE_SIZE, PROT_READ|PROT_WRITE,
                       MAP_SHARED, event_fd, 0);
```

`NR_PAGES` is the number of memory pages that is mapped from the kernel
ring-buffer.

Next step is to configure the user program that initiated the `perf_event_open`
to handle the *wakeup* signal/condition and save the captured samples after the
memory mapped pages fill up (or a certain threshold is reached). To be more
specific, a wakeup signal will be sent to the user program when either the
number of samples that are stored in the ring-buffer reaches the
`perf_event_attr.wakeup_events` value, or the ring-buffer consumes all of the
memory mapped pages. Wake up conditions can be captured by polling the file
descriptor using the `poll(2)` or `select(2)` over the `POLL_IN` event, or by
setting up a signal handler that handles the `SIGIO` signal using the `fcntl(2)`.
The Linux perf utility preferred to use the polling method rather than the signal
handling as it interrupts the execution flow of the program and makes it
exceedingly slow. Once the wakeup event is captured, the user-space program
retrieves the pointer that points the beginning of the buffer
(`perf_event_mmap_page->data_head`), and then starts to read the captured
samples form the next 2^n pages.

### `perf_event` Supported Events (Data Sources)

Events including:
1. Hardware
2. Software
3. Tracepoint
    - static tracepoint
    - dynamic trace
        + uprobe
        + kprobe
4. Hardware Cache
5. Raw
6. Breakpoint

For Intel x86-64, PMUs are:

| Name          | Type  | Feature |
| ------------- | ----- | ------- |
| cpu           | raw   | Intel x86 CPU |
| intel_bts     | -1    | Branch Tracing Store |
| intel_cqm     | -1    | Intel Cache Quality-of-Service Monitoring (CQM): LLC occupancy/Total MBM/Local MBM |
| cstate_core   | -1    | cstate residency counters, won't appear when running on a hypervisor (X86_FEATURE_HYPERVISOR is true) |
| cstate_pkg    | -1    | similiar to cstate_core | 
| intel_pt      | -1    | Intel Processor Trace |
| power         | -1    | Intel RAPL/Runtime Average Power Limit |
| uncore*       | -1    | Uncore events |

Type -1 means dynamic allocated type which is other than the following types:
(from 1 to 5)
- PERF_TYPE_HARDWARE
- PERF_TYPE_SOFTWARE
- PERF_TYPE_TRACEPOINT
- PERF_TYPE_HW_CACHE
- PERF_TYPE_RAW
- PERF_TYPE_BREAKPOINT

```
"Intel": static __initconst const struct x86_pmu intel_pmu

early_initcall(init_hw_perf_events)

x86 hardware PMU (`x86_pmu`) always uses NMI for PMI.
```

### How it Works Under the Hood? How Does perf_event Collect Those HW/SW Events?

#### Hardware Events

#### Hardware Cache

#### Hardware Raw

#### Software Events

#### Tracepoints

static 

dynamic: kprobe, uprobe

#### ftrace ?


#### eBPF ?


### Data Strucures

1. `struct pmu`: generic performance monitoring unit

It is defined in file `include/linux/perf_event.h`. Take the pmu defined for
x86 architecure as an example (`arch/x86/events/core.c`).

```
2216 static struct pmu pmu = {
   1         .pmu_enable             = x86_pmu_enable,
   2         .pmu_disable            = x86_pmu_disable,
   3
   4         .attr_groups            = x86_pmu_attr_groups,
   5
   6         .event_init             = x86_pmu_event_init,
   7
   8         .event_mapped           = x86_pmu_event_mapped,
   9         .event_unmapped         = x86_pmu_event_unmapped,
  10
  11         .add                    = x86_pmu_add,
  12         .del                    = x86_pmu_del,
  13         .start                  = x86_pmu_start,
  14         .stop                   = x86_pmu_stop,
  15         .read                   = x86_pmu_read,
  16
  17         .start_txn              = x86_pmu_start_txn,
  18         .cancel_txn             = x86_pmu_cancel_txn,
  19         .commit_txn             = x86_pmu_commit_txn,
  20
  21         .event_idx              = x86_pmu_event_idx,
  22         .sched_task             = x86_pmu_sched_task,
  23         .task_ctx_size          = sizeof(struct x86_perf_task_context),
  24 };
```

2. `struct perf_event`: performance event kernel representation

3. `struct x86_pmu`: x86 specific pmu

4. Syscall `sys_perf_event_open`: open a performance event, associate it to a task/cpu.

#### Public APIs

##### Core APIs

- Register a PMU device.

```c
int perf_pmu_register(struct pmu *pmu, const char *name, int type)
```

Use `grep -n EXPORT_SYMBOL $KSRC/events/core.c` to find them.

```c
EXPORT_SYMBOL_GPL(perf_event_disable);
EXPORT_SYMBOL_GPL(perf_event_enable);
EXPORT_SYMBOL_GPL(perf_event_addr_filters_sync);
EXPORT_SYMBOL_GPL(perf_event_refresh);
EXPORT_SYMBOL_GPL(perf_event_release_kernel);
EXPORT_SYMBOL_GPL(perf_event_read_value);
EXPORT_SYMBOL_GPL(perf_register_guest_info_callbacks);
EXPORT_SYMBOL_GPL(perf_unregister_guest_info_callbacks);
EXPORT_SYMBOL_GPL(perf_swevent_get_recursion_context);
EXPORT_SYMBOL_GPL(perf_trace_run_bpf_submit);
EXPORT_SYMBOL_GPL(perf_tp_event);
EXPORT_SYMBOL_GPL(perf_pmu_register);
EXPORT_SYMBOL_GPL(perf_pmu_unregister);
EXPORT_SYMBOL_GPL(perf_event_create_kernel_counter);
EXPORT_SYMBOL_GPL(perf_pmu_migrate_context);
EXPORT_SYMBOL_GPL(perf_event_sysfs_show);
```

callees of `perf_pmu_register` are:

```c
// AMD
arch/x86/events/amd/ibs.c|720| <<perf_ibs_pmu_init>> ret = perf_pmu_register(&perf_ibs->pmu, name, -1);
arch/x86/events/amd/iommu.c|462| <<_init_perf_amd_iommu>> ret = perf_pmu_register(&perf_iommu->pmu, name, -1);
arch/x86/events/amd/power.c|297| <<amd_power_pmu_init>> ret = perf_pmu_register(&pmu_class, "power", -1);
arch/x86/events/amd/uncore.c|504| <<amd_uncore_init>> ret = perf_pmu_register(&amd_nb_pmu, amd_nb_pmu.name, -1);
arch/x86/events/amd/uncore.c|518| <<amd_uncore_init>> ret = perf_pmu_register(&amd_l2_pmu, amd_l2_pmu.name, -1);
// Intel
arch/x86/events/intel/core.c|3565| <<intel_pmu_init>> /* x86_pmu, BTS, PEBS probe and setup */
arch/x86/events/intel/bts.c|621| <<bts_init>> return perf_pmu_register(&bts_pmu, "intel_bts", -1);
arch/x86/events/intel/cqm.c|1753| <<intel_cqm_init>> ret = perf_pmu_register(&intel_cqm_pmu, "intel_cqm", -1);
arch/x86/events/intel/cstate.c|619| <<cstate_init>> err = perf_pmu_register(&cstate_core_pmu, cstate_core_pmu.name, -1);
arch/x86/events/intel/cstate.c|629| <<cstate_init>> err = perf_pmu_register(&cstate_pkg_pmu, cstate_pkg_pmu.name, -1);
arch/x86/events/intel/pt.c|1459| <<pt_init>> ret = perf_pmu_register(&pt_pmu.pmu, "intel_pt", -1);
arch/x86/events/intel/rapl.c|813| <<rapl_pmu_init>> ret = perf_pmu_register(&rapl_pmus->pmu, "power", -1);
arch/x86/events/intel/uncore.c|753| <<uncore_pmu_register>> ret = perf_pmu_register(&pmu->pmu, pmu->name, -1);
// Common Core
arch/x86/events/core.c|1843| <<init_hw_perf_events>> err = perf_pmu_register(&pmu, "cpu", PERF_TYPE_RAW);
arch/x86/events/msr.c|258| <<msr_init>> perf_pmu_register(&pmu_msr, "msr", -1);
kernel/events/core.c|7772| <<perf_tp_register>> perf_pmu_register(&perf_tracepoint, "tracepoint", PERF_TYPE_TRACEPOINT);
kernel/events/core.c|10818| <<perf_event_init>> perf_pmu_register(&perf_swevent, "software", PERF_TYPE_SOFTWARE);
kernel/events/core.c|10819| <<perf_event_init>> perf_pmu_register(&perf_cpu_clock, NULL, -1);
kernel/events/core.c|10820| <<perf_event_init>> perf_pmu_register(&perf_task_clock, NULL, -1);
kernel/events/hw_breakpoint.c|640| <<init_hw_breakpoint>> perf_pmu_register(&perf_breakpoint, "breakpoint", PERF_TYPE_BREAKPOINT);

// note that BTS(Branch Trace Store) and PEBS(Precise-Event Based Sampling) are a.k.a DS/Debug Store.
// e.g. see X86_FEATURE_PEBS in arch/x86/include/asm/cpufeatures.h
```


#### Initialization

`perf_events` subsystem gets initialized in `perf_event_init`, which is called
in `start_kernel` and after all dependencies such as timekeeping subsystem, and
timer subsystem.

In order to expose its device model,  `perf_events` subsystem has its own bus
named "event_source", which is initialized in function `perf_event_sysfs_init`
via`device_initcall`, see `do_initcalls` form `init/main.c`.

## Others Hardware Counter

- Intel RAPL: RAPL exposes energy counters and performance counters that Intel
  believes matches actual power measurements. (RAPL is short for Running Average
  Power Limit)

## References

- [Hardware performance counter](https://en.wikipedia.org/wiki/Hardware_performance_counter)
- [How does perf work?](https://jvns.ca/blog/2016/03/12/how-does-perf-work-and-some-questions/)
- [HN: How does perf work?](https://news.ycombinator.com/item?id=11277172)
- [Jim Kukunas. Power and Performance, Software Analysis and Optimization: Chapter 8 Perf, 2015.](https://www.elsevier.com/books/power-and-performance/kukunas/978-0-12-800726-6)
- [Intel Architecture Software Developer Manual](http://www.intel.com/content/www/us/en/processors/architectures-software-developer-manuals.html)
- [Intel ASDM](https://software.intel.com/en-us/articles/intel-sdm)
    - Uncore Performance Monitoring Reference Manuals
        - e.g. Intel® Xeon® Processor E5-2600 v2 Product Family Uncore Performance Monitoring Reference Manual
    - Related Specifications
        - e.g. Intel® 64 and IA32 Architectures Performance Monitoring Events
- [A Study of Performance Monitoring Unit, perf and perf_events Subsystem](http://rts.lab.asu.edu/web_438/project_final/CSE_598_Performance_Monitoring_Unit.pdf)
- [A Study of Linux Perf and Slab Allocation Sub-Systems](https://uwspace.uwaterloo.ca/handle/10012/10184)
