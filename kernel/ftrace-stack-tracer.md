---
layout: default
---

# Ftrace: Kernel Stack Tracing

Kernel has the ability to examine the size of the kernel stack and how
much stack space each function is using.
Enabling the stack tracer (`CONFIG_STACK_TRACER`) will show where the
biggest use of the stack takes place.

The stack tracer is built form the function tracer infrastructure. It
does not uses the Ftrace ring buffer, but it does use the function tracer
to hook into every function call. Because it uses the function tracer
infrastructure, it does not add overhead when not enabled. To enable
the stsack tracer, echo 1 into `/proc/sys/kernel/stack_tracer_enabled`.
To see the max stack size during boot up, add "stacktrace" to the kernel
boot parameters.

```
# echo 1 > /proc/sys/kernel/stack_tracer_enabled
```

The stack tracer checks the size of the stack size at every function call.
If it is greater than the last recorded maximum, it records the stack trace
and updates the maximum with the new size. To see the current maximum, look
at `stack_max_size` file (`/sys/kernel/debug/tracing/stack_max_size`).

```
# cat stack_max_size
4072
```

Not only does this give you the size of the maximum stack found, it also shows
the breakdown of the stack sizes used by each function.

```
# cat stack_trace
        Depth    Size   Location    (44 entries)
        -----    ----   --------
  0)     4072      64   ___slab_alloc+0x5/0x540
  1)     4008      48   __slab_alloc+0x20/0x40
  2)     3960      64   kmem_cache_alloc+0x16e/0x1b0
  3)     3896      16   mempool_alloc_slab+0x1d/0x30
  4)     3880     128   mempool_alloc+0x77/0x1c0
  5)     3752      16   sg_pool_alloc+0x45/0x50
  6)     3736      88   __sg_alloc_table+0x10c/0x160
  7)     3648      48   sg_alloc_table_chained+0x48/0xb0
  8)     3600      40   scsi_init_sgtable+0x26/0x70
  9)     3560      72   scsi_init_io+0x44/0x1c0
 10)     3488     120   sd_init_command+0x2b2/0xde0
 11)     3368       8   scsi_setup_cmnd+0x101/0x160
 12)     3360      80   scsi_prep_fn+0xf4/0x180
 13)     3280      56   blk_peek_request+0x16e/0x2b0
 14)     3224     104   scsi_request_fn+0x3f/0x5f0
 15)     3120      24   __blk_run_queue+0x33/0x40
 16)     3096     192   cfq_insert_request+0x2dd/0x600
 17)     2904      56   __elv_add_request+0x1b1/0x2b0
 18)     2848      96   blk_flush_plug_list+0x201/0x230
 19)     2752      48   io_schedule_timeout+0x4b/0x110
 20)     2704      24   bit_wait_io+0x1b/0x70
 21)     2680      64   __wait_on_bit+0x58/0x90
 22)     2616     120   out_of_line_wait_on_bit+0x82/0xb0
 23)     2496      96   do_get_write_access+0x1f4/0x440
 24)     2400      40   jbd2_journal_get_write_access+0x53/0x70
 25)     2360      56   __ext4_journal_get_write_access+0x3b/0x80
 26)     2304      48   ext4_reserve_inode_write+0x77/0xa0
 27)     2256      88   ext4_mark_inode_dirty+0x53/0x220
 28)     2168      32   ext4_dirty_inode+0x48/0x70
 29)     2136      56   __mark_inode_dirty+0x165/0x350
 30)     2080      72   ext4_da_update_reserve_space+0x1a0/0x1b0
 31)     2008     328   ext4_ext_map_blocks+0xcd3/0x1cc0
 32)     1680     128   ext4_map_blocks+0x172/0x5e0
 33)     1552     328   ext4_writepages+0x71f/0xca0
 34)     1224      16   do_writepages+0x1e/0x30
 35)     1208      72   __writeback_single_inode+0x45/0x310
 36)     1136     224   writeback_sb_inodes+0x24b/0x5d0
 37)      912      72   __writeback_inodes_wb+0x92/0xc0
 38)      840     176   wb_writeback+0x268/0x300
 39)      664     176   wb_workfn+0x22e/0x410
 40)      488      72   process_one_work+0x184/0x430
 41)      416     104   worker_thread+0x4e/0x480
 42)      312     136   kthread+0xd8/0xf0
 43)      176     176   ret_from_fork+0x1f/0x40
```

To reset the maximum, echo "0" into the `stack_max_size` file.

```
# echo 0 > stack_max_size
```

Keeping this running for a while will show where the kernel is using
a bit too much stack. But remember that the stack tracer only has no
overhead when it is not enabled. When it is running you may notice a
bit of performance degradation.

Note that the stack tracer will not tracer the max stack size when
kernel is using a separate stack. Because interrupt have their own
stack, it will no trace the stack usage there.

# References

- [Debugging the kernel using Ftrace - part 2](https://lwn.net/Articles/366796/)
- [Embedded Linux Wiki: ftrace](http://elinux.org/Ftrace)
- [PATCH ftrace: print stack usage right before oops](https://lists.gt.net/linux/kernel/1933198)

[back](../)


