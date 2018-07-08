# Wring Virtio Driver

## Virtio Over PCI Bus (section 4.1 in Spec v1.0)

Virtio devices in QEMU are PCI devices with VendorID 1af4,
PCI_VENDOR_ID_REDHAT_QUMRANET. Qumranet donated their vendor ID for devices
0x1000 thru 0x10FF. (see `drivers/virtio/virtio_pci_common.c` or 
`virtio_pci_id_table`).

It looks like the following (e.g. `virtio_net`) in QEMU: (`info pci`)

```
  Bus  0, device   2, function 0:
    Ethernet controller: PCI device 1af4:1000
                                    ^~~~~~~~~
      IRQ 11.
      BAR0: I/O at 0xc040 [0xc05f].
      BAR1: 32 bit memory at 0xfebd1000 [0xfebd1fff].
      BAR4: 64 bit prefetchable memory at 0xfe000000 [0xfe003fff].
      BAR6: 32 bit memory at 0xffffffffffffffff [0x0003fffe].
      id ""
```

And the following by executing `lspci` in guest.

```
/root # lspci -vvs 00:02.0
00:02.0 Class 0200: Device 1af4:1000
        Subsystem: Device 1af4:0001
        Control: I/O+ Mem+ BusMaster+ SpecCycle- MemWINV- VGASnoop- ParErr- Stepping- SERR+ FastB2B- DisINTx-
        Status: Cap+ 66MHz- UDF- FastB2B- ParErr- DEVSEL=fast >TAbort- <TAbort- <MAbort- >SERR- <PERR- INTx-
        Latency: 0
        Interrupt: pin A routed to IRQ 11
        Region 0: I/O ports at c040 [size=32]
        Region 1: Memory at febd1000 (32-bit, non-prefetchable) [size=4K]
        Region 4: Memory at fe000000 (64-bit, prefetchable) [size=16K]
        Expansion ROM at feb80000 [disabled] [size=256K]
        Capabilities: [98] MSI-X: Enable- Count=3 Masked-
                Vector table: BAR=1 offset=00000000
                PBA: BAR=1 offset=00000800
        Capabilities: [84] Vendor Specific Information: VirtIO: <unknown>
                BAR=0 offset=00000000 size=00000000
        Capabilities: [70] Vendor Specific Information: VirtIO: Notify
                BAR=4 offset=00003000 size=00001000 multiplier=00000004
        Capabilities: [60] Vendor Specific Information: VirtIO: DeviceCfg
                BAR=4 offset=00002000 size=00001000
        Capabilities: [50] Vendor Specific Information: VirtIO: ISR
                BAR=4 offset=00001000 size=00001000
        Capabilities: [40] Vendor Specific Information: VirtIO: CommonCfg
                BAR=4 offset=00000000 size=00001000
        Kernel driver in use: virtio-pci
```

### Device Layout

```
virtio_pci_device
    + struct virio_device vdev;
    + struct pci_dev *pci_dev;
```

### Probing and Initialization

virtio device is registered by `virtio-pci` in the end of `virtio_pci_probe`.

1. `virtio_pci_probe`, create virtio device and set its bus to `virtio_bus`,
   invokes secondary probe, `virtio_pci_{legacy|modern}_probe`, which setups
   up `virtio_config_ops` for the `virtio_device`.

```
virtio_pci_probe                @drivers/virtio/virtio_pci_common.c
    virtio_pci_modern_probe     @drivers/virtio/virtio_pci_modern.c
```

   And call `register_virtio_device`, which in turn, invokes `device_register`.

```
register_virtio_device
    device_register
        device_add
            bus_add_device
            bus_probe_device
                virtio_dev_match
                virtio_dev_probe (.probe of virtio_bus)
```
 
2. `virtio_dev_probe`
3. virtio driver probe, i.e. `drv->probe(dev)`.

## Core data structures and APIs

- driver register

`struct virtio_device_id` - id_table

`struct vritio_driver @include/linux/virtio.h` 

`register_virtio_driver @drivers/virtio/virtio.c`

`MODULE_DEVICE_TABLE(virtio, id_table)`

## Virtqueue Configuration and Allocation / config->find_vqs

* Virtio Protocol: Virtio Protocol
* Virtqueue allocation and initialization
* setup callbacks (vp_callback_t)

`vdev->config->find_vqs` => `vp_modern_find_vqs` -> `vp_find_vqs`
(type of `vdev` is `virtio_device`)

```
vp_find_vqs
    vp_try_to_find_vqs
    allocate an array of `virtio_pci_vq_info*` for house-keeping
    vp_request_msix_vectors
    vp_setup_vq
        allocate `virtio_pci_vq_info`
        vp_dev->setup_vq    /* alloc virtqueue and setup callback */
            virtio_pci_modern.c:setup_vq
                                ^~~~~~~~
                /* Check if queue is either not available or already active. */
                num = vp_ioread16(&cfg->queue_size);
                /* get offset of notification word for this vq */
                off = vp_ioread16(&cfg->queue_notify_off);
                /**! create `vring_virtqueue` but returns `virtqueue` **/
                vring_create_virtqueue
                ^~~~~~~~~~~~~~~~~~~~~~
                    /**! allocate each vring individually */
                    vring_alloc_queue
                    ^~~~~~~~~~~~~~~~~
                    vring_init
                    __vring_new_virtqueue
                    ^~~~~~~~~~~~~~~~~~~~~
                /* activate the queue: write size, desc, avail, used */
                vp_iowrite16(&cfg->queue_size)
                vp_iowrite64_twopart(&cfg->queue_desc_lo, &cfg->desc_queue_hi)
                vp_iowrite64_twopart(&cfg->queue_avail_lo, &cfg->desc_avail_hi)
                vp_iowrite64_twopart(&cfg->queue_used_lo, &cfg->queue_used_hi)
                /* optionaly map notification resource (IO port and Memory),
                 * and save the address to `vq->priv`*/
                    pci_iomap_range
                vp_iowrite16(&cfg->queue_msix_vector)/* activate msix vector */

    request_irq  /* allocate per-vq irq and setup vring_interrupt entry */
```

`virtio_dev.config_ops` -> `virtio_pci_config_ops`

- setup_vq, which is a communication process between virtio fron-end driver
  and QEMU's host device. They communicate control information including vring
  and ISR via PCI config space. 

  For the driver, `virtio_pci_common_cfg` is located at `vp_dev->common`

```
* callback list
vp_dev->|virtqueue| -> |virtio_pci_vq_info| -> |virtio_pci_vq_info| -> ...
         list_head  ->  info->node     ->       info->node  -> ...
```

```
vring_create_virtqueue
    `vring_alloc_queue      <= allocate a memory chunk for vring
    `vring_init             <= initialize a vring structure/descriptor
    `__vring_new_virtqueue  <= allocate vring_virtqueue but return its virtuque
                               (use to_vvq() to convert a virtqueue back to a
                                vring_virtqueue)
```

```
vring_interrupt -> vq.callback

vp_interrupt            <= A small wrapper to also acknowledge the interrupt
vp_vring_interrupt      <= Notify all virtqueues on an interrupt
    -> vring_interrupt  <= Invoke callback of a virtqueue
```

## Virtio Rings

**`vring` is actual memory layout for this queue**

**`virtqueue` is a queue to register buffers for sending and receiving**
All transport methods manipulate around `virtqueue`, such as:

- `virtqueue_add`
- `virtqueue_kick`
- `virtqueue_notify`
- `virtqueue_get_buf`
- `virtqueue_disable|enable_cb`
- `virtqueue_poll`
- `virtqueue_get_ring_size`
- `virtqueue_get_desc_addr`
- `virtqueue_get_avail_addr`
- `virtqueue_get_used_addr`

**`vring_virtqueue` is the key fundamental structure and used internally**
in a virtio driver.

In short `vring` is about memory, `virtqueue` is about API, and last but
not least `vring_virtqueue` is used internally in virtio driver.

Meaning of the vring descriptor and the memory layout of virtqueue.

```
vr->num = N
v---------(align to 64B)--------v---------------------------
| N * vring_desc | N * 2B + 6B  | N * vring_used_elem + 6B |
------------------------------------------------------------
^ vr->desc       ^ vr->avail    ^ vr->used

The descriptor ring, desc ring, is a static single linked list.

Virtio Front-end:
publish avail->idx
maintain local: (avail->idx == last_avail_idx == free_head)

Virtio Backend:
publish used->idx
maintain local: last_avail_idx, last_used_idx
```

1. every kick resets `vq->num_added` to 0.

`vring_virtqueue`, `vring`, and `virtqueue`

```
vring_virtqeuue
    * struct virtqueue vq;
    * struct vring vring;
      free_head
      num_added
      last_used_idx
      ...
    * struct vring_desc_state desc_state[]; (*N)
```

- virtio_device_ready

## Transport

Tx: virtqueue_add_outbuf, virtqueue_kick

`virtqueue_kick` is actually invokes `vp_notify` (see `virtio_common_pci.c`).

```c
/* the notify function used when creating a virt queue */
bool vp_notify(struct virtqueue *vq)
{
    /* we write the queue's selector into the notification register to
     * signal the other end */
     iowrite16(vq->index, (void __iomem *)vq->priv);
     return true;
}
```

Rx: MSIX interrupts invoke callbacks we registered when configuring virtqueue,
virtqueue_get_buf.

- kick/notify can be traced by `trace_virtio_queue_notify`, see
  `virtio_queue_notify`, `virtio_queue_notify_vq`, `virtio_add_queue`,
  `trace_virtqueue_pop`.

- `struct Vring` in QEMU is equal to `vring` in kernel.
- `struct VirtQueue` in QEMU is quivalent to `vring_virtqueue` in kernel.
- QEMU translante guest physical address (GPA) to host virtual address (HVA),
  see `virtqueue_pop`, `vring_desc_read`

```
vring_desc_read
    address_space_read_cached
        address_space_read
            flatview_read
                flatview_translate
                qemu_map_ram_ptr
```

- When exchanging data between guest and host, lots of cycles are used for
  address translating (e.g. GFP->HVA). Thus, efficient scatter list is
  preferable. (It also means that the guest is free to use any valid GPA)


## Q: How virtio front-end drivers to share vring (or virtqueue) with QEMU?

Front-end drivers kick QEMU by writing to an I/O prot (in the PCI virtio 
device's I/O BARs; you can find the address with `lspci`).

To share memory between the guest and the virtio device, QEMU does "DMA"
with `address_space_map` and `address_space_unmap` (or `cpu_physical_memory_map` and `cpu_physical_memory_unmap` instead depending on the QEMU version).

--Paolo Bonzini

## Vhost

It's a.k.a Vhost-kernel.

key notes:
- how vhost provides in-kernel virtio devices for KVM.
- ioeventfd, a kick eventfd
- irqfd, a call eventfd
- the guest memory mapping

where to find more (here are the main points):
- `driver/vhost/vhost.c`, common vhost drive code
- `driver/vhost/net.c`, vhost-net driver
- `virt/kvm/eventfd.c`, ioeventfd and irqfd

The QEMU userspace code shows how to initialize the vhost instance:
- `hw/vhost.c`, common vhost initialization code
- `hw/vhost_net.c`, vhost-net initialization

ioctl path: `vhost_net_ioctl`, `vhost_dev_ioctl`, `vhost_vring_ioctl`.
files: `drivers/vhost/{net,vhost}.c`

### Vhost overview

The vhost drivers in Linux provide in-kernel virtio device emulation. Normally the QEMU userspace process emulates I/O accesses from the guest. Vhost puts virtio emulation code into the kernel, taking QEMU userspace out of the picture. This allows device emulation code to directly call into kernel subsystems instead of performing system calls from userspace.

The vhost-net driver emulates the virtio-net network card in the host kernel. Vhost-net is the oldest vhost device and the only one which is available in mainline Linux. Experimental vhost-blk and vhost-scsi devices have also been developed.

In Linux 3.0 the vhost code lives in drivers/vhost/. Common code that is used by all devices is in drivers/vhost/vhost.c. This includes the virtio vring access functions which all virtio devices need in order to communicate with the guest. The vhost-net code lives in drivers/vhost/net.c.

### The vhost driver model

The vhost-net driver creates a /dev/vhost-net character device on the host. This character device serves as the interface for configuring the vhost-net instance.

When QEMU is launched with -netdev tap,vhost=on it opens /dev/vhost-net and initializes the vhost-net instance with several ioctl(2) calls. These are necessary to associate the QEMU process with the vhost-net instance, prepare for virtio feature negotiation, and pass the guest physical memory mapping to the vhost-net driver.

During initialization the vhost driver creates a kernel thread called vhost-$pid, where $pid is the QEMU process pid. This thread is called the "vhost worker thread". The job of the worker thread is to handle I/O events and perform the device emulation.

### In-kernel virtio emulation

Vhost does not emulate a complete virtio PCI adapter. Instead it restricts itself to virtqueue operations only. QEMU is still used to perform virtio feature negotiation and live migration, for example. This means a vhost driver is not a self-contained virtio device implementation, it depends on userspace to handle the control plane while the data plane is done in-kernel.

The vhost worker thread waits for virtqueue kicks and then handles buffers that have been placed on the virtqueue. In vhost-net this means taking packets from the tx virtqueue and transmitting them over the tap file descriptor.

File descriptor polling is also done by the vhost worker thread. In vhost-net the worker thread wakes up when packets come in over the tap file descriptor and it places them into the rx virtqueue so the guest can receive them.

Vhost as a userspace interface
One surprising aspect of the vhost architecture is that it is not tied to KVM in any way. Vhost is a userspace interface and has no dependency on the KVM kernel module. This means other userspace code, like libpcap, could in theory use vhost devices if they find them convenient high-performance I/O interfaces.

When a guest kicks the host because it has placed buffers onto a virtqueue, there needs to be a way to signal the vhost worker thread that there is work to do. Since vhost does not depend on the KVM kernel module they cannot communicate directly. Instead vhost instances are set up with an eventfd file descriptor which the vhost worker thread watches for activity. The KVM kernel module has a feature known as ioeventfd for taking an eventfd and hooking it up to a particular guest I/O exit. QEMU userspace registers an ioeventfd for the VIRTIO_PCI_QUEUE_NOTIFY hardware register access which kicks the virtqueue. This is how the vhost worker thread gets notified by the KVM kernel module when the guest kicks the virtqueue.

On the return trip from the vhost worker thread to interrupting the guest a similar approach is used. Vhost takes a "call" file descriptor which it will write to in order to kick the guest. The KVM kernel module has a feature called irqfd which allows an eventfd to trigger guest interrupts. QEMU userspace registers an irqfd for the virtio PCI device interrupt and hands it to the vhost instance. This is how the vhost worker thread can interrupt the guest.

In the end the vhost instance only knows about the guest memory mapping, a kick eventfd, and a call eventfd.

- [QEMU Internals: vhost architecture](http://blog.vmsplice.net/2011/09/qemu-internals-vhost-architecture.html)

## Vhost-user Protocol

- Master is the application that shares its virtqueues, in our case QEMU.
- Slave is the consumer of the virtqueues.

The protocol is designed kind of like TFLV. (Tag,Flag,Length,Variant)

```
------------------------------------
| request | flags | size | payload |
------------------------------------
```

## DPDK Libaray

EAL: Environment Abstraction Layer

- argparse: dpdk/lib/librte_eal/common/include/rte_common.h
- hugepage: dpdk/lib/librte_eal/linuxapp/eal/eal_hugepage_info.c

- [DPDK Programming Guide](http://dpdk.org/doc/guides/prog_guide/overview.html)

## QEMU Device Type: vhost-user-vchan-pci

- Define device type in `hw/virtio/virtio-pci.h`
    + define your own Virtio/Vhost device type in `include/hw/virtio/`
    + and define a PCI wrapper in `hw/virtio/virtio-pci.h`
    + see `VHostUserSCSIPCI`/`VHostUserSCSI`

- Add TypeInfo to file `hw/virtio/virtio-pci.c`, see `include/qom/object.h`,
  and `vhost_user_blk_pci_info`. (or `ag 'TypeInfo vhost_user_' qemu/`)
    + see `type_new`, `type_register`, `type_initialized` in file
    `qom/object.c`

- Add `static const TypeInfo vhost_user_vchan_pci_info` to file
`hw/virtio/virtio_pci.c`.
    + see `hw/scsi/vhost-scsi.c` and `hw/scsi/vhost-user-scsi.c`

- Add `[device]_pci_instance_init` and `[device]_pci_class_init`

- Register type in `virtio_pci_register_types`

Notes:

1. Virtio-PCI is odd. Ioports are LE but config space is target native endian
2. `hw/virtio/virtio-pci.c` - host device core functions
3. pci config read and write are implemented as `MemoryRegionsOps`.

```
virtio_pci_config_write
    virtio_ioport_write
        virtio_set_features
        /* VIRTIO_PCI_QUEUE_PFN */
        virtio_queue_set_addr
        ...
```

4. All `MemoryRegionOps` are initialized in `virtio_pci_modern_regions_init`.

## Virtio PCI Common Config

Fields in VIRTIO_PCI_CAP_COMMON_CFG is defined `struct virtio_pci_common_cfg`
in file `include/uapi/linux/virtio_pci.h`, and saved in `vp_dev->common`.

### debug

```
#gdb /home/cxf154136/opt/bin/qemu-system-x86_64

(gdb) b vhost_user_scsi_pci_instance_init
Breakpoint 1 at 0x56ad7b: file /home/cxf154136/work/srcs/qemu-2.10.2/hw/virtio/virtio-pci.c, line 2176.

-machine q35,accel=kvm,usb=off -cpu host -m 1G \
-object memory-backend-file,id=mem,size=1G,mem-path=/dev/hugepages/libvirt/qemu,share=on \
-numa node,memdev=mem \
-realtime mlock=off \
-smp 1,sockets=1,cores=1,threads=1 -nographic \
-rtc base=utc,clock=vm,driftfix=slew \
-no-reboot -boot strict=on \
-kernel /home/cxf154136/work/odps-gpu/kernel-for-odps \
-initrd /home/cxf154136/work/odps-gpu/mini-initramfs/custom-initramfs.cpio.gz \
-append 'console=ttyS0 panic=1 no_timer_check' \
-chardev socket,id=char0,path=/var/tmp/vhost.1 \
-device vhost-user-vchan-pci,id=scsi0,chardev=char0


Thread 1 "qemu-system-x86" hit Breakpoint 1, vhost_user_scsi_pci_instance_init (obj=0x5555577076c0)
    at /home/cxf154136/work/srcs/qemu-2.10.2/hw/virtio/virtio-pci.c:2176
2176        VHostUserSCSIPCI *dev = VHOST_USER_SCSI_PCI(obj);
(gdb) bt
#0  vhost_user_scsi_pci_instance_init (obj=0x5555577076c0)
    at /home/cxf154136/work/srcs/qemu-2.10.2/hw/virtio/virtio-pci.c:2176
#1  0x0000555555b03790 in object_init_with_type (obj=0x5555577076c0, ti=0x555556718660)
    at /home/cxf154136/work/srcs/qemu-2.10.2/qom/object.c:344
#2  0x0000555555b039c5 in object_initialize_with_type (data=0x5555577076c0, size=34592,
    type=0x555556718660) at /home/cxf154136/work/srcs/qemu-2.10.2/qom/object.c:375
#3  0x0000555555b03e30 in object_new_with_type (type=0x555556718660)
    at /home/cxf154136/work/srcs/qemu-2.10.2/qom/object.c:483
#4  0x0000555555b03e6d in object_new (typename=0x555556721980 "vhost-user-scsi-pci")
    at /home/cxf154136/work/srcs/qemu-2.10.2/qom/object.c:493
#5  0x00005555558dbb17 in qdev_device_add (opts=0x5555567218b0, errp=0x7fffffffb778)
    at /home/cxf154136/work/srcs/qemu-2.10.2/qdev-monitor.c:613
#6  0x00005555558e3be5 in device_init_func (opaque=0x0, opts=0x5555567218b0, errp=0x0)
    at /home/cxf154136/work/srcs/qemu-2.10.2/vl.c:2336
#7  0x0000555555c220a4 in qemu_opts_foreach (list=0x555556129420 <qemu_device_opts>,
    func=0x5555558e3ba7 <device_init_func>, opaque=0x0, errp=0x0)
    at /home/cxf154136/work/srcs/qemu-2.10.2/util/qemu-option.c:1104
#8  0x00005555558e9215 in main (argc=31, argv=0x7fffffffbbf8, envp=0x7fffffffbcf8)
    at /home/cxf154136/work/srcs/qemu-2.10.2/vl.c:4663
```

- vring operations

```
vhost_user_vchan_start
    vhost_dev_start                                 [qemu/hw/virtio/vhost.c]
        vhost_dev_set_features
        hdev->vhost_ops->vhost_set_mem_table
        vhost_virtqueue_start                       [qemu/hw/virtio/vhost.c]
            dev->vhost_ops->vhost_get_vq_index
            dev->vhost_ops->vhost_set_vring_num
            dev->vhost_ops->vhost_set_vring_base
            vhost_virtqueue_set_addr
                dev->vhost_ops->vhost_set_vring_addr
```

### debug remove_vq_common (virtio_vchan)

```
ps -ef|grep build.out|grep -v sudo | awk '{print $2}' | head -n1
```

```
rmmod virtio_vchan
vchan->vdev->config->reset() ==> vp_reset [virtio_pci_modern.c]
```

```
./build/app/vhost-vchan -l 0-1 --no-huge -m 1024 -- --socket-file /var/tmp/vhost.1
```

VHOST_CONFIG: read message VHOST_USER_GET_VRING_BASE

```
$sudo pstack 70765
Thread 3 (Thread 0x7fdc4bc01700 (LWP 70766)):
#0  0x00007fdc4bef7489 in syscall () from /lib64/libc.so.6
#1  0x00007fdc4e957c0c in qemu_futex_wait (f=0x7fdc4f4179b4 <rcu_call_ready_event>, val=4294967295) at /home/cxf154136/work/srcs/qemu/include/qemu/futex.h:26
#2  0x00007fdc4e957dd5 in qemu_event_wait (ev=0x7fdc4f4179b4 <rcu_call_ready_event>) at /home/cxf154136/work/srcs/qemu/util/qemu-thread-posix.c:442
#3  0x00007fdc4e96f393 in call_rcu_thread (opaque=0x0) at /home/cxf154136/work/srcs/qemu/util/rcu.c:249
#4  0x00007fdc4c1cfdc5 in start_thread () from /lib64/libpthread.so.0
#5  0x00007fdc4befcd0d in clone () from /lib64/libc.so.6
Thread 2 (Thread 0x7fdc0b3ff700 (LWP 70769)):
#0  0x00007fdc4c1d667d in recvmsg () from /lib64/libpthread.so.0
#1  0x00007fdc4e8fc654 in qio_channel_socket_readv (ioc=0x7fdc4f4c5f60, iov=0x7fdc0b3fe4a0, niov=1, fds=0x7fdc0b3fe480, nfds=0x7fdc0b3fe488, errp=0x0) at /home/cxf154136/work/srcs/qemu/io/channel-socket.c:477
#2  0x00007fdc4e8f7e89 in qio_channel_readv_full (ioc=0x7fdc4f4c5f60, iov=0x7fdc0b3fe4a0, niov=1, fds=0x7fdc0b3fe480, nfds=0x7fdc0b3fe488, errp=0x0) at /home/cxf154136/work/srcs/qemu/io/channel.c:64
#3  0x00007fdc4e8e9604 in tcp_chr_recv (chr=0x7fdc4f4c5bc0, buf=0x7fdc0b3fe5c0 "\v", len=12) at /home/cxf154136/work/srcs/qemu/chardev/char-socket.c:277
#4  0x00007fdc4e8e9e1c in tcp_chr_sync_read (chr=0x7fdc4f4c5bc0, buf=0x7fdc0b3fe5c0 "\v", len=12) at /home/cxf154136/work/srcs/qemu/chardev/char-socket.c:458
#5  0x00007fdc4e8e4271 in qemu_chr_fe_read_all (be=0x7fdc50448f70, buf=0x7fdc0b3fe5c0 "\v", len=12) at /home/cxf154136/work/srcs/qemu/chardev/char-fe.c:72
#6  0x00007fdc4e56b69a in vhost_user_read (dev=0x7fdc50448fb0, msg=0x7fdc0b3fe5c0) at /home/cxf154136/work/srcs/qemu/hw/virtio/vhost-user.c:142
#7  0x00007fdc4e56c2ab in vhost_user_get_vring_base (dev=0x7fdc50448fb0, ring=0x7fdc0b3fe730) at /home/cxf154136/work/srcs/qemu/hw/virtio/vhost-user.c:457
#8  0x00007fdc4e569529 in vhost_virtqueue_stop (dev=0x7fdc50448fb0, vdev=0x7fdc50448de0, vq=0x7fdc504528f0, idx=0) at /home/cxf154136/work/srcs/qemu/hw/virtio/vhost.c:1137
#9  0x00007fdc4e56ab6a in vhost_dev_stop (hdev=0x7fdc50448fb0, vdev=0x7fdc50448de0) at /home/cxf154136/work/srcs/qemu/hw/virtio/vhost.c:1598
#10 0x00007fdc4e571441 in vhost_user_vchan_stop (vdev=0x7fdc50448de0) at /home/cxf154136/work/srcs/qemu/hw/virtio/vhost-user-vchan.c:150
#11 0x00007fdc4e571537 in vhost_user_vchan_set_status (vdev=0x7fdc50448de0, status=0 '\000') at /home/cxf154136/work/srcs/qemu/hw/virtio/vhost-user-vchan.c:177
#12 0x00007fdc4e5604f5 in virtio_set_status (vdev=0x7fdc50448de0, val=0 '\000') at /home/cxf154136/work/srcs/qemu/hw/virtio/virtio.c:1146
#13 0x00007fdc4e8058c3 in virtio_pci_common_write (opaque=0x7fdc50440b30, addr=20, val=0, size=1) at /home/cxf154136/work/srcs/qemu/hw/virtio/virtio-pci.c:1294
#14 0x00007fdc4e4ed381 in memory_region_write_accessor (mr=0x7fdc50441500, addr=20, value=0x7fdc0b3fe938, size=1, shift=0, mask=255, attrs=...) at /home/cxf154136/work/srcs/qemu/memory.c:560
#15 0x00007fdc4e4ed58c in access_with_adjusted_size (addr=20, value=0x7fdc0b3fe938, size=1, access_size_min=1, access_size_max=4, access=0x7fdc4e4ed2a0 <memory_region_write_accessor>, mr=0x7fdc50441500, attrs=...) at /home/cxf154136/work/srcs/qemu/memory.c:626
#16 0x00007fdc4e4f044a in memory_region_dispatch_write (mr=0x7fdc50441500, addr=20, data=0, size=1, attrs=...) at /home/cxf154136/work/srcs/qemu/memory.c:1502
#17 0x00007fdc4e49ff55 in flatview_write_continue (fv=0x7fdc0411f430, addr=4261412884, attrs=..., buf=0x7fdc4e1ba028 "", len=1, addr1=20, l=1, mr=0x7fdc50441500) at /home/cxf154136/work/srcs/qemu/exec.c:2929
#18 0x00007fdc4e4a00ba in flatview_write (fv=0x7fdc0411f430, addr=4261412884, attrs=..., buf=0x7fdc4e1ba028 "", len=1) at /home/cxf154136/work/srcs/qemu/exec.c:2974
#19 0x00007fdc4e4a04aa in flatview_rw (fv=0x7fdc0411f430, addr=4261412884, attrs=..., buf=0x7fdc4e1ba028 "", len=1, is_write=true) at /home/cxf154136/work/srcs/qemu/exec.c:3083
#20 0x00007fdc4e4a0563 in address_space_rw (as=0x7fdc4efb8dc0 <address_space_memory>, addr=4261412884, attrs=..., buf=0x7fdc4e1ba028 "", len=1, is_write=true) at /home/cxf154136/work/srcs/qemu/exec.c:3093
#21 0x00007fdc4e50564a in kvm_cpu_exec (cpu=0x7fdc4f4d3800) at /home/cxf154136/work/srcs/qemu/accel/kvm/kvm-all.c:2045
#22 0x00007fdc4e4d5d67 in qemu_kvm_cpu_thread_fn (arg=0x7fdc4f4d3800) at /home/cxf154136/work/srcs/qemu/cpus.c:1128
#23 0x00007fdc4c1cfdc5 in start_thread () from /lib64/libpthread.so.0
#24 0x00007fdc4befcd0d in clone () from /lib64/libc.so.6
Thread 1 (Thread 0x7fdc4e282ac0 (LWP 70765)):
#0  0x00007fdc4c1d5f4d in __lll_lock_wait () from /lib64/libpthread.so.0
#1  0x00007fdc4c1d1d02 in _L_lock_791 () from /lib64/libpthread.so.0
#2  0x00007fdc4c1d1c08 in pthread_mutex_lock () from /lib64/libpthread.so.0
#3  0x00007fdc4e957459 in qemu_mutex_lock (mutex=0x7fdc4efbc440 <qemu_global_mutex>) at /home/cxf154136/work/srcs/qemu/util/qemu-thread-posix.c:65
#4  0x00007fdc4e4d68db in qemu_mutex_lock_iothread () at /home/cxf154136/work/srcs/qemu/cpus.c:1581
#5  0x00007fdc4e953190 in os_host_main_loop_wait (timeout=79063890938868) at /home/cxf154136/work/srcs/qemu/util/main-loop.c:258
#6  0x00007fdc4e953251 in main_loop_wait (nonblocking=0) at /home/cxf154136/work/srcs/qemu/util/main-loop.c:515
#7  0x00007fdc4e62af31 in main_loop () at /home/cxf154136/work/srcs/qemu/vl.c:1917
#8  0x00007fdc4e632ad9 in main (argc=31, argv=0x7ffe378504e8, envp=0x7ffe378505e8) at /home/cxf154136/work/srcs/qemu/vl.c:4795
```

## Vhost Libraries

### DPDK `librte_eal`

`rte_eal_init_`

### DPDK `librte_vhost`

`rte_vhost_driver_register`, register a new vhost-user socket.

`rte_vhost_driver_set_features`, which overwrites the default virtio_net
features.

`rte_vhost_driver_callback_register`, 

`struct vhost_device_ops`

`rte_vhost_driver_start`, create a event loop thread running
`fdset_event_dispatch`.

- vhost-user message handling

```
rte_vhost_driver_start
    vhost_user_start_server
    fdset_event_dispatch
        vhost_user_server_new_connection
            vhost_user_add_connection
                vhost_user_read_cb
                    vhost_user_msg_handler
```


```c
uint16_t rte_vhost_avail_entries();
uint16_t rte_vhost_enqueue_burst(int vid, uint16_t queue_id,
                                 struct rte_mbuf **pkts, uint16_t count);

uint16_t rte_vhost_dequeue_burst(int vid, uint16_t queue_id,
                                 struct rte_mempool *mbuf_pool,
                                 struct rte_mbuf **pkts, uint16_t count);

struct rte_mbuf;
```

`rte_pktmbuf_alloc`

- `dpdk/lib/librte_vhost/`
- `lib/librte_mbuf/rte_mbuf.h`

TODO: write configuration process such as

create a vchan device type

`rte_vhost_get_negotiated_features`

`rte_vhost_get_mem_table`

`rte_vhost_get_vring_num`

`rte_vhost_get_vhost_vring`

see `vs_vhost_net_setup`, `vhost_queue`, `vs_vhost_net_remove`

TODO: write TX, RX path, see `enqueue_pkt`, `dequeue_pkt`

## Sources

- vring implementation, `drivers/virtio/virtio_ring.c`

## Tools

There are tools in kernel source helping testing virtio/ring/vhost,
see `$KSRC/tools/virtio`.

## EPT Misconfiguration

The detailed definition can be seen at 28.2.3.1 in Intel-SDM-vol-3.

In x86 KVM, guest can trigger VM-EXIT with reason EXIT_REASON_EPT_MISCONFIG.

## Build DPDK

```
make config O=build2 T=x86_64-native-linuxapp-gcc
make O=build2 -j32

cd ${RTE_SDK}
export RTE_SDK=$PWD
export RTE_TARGET=build2
export EXTRA_CFLAGS='-O0 -g'
cd examples;
make
```

```
./build/app/vhost-vchan -l 4-5 --no-huge -m 1024 -- --socket-file /var/tmp/vhost.1 [-n]
```

- [Development Kit Root Makefile Help](http://dpdk.org/doc/guides/prog_guide/dev_kit_root_make_help.html)

- [Vhost Sample Application](http://dpdk.org/doc/guides/sample_app_ug/vhost.html)

## Memory Barriers in Virtio

There are comments on the top in file `include/linux/virtio_ring.h` saying
that:

Barriers in virtio are tricky. Non-SMP virito guests can't assume they're not
on an SMP host system, so they need to assume real barriers. Non-SMP virtio
host could skip the barriers, but does anyone care?

For `virtio_pci` on SMP, we don't need to order with respect to MMIO accesses
through relaxed memory I/O windoes, so `virt_mb()` et al are sufficient.

I think that it's because when we are using `virtio_pci`, the control register
such as vring's memory is mapped via `ioremap_nocache`, which ensures that the
memory is marked uncachable on the CPU. And when controlling vring, we only
need a compiler barrier (i.e. `asm volative("" ::: "memory")`). It's also known
as weak barrier mode for virtio (i.e. `weak_barriers` is true).

## References

- [Virtio](https://wiki.osdev.org/Virtio)
- [VIRTIO DRIVER IMPLEMENTATION](http://www.dumais.io/index.php?article=aca38a9a2b065b24dfa1dee728062a12)
- [Memory Part 2](https://lwn.net/Articles/252125/)

## Test

### Latency

```
// begin interaction
      Guest                  Host
        |                      |
update avail on RX             |
update avail on TX        polling TX
publish avail on TX            |
        |                 TX avail noticed
        |                 update used TX
        |                 update used on RX
TX used noticed
RX used noticed
// end interaction
```

Profiling:

- hotspot (Flame Graph)
- Working set size
- IPC, Cache and TLB misses

```
examples:

# Various basic CPU statistics, system wide, for 10 seconds:
perf stat -e cycles,instructions,cache-references,cache-misses,bus-cycles -a sleep 10

# Various CPU level 1 data cache statistics for the specified command:
perf stat -e L1-dcache-loads,L1-dcache-load-misses,L1-dcache-stores command

# Various CPU data TLB statistics for the specified command:
perf stat -e dTLB-loads,dTLB-load-misses,dTLB-prefetch-misses command

# Various CPU last level cache statistics for the specified command:
perf stat -e LLC-loads,LLC-load-misses,LLC-stores,LLC-prefetches command

# Using raw PMC counters, eg, unhalted core cycles:
perf stat -e r003c -a sleep 5 

```

```
perf stat -e cycles,instructions,cache-references,cache-misses,bus-cycles -a sleep 10

perf stat -e cycles,instructions,cache-references,cache-misses,bus-cycles,L1-dcache-loads,L1-dcache-load-misses,L1-dcache-stores,dTLB-loads,dTLB-load-misses,dTLB-prefetch-misses,LLC-loads,LLC-load-misses,LLC-stores,LLC-prefetches -p `pgrep vhost-vchan`
```

```
_qid=`ps -ef|grep build.out|grep -v sudo|grep -v grep| awk '{print $2}' | head -n1`
perf kvm stat live -p $_qid
```

- [Perf KVM Events](https://www.linux-kvm.org/page/Perf_events)
