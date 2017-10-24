---
layout : default
---

# System V Reboot is Funny

The system reboot process can be quite involved. Especially on some old
systems which adopts the older System-V Init system, e.g. sysvinit-2.85.

Under the older sysvinit release, both `reboot` and `poweroff` are just a 
symlink pointing to `halt` (`/sbin/halt`).

The whole process looks like the following:

```
// reboot.h
#define BMAGIC_REBOOT        0x01234567
#define init_reboot(magic) reboot(0xfee1dead, 672274793, magic)

// @file: sysvinit-2.85/src/halt.c
# reboot // it's symlink pointing to /sbin/halt

main @sysvinit-2.85/src/halt.c
|
`chdir("/")
`get_runlevel // get runlevel 4.
`do_shutdown(do_reboot ? "-r" : "-h", tm)
`open("/halt", O_RDWR|O_CREAT, 0644)
`execve("/sbin/shutdown", ["shutdown", "-r", "now"], [/* 19 vars */]);
    |
    v
    `main @src/shutdown.c
        | /* "-r", Automatic reboot */
        `down_level[0] = '6';
        `shutdown
            |
            `execve("/sbin/init", ["/sbin/init", "6"], [/* 19 vars */]);
                |
                v
                main @src/init.c
                | /* Open the fifo /dev/initctl and write a command asking runlevel 6. */
                `telinit
                    |
       +------------+
       |
       v
/* (pid 1) init got a change runlevel request through the
 * init's control file. */
fifo_new_level
    read_inittab --\ // change to runlevel 6 by executing "l6:6:wait:/etc/rc.d/rc 6"
    setproctitle   |
                   |
/etc/rc.d/rc 6 <---/ // export runlevel 6; First, run KILL scripts, /etc/rc$runlevel.d/K??
    |                // then, run the START scripts. /etc/rc$runlevel.d/S??;
    v
K25dhcpd -> ../init.d/dhcpd
K40xinetd -> ../init.d/xinetd
K50crond -> ../init.d/crond
K55krb5kdc -> ../init.d/krb5kdc
K77rpcbind -> ../init.d/rpcbind
K78syslog -> ../init.d/syslog
K80network -> ../init.d/network
S00killall -> ../init.d/killall
S01reboot -> ../init.d/halt // This is a really workhourse! It is doing a lot,
    |                       // but finally, it forks /sbin/reboot. (Hi reboot!)
    v
"/sbin/reboot -i -d"        // Note that now the runlevel is 6.
    |
    `sync
    `sleep(2)
    `ifdown
    `init_reboot(BMAGIC_REBOOT) @sysvinit-2.85/src/halt.c
        | (Glibc syscall wrapper)
        `reboot @glibc-2.17-c758a686/sysdeps/unix/sysv/linux/reboot.c
            |
            `INLINE_SYSCALL (reboot, 3, (int) 0xfee1dead, 672274793, howto)
                | (syscall)
      _________ v (Kernel Mode) _________________________
            system_call_fastpath
                |
                `sys_reboot @kernel/sys.c
                    |
                    `kernel_restart
                        |
                        `kernel_restart_prepare
                            |
                            `blocking_notifier_call_chain
                            |   `notifier_call_chain
                            |       `...
                            `system_state = SYSTEM_RESTART
                            `usermodehelper_disable
                            `device_shutdown
                            `syscore_shutdown

```

[back](../)

