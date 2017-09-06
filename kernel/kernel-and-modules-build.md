---
layout: default
---

# Kernel and Modules Build

## About `init/main.o`

`init/main.o` includes a file path which is helpful when extracting kernel build
information from `vmlinuz`, `vmlinux`, or `main.o`.

In fact, the file path is added by the C preprocessor, CPP, while replacing
the macro `__FILE__`. In `init/main.c`, `start_kernel()` invokes `WARN()`.
`WARN()` invokes `__WARN_printf()`, which invokes `warn_slowpath_fmt()` and
passes file path `__FILE__`. Also, `BUG()` is replaced by a code snippet
including a file path in case of `CONFIG_DEBUG_BUGVERBOSE` was set.

```c
//...
#define __WARN_printf(arg...)   warn_slowpath_fmt(__FILE__, __LINE__, arg)
//...
#ifndef WARN
#define WARN(condition, format...) ({       \
	int __ret_warn_on = !!(condition);  \
	if (unlikely(__ret_warn_on))        \
		__WARN_printf(format);      \
	unlikely(__ret_warn_on);            \
})
#endif
```

It can be figured out by getting assembler output. (like the following on Fedora 26)

```c
gcc -Wp,-MD,init/.main.o.d  -nostdinc -isystem /usr/lib/gcc/x86_64-redhat-linux/7/include \
-I/auto/homeX/caij9/src/repos/kernel/linux-stable/arch/x86/include \
-Iarch/x86/include/generated/uapi -Iarch/x86/include/generated \
-I/auto/homeX/caij9/src/repos/kernel/linux-stable/include -Iinclude \
-I/auto/homeX/caij9/src/repos/kernel/linux-stable/arch/x86/include/uapi \
-Iarch/x86/include/generated/uapi \
-I/auto/homeX/caij9/src/repos/kernel/linux-stable/include/uapi -Iinclude/generated/uapi \
-include /auto/homeX/caij9/src/repos/kernel/linux-stable/include/linux/kconfig.h \
-I/auto/homeX/caij9/src/repos/kernel/linux-stable/init -Iinit -D__KERNEL__ -Wall -Wundef \
-Wstrict-prototypes -Wno-trigraphs -fno-strict-aliasing -fno-common \
-Werror-implicit-function-declaration -Wno-format-security -std=gnu89 -fno-PIE \
-mno-sse -mno-mmx -mno-sse2 -mno-3dnow -mno-avx -m64 -falign-jumps=1 -falign-loops=1 -mno-80387 \
-mno-fp-ret-in-387 -mpreferred-stack-boundary=3 -mskip-rax-setup -mtune=generic -mno-red-zone \
-mcmodel=kernel -funit-at-a-time -maccumulate-outgoing-args \
-DCONFIG_AS_CFI=1 -DCONFIG_AS_CFI_SIGNAL_FRAME=1 -DCONFIG_AS_CFI_SECTIONS=1 \
-DCONFIG_AS_FXSAVEQ=1 -DCONFIG_AS_SSSE3=1 -DCONFIG_AS_CRC32=1 -DCONFIG_AS_AVX=1 \
-DCONFIG_AS_AVX2=1 -DCONFIG_AS_SHA1_NI=1 -DCONFIG_AS_SHA256_NI=1 -pipe \
-Wno-sign-compare -fno-asynchronous-unwind-tables -fno-delete-null-pointer-checks \
-Wno-maybe-uninitialized -Wno-frame-address -Wno-format-truncation -Wno-format-overflow \
-Wno-int-in-bool-context -O2 --param=allow-store-data-races=0 \
-DCC_HAVE_ASM_GOTO -Wframe-larger-than=2048 -fno-stack-protector \
-Wno-unused-but-set-variable -Wno-unused-const-variable -fno-omit-frame-pointer \
-fno-optimize-sibling-calls -fno-var-tracking-assignments -g -Wdeclaration-after-statement \
-Wno-pointer-sign -fno-strict-overflow -fconserve-stack -Werror=implicit-int \
-Werror=strict-prototypes -Werror=date-time \
-D"KBUILD_STR(s)=#s" -D"KBUILD_BASENAME=KBUILD_STR(main)" \
-D"KBUILD_MODNAME=KBUILD_STR(main)" -Wa,-aslh \
-c -o init/main.o init/main.c > <PATH_TO_OUTPUT>.s
```

>`-Wa`: Pass option to the assembler.
>
>`-aslh`: Ask assembler to include assembly code.

In this case, the string of the file path is allocated in section ".rodata.str1.8".

```
177                 .section .rodata.str1.8    <= section
178 001f 00         .align 8
179                .LC3:
180 0020 2F617574   .string "/auto/homeX/caij9/src/repos/kernel/builds/linux-4.4.y/jason_test/main.c"                <== string at offset 20h
180      6F2F686F
180      6D65322F
180      6361696A
180      392F7372
... ...
680 064d E8000000   call perf_event_init
680      00
681
682                .linefile 807"/auto/homeX/caij9/src/repos/kernel/linux-stable/arch/x86/include/asm/paravirt.h"1
695
696 0659 0FBAE009   btl $9,%eax
697 065d 7318       jnc .L113
698 065f 48C7C200   movq $.LC13,%rdx
698      000000
699 0666 BE550200   movl $597,%esi
699      00
700 066b 48C7C700   movq $.LC3,%rdi     <== passed as the 1st argument.
700      000000
701 0672 E8000000   call warn_slowpath_fmt
701      00
... ...
```

Note that the default options compiling `init/main.o` include option `-g`
which also leads to generate a string of the file path in section `.debug_str`.

[back](../)

