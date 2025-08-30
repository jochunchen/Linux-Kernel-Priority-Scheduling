## Environment

Ubuntu Server 22.04 in VMWare
* Bundled Kernel version: `5.15.0`
* Target Kernel version: `5.15.138`

## Compile the kernel


```
make mrproper

sudo make -j6

sudo make modules_install -j6

sudo make install -j6

sudo update-grub

reboot
```


## Add system call

#### `arch/x86/entry/syscalls/syscall_64.tbl`

```clike=373
449 common  set_process_priority	sys_set_process_priority
```

#### `arch/x86/entry/syscalls/syscall_32.tbl`

```clike=456
449 i386	set_process_priority	sys_set_process_priority
```

#### `include/linux/syscalls.h`

```c=1056
/* kernel/set_process_priority.c */
asmlinkage long sys_set_process_priority(int prio);
```

<!-- About __user -->

#### `include/uapi/asm-generic/unistd.h`

```c=883
#define __NR_set_process_priority 449
__SYSCALL(__NR_set_process_priority, sys_set_process_priority)

#undef __NR_syscalls
#define __NR_syscalls 450
```

<!-- #### `arch/x86/kernel/Makefile` -->
#### `kernel/Makefile`

```c=15
obj-y += set_process_priority.o
```

#### `include/linux/sched.h`

```cl=1492
    int fixed_priority;
    
    /*
     * New fields for task_struct should be added above here, so that
     * they are included in the randomized portion of task_struct.
     */
    randomized_struct_fields_end
```

#### `set_process_priority.c`

```c=
#include <stdlib.h>
#include <linux/syscalls.h>

SYSCALL_DEFINE1(set_process_priority, int, prio)
{
    struct task_struct *p;

    if (prio < 101 || prio > 139) {
        /* Error */
        return 1;
    }

    p = current;

    current->fixed_priority = prio;

    /* Seccuss */
    return 0;
}
```

## Tracing the context switch in 5.X Kernel

#### `kernel/sched/core.c`

```c=6448
asmlinkage __visible void __sched schedule(void)
```

* Disable preemption
* Call `__schedule`

#### `kernel/sched/core.c`

The main scheduler function

```c=6250
static void __sched notrace __schedule(unsigned int sched_mode)
```

Decide next

```c=6336
next = pick_next_task(rq, prev, &rf);
```

Call context_switch

```c=6372
rq = context_switch(rq, prev, next, &rf);
```

#### `kernel/sched/core.c`

```c=4974
static __always_inline struct rq *
context_switch(struct rq *rq, struct task_struct *prev,
	       struct task_struct *next, struct rq_flags *rf)
```

```c=5026
switch_to(prev, next, prev);
```

#### `arch/x86/include/asm/switch_to.h`

```c=47
#define switch_to(prev, next, last)					\
do {									\
	((last) = __switch_to_asm((prev), (next)));			\
} while (0)
```

#### `arch/x86/entry/entry_64.S`

```c=230
SYM_FUNC_START(__switch_to_asm)
```

```c=268
jmp	__switch_to
```

#### `arch/x86/kernel/process_64.c`

```c=556
__visible __notrace_funcgraph struct task_struct *
__switch_to(struct task_struct *prev_p, struct task_struct *next_p)
```

<!-- 
Pick next task

core.c L5744

Where does it set static_prio?
Add static_prio = p->fixed_priority during context switch

L1492 `static __latent_entropy struct task_struct *copy_process(` -->


### Change the `static_prio` when context switch

In the end of `arch/x86/kernel/process_64.c` `__switch_to`
or `kernel/sched/core.c` after L5026 `switch_to(prev, next, prev);`

```c=661
	/*
	 * If the value of the fixed_priority field is 0,
	 * do not need to change the priority of the process.
	 * Using current here is the same as next_p,
	 * since this_cpu_write(current_task, next_p); has
	 * assigned next_p to current.
	 */
	if(next_p->fixed_priority != 0
           && next->pid!=0
           && next->fixed_priority>100
           && next->fixed_priority<140) {
		next_p->static_prio = next_p->fixed_priority;
	}
```

## User space program

`test_sched.c`

```c=
#include <stdio.h>
#include <stdlib.h>
#include <sys/syscall.h>
#include <sys/time.h>
#include <time.h>
#include <unistd.h>

#define TOTAL_ITERATION_NUM 1000000000
#define NUM_OF_PRIORITIES_TESTED 40

int my_set_process_priority(int priority) {
    return (int)syscall(449, priority);
}

int main(int argc, char** argv) {
    int index = 0;
    int priority, i;
    struct timeval start[NUM_OF_PRIORITIES_TESTED], end[NUM_OF_PRIORITIES_TESTED];

    gettimeofday(&start[index], NULL);  // begin
    for (i = 1; i <= TOTAL_ITERATION_NUM; i++)
        rand();
    gettimeofday(&end[index], NULL);  // end

    /*================================================================================*/

    for (index = 1, priority = 101; priority <= 139; ++priority, ++index) {
        if (my_set_process_priority(priority))
            printf("Cannot set priority %d.\n", priority);

        gettimeofday(&start[index], NULL);  // begin
        for (i = 1; i <= TOTAL_ITERATION_NUM; i++)
            rand();
        gettimeofday(&end[index], NULL);  // end
    }

    /*================================================================================*/

    printf("The process spent %ld uses to execute when priority is not adjusted.\n",
           ((end[0].tv_sec * 1000000 + end[0].tv_usec) - (start[0].tv_sec * 1000000 + start[0].tv_usec)));

    for (i = 1; i <= NUM_OF_PRIORITIES_TESTED - 1; i++)
        printf("The process spent %ld uses to execute when priority is %d.\n",
               ((end[i].tv_sec * 1000000 + end[i].tv_usec) - (start[i].tv_sec * 1000000 + start[i].tv_usec)), i + 100);
}
```

### Compile and run the program

```shell
gcc test_sched.c -o test_sched

./test_sched
```

## Result

We increase the loop by 10x to meets the needs of at least one context switch per priority. If <1 context switch per priority the fixed_priority won't be assign to static_prio.

```
The process spent 6043467 uses to execute when priority is not adjusted.
The process spent 5857432 uses to execute when priority is 101.
The process spent 5772383 uses to execute when priority is 102.
The process spent 5812352 uses to execute when priority is 103.
The process spent 6022521 uses to execute when priority is 104.
The process spent 6074865 uses to execute when priority is 105.
The process spent 5821207 uses to execute when priority is 106.
The process spent 5824510 uses to execute when priority is 107.
The process spent 5871532 uses to execute when priority is 108.
The process spent 5794556 uses to execute when priority is 109.
The process spent 5839980 uses to execute when priority is 110.
The process spent 5846327 uses to execute when priority is 111.
The process spent 5890920 uses to execute when priority is 112.
The process spent 5847760 uses to execute when priority is 113.
The process spent 5836641 uses to execute when priority is 114.
The process spent 5815150 uses to execute when priority is 115.
The process spent 5810396 uses to execute when priority is 116.
The process spent 5961219 uses to execute when priority is 117.
The process spent 5926533 uses to execute when priority is 118.
The process spent 5938364 uses to execute when priority is 119.
The process spent 5977416 uses to execute when priority is 120.
The process spent 5840564 uses to execute when priority is 121.
The process spent 5865859 uses to execute when priority is 122.
The process spent 6014342 uses to execute when priority is 123.
The process spent 5854424 uses to execute when priority is 124.
The process spent 5834999 uses to execute when priority is 125.
The process spent 5980547 uses to execute when priority is 126.
The process spent 6200714 uses to execute when priority is 127.
The process spent 5999795 uses to execute when priority is 128.
The process spent 6216224 uses to execute when priority is 129.
The process spent 6199692 uses to execute when priority is 130.
The process spent 6123536 uses to execute when priority is 131.
The process spent 6021712 uses to execute when priority is 132.
The process spent 6015517 uses to execute when priority is 133.
The process spent 6215729 uses to execute when priority is 134.
The process spent 6115290 uses to execute when priority is 135.
The process spent 6321504 uses to execute when priority is 136.
The process spent 6125948 uses to execute when priority is 137.
The process spent 6084739 uses to execute when priority is 138.
The process spent 5974991 uses to execute when priority is 139.
```
