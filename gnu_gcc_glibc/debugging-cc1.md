---
layout: default
---

# Debugging cc1


We can use the option `-print-prog-name` to display the full path of a compiler
component. e.g. cc1

```
$ gcc -print-prog-name=cc1
/usr/libexec/gcc/x86_64-redhat-linux/7/cc1
```

GCC has an option `-wrapper` to invoke all subcommands under a wrapper program.
In this case, use gdb.

`gcc <options> -wrapper gdb,--args`

It's nedded to install debuginfo of gcc and binutils. Otherwise gdb will
complain that it can't find debugging symbols.

```
$ gcc -c bit.c -wrapper gdb,--args

GNU gdb (GDB) Fedora 8.0.1-26.fc26
Copyright (C) 2017 Free Software Foundation, Inc.
License GPLv3+: GNU GPL version 3 or later <http://gnu.org/licenses/gpl.html>
This is free software: you are free to change and redistribute it.
There is NO WARRANTY, to the extent permitted by law.  Type "show copying"
and "show warranty" for details.
This GDB was configured as "x86_64-redhat-linux-gnu".
Type "show configuration" for configuration details.
For bug reporting instructions, please see:
<http://www.gnu.org/software/gdb/bugs/>.
Find the GDB manual and other documentation resources online at:
<http://www.gnu.org/software/gdb/documentation/>.
For help, type "help".
Type "apropos word" to search for commands related to "word"...
Reading symbols from /usr/libexec/gcc/x86_64-redhat-linux/7/cc1...(no debugging symbols found)...done.
Missing separate debuginfos, use: dnf debuginfo-install cpp-7.2.1-2.fc26.x86_64
(gdb) q
GNU gdb (GDB) Fedora 8.0.1-26.fc26
Copyright (C) 2017 Free Software Foundation, Inc.
License GPLv3+: GNU GPL version 3 or later <http://gnu.org/licenses/gpl.html>
This is free software: you are free to change and redistribute it.
There is NO WARRANTY, to the extent permitted by law.  Type "show copying"
and "show warranty" for details.
This GDB was configured as "x86_64-redhat-linux-gnu".
Type "show configuration" for configuration details.
For bug reporting instructions, please see:
<http://www.gnu.org/software/gdb/bugs/>.
Find the GDB manual and other documentation resources online at:
<http://www.gnu.org/software/gdb/documentation/>.
For help, type "help".
Type "apropos word" to search for commands related to "word"...
Reading symbols from as...Reading symbols from /home/jason/src/test/clang/as...(no debugging symbols found)...done.
(no debugging symbols found)...done.
Missing separate debuginfos, use: dnf debuginfo-install binutils-2.27-24.fc26.x86_64
(gdb) q
```

`$ dnf debuginfo-install gcc-7.2.1-2.fc26 binutils-2.26.1-1.fc25.x86_64`

Having debuginfo installed, we can open the box and get to know how it works.

This is a simple backtrace of a demo for debugging preprocessing operation.

```
(gdb) bt
#0  open64 () at ../sysdeps/unix/syscall-template.S:84
#1  0x00000000012f6572 in open_file(_cpp_file*) () at ../../libcpp/files.c:229
#2  0x00000000012f6152 in find_file_in_dir (invalid_pch=<synthetic pointer>, file=<optimized out>, pfile=0x1e09f10) at ../../libcpp/files.c:422
#3  _cpp_find_file () at ../../libcpp/files.c:533
#4  0x00000000012f73cd in cpp_read_main_file(cpp_reader*, char const*) () at ../../libcpp/init.c:626
#5  0x000000000122e418 in c_common_post_options(char const**) () at ../../gcc/c-family/c-opts.c:986
#6  0x0000000000ce4950 in process_options () at ../../gcc/toplev.c:1217
#7  do_compile () at ../../gcc/toplev.c:1942
#8  toplev::main(int, char**) () at ../../gcc/toplev.c:2094
#9  0x0000000000ce66e7 in main () at ../../gcc/main.c:39
#10 0x00007ffff6c00401 in __libc_start_main (main=0xce66b0 <main>, argc=12, argv=0x7fffffffdfb8, init=<optimized out>, fini=<optimized out>, rtld_fini=<optimized out>,
    stack_end=0x7fffffffdfa8) at ../csu/libc-start.c:289
    #11 0x0000000001211c4a in _start ()
```

[back](../)
