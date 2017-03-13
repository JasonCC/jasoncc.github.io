How Many Forks Will Be Invoked in The following Code?

```
#include <stdlib.h>
int main(void) {
	system("bash -c pidof");
	return 0;
}
```

## Figuring it out by using `strace`

`strace -f <prog>`

## Figuring it out by useing `perf`

`sudo perf record -e 'syscalls:sys_enter_*' -e 'sched:sched_*' <prog>`
`sudo perf script`

`sudo perf record -e 'sched:sched_process_fork' -a`
`sudo perf script`

`sudo perf record -e 'sched:sched_process_fork' -e 'sched:sched_process_exec' -a`
`sudo perf script`

## Answer

It only fork once, which is called by `system(3)`.

