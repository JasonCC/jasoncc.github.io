---
layout : default
---

# A Silly FPU Issue

- [A Silly FPU Issue](https://lkml.org/lkml/2017/3/21/896)

# Test-case for Reproducing FPU Issue

## The abuse program.

```
#include <sys/types.h>
#include <stdio.h>
#include <unistd.h>
#include <signal.h>
#include <stdlib.h>
#include <string.h>
#include <math.h>

int main(void) {
        pid_t pid;
	pid = getpid();
	printf("%u: pid / PI = %f\n", pid, pid / M_PI);
	kill(pid, SIGSEGV);
	return 0;
}
```

## Piping Cores to a Script `getcore.sh`.

```
#!/bin/bash
/bin/gzip > "$1"
```

## Modify core_pattern and Test it.

```

#!/bin/bash

mkdir -p /var/core/core2
echo "|/var/home/sysadmin/getcore.sh /var/core/core2/core.%e.%p.%t.gz" > /proc/sys/kernel/core_pattern

CORE_DIR="/var/core/core2"

clean_cores() {
  find ${CORE_DIR} -name "core.abuse.*" | xargs rm
}

ROUND=0

run_test_one() {
  ROUND=$[$ROUND+1]
  echo "[Testing FPU Abuse] Round ${ROUND}"
  echo
  for i in `seq 1 1 10000`; do ./abuse & echo -n; done
  echo "[Tesing FPU Abuse] Round ${ROUND} finished, waiting for 10 seconds ..."
  sleep 10
  echo "[Tesing FPU Abuse] Cleaning cores ..."
  clean_cores
  echo "[Tesing FPU Abuse] Cleaning cores, done"
}

run_test() {
  echo "Started " `date` > test_abuse.log
  ROUND=0
  while true
  do
    run_test_one
  done
}

echo "Started " `date` > test_abuse.log
run_test_one
#run_test
```

# References

- [big x86 FPU code rewrite](https://lwn.net/Articles/643235/)
- [Debug hints for fpu state NULL pointer dereference on context switch during core dump in 3.0.101](https://lkml.org/lkml/2016/12/19/434)
- [Racy manipulation of task_struct->flags in cgroups code causes hard to reproduce kernel panics](https://lkml.org/lkml/2014/9/19/230)



[back](../)


