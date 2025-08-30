### Changing static_prio is the same as changing NICE value

The systam call of changing nice

`include/linux/syscall.c`
```c=738
/* kernel/sys.c */
asmlinkage long sys_setpriority(int which, int who, int niceval);
asmlinkage long sys_getpriority(int which, int who);
```

Implementation of the system call

`kernel/sys.c`
```c=322
SYSCALL_DEFINE2(getpriority, int, which, int, who)
{
    ...
        niceval = nice_to_rlimit(task_nice(p));
    ...
}
```

```c=203
SYSCALL_DEFINE3(setpriority, int, which, int, who, int, niceval)
{
    ...
        error = set_one_prio(p, niceval, error);
    ...
    }
```

```c=203
static int set_one_prio(struct task_struct *p, int niceval, int error)
{
    ...
	set_user_nice(p, niceval);
    ...
}
```

The conversion between static_prio and nice value

`include/linux/sched.h`

```c=1845
/**
 * task_nice - return the nice value of a given task.
 * @p: the task in question.
 *
 * Return: The nice value [ -20 ... 0 ... 19 ].
 */
static inline int task_nice(const struct task_struct *p)
{
	return PRIO_TO_NICE((p)->static_prio);
}
```

`kernel/sched/core.c`
```c=6956
void set_user_nice(struct task_struct *p, long nice)
{
    ...
    p->static_prio = NICE_TO_PRIO(nice);
    ...
    }
```

### How nice affect vruntime?

[`set_user_nice`](https://elixir.bootlin.com/linux/latest/source/kernel/sched/core.c#L7214)
-> [`set_load_weight`](https://elixir.bootlin.com/linux/latest/source/kernel/sched/core.c#L1304)
-> `load->weight = scale_load(sched_prio_to_weight[prio]);`

[`update_curr`](https://elixir.bootlin.com/linux/latest/source/kernel/sched/fair.c#L1135)
-> [`calc_delta_fair`](https://elixir.bootlin.com/linux/latest/source/kernel/sched/fair.c#L296)
-> [`__calc_delta` ](https://elixir.bootlin.com/linux/latest/source/kernel/sched/fair.c#L266)

```c=224
/*
 * delta_exec * weight / lw.weight
 *   OR
 * (delta_exec * (weight * lw->inv_weight)) >> WMULT_SHIFT
 *
 * Either weight := NICE_0_LOAD and lw \e sched_prio_to_wmult[], in which case
 * we're guaranteed shift stays positive because inv_weight is guaranteed to
 * fit 32 bits, and NICE_0_LOAD gives another 10 bits; therefore shift >= 22.
 *
 * Or, weight =< lw.weight (because lw.weight is the runqueue weight), thus
 * weight/lw.weight <= 1, and therefore our shift will also be positive.
 */
static u64 __calc_delta(u64 delta_exec, unsigned long weight, struct load_weight *lw)
```

```c=10911
/*
 * Nice levels are multiplicative, with a gentle 10% change for every
 * nice level changed. I.e. when a CPU-bound task goes from nice 0 to
 * nice 1, it will get ~10% less CPU time than another CPU-bound task
 * that remained on nice 0.
 *
 * The "10% effect" is relative and cumulative: from _any_ nice level,
 * if you go up 1 level, it's -10% CPU usage, if you go down 1 level
 * it's +10% CPU usage. (to achieve that we use a multiplier of 1.25.
 * If a task goes up by ~10% and another task goes down by ~10% then
 * the relative distance between them is ~25%.)
 */
const int sched_prio_to_weight[40] = {
 /* -20 */     88761,     71755,     56483,     46273,     36291,
 /* -15 */     29154,     23254,     18705,     14949,     11916,
 /* -10 */      9548,      7620,      6100,      4904,      3906,
 /*  -5 */      3121,      2501,      1991,      1586,      1277,
 /*   0 */      1024,       820,       655,       526,       423,
 /*   5 */       335,       272,       215,       172,       137,
 /*  10 */       110,        87,        70,        56,        45,
 /*  15 */        36,        29,        23,        18,        15,
};
```

### Modify the vruntime

vruntime, delta_exec, ... are calculated in terms of nanoseconds(ns)

`include/linux/sched.h`

```c=561
int fixed_offset;
```

`kernel/sched/fair.c`

```c=852
if (curr->fixed_offset != 0) {
    printk("delta_exec: %lld, fixed_offset: %lld\n", delta_exec, curr->fixed_offset);
}

custom_offset = min(delta_exec - 10000, curr->fixed_offset);

curr->vruntime += calc_delta_fair(delta_exec, curr) - custom_offset;
```

`kernel/fork.c`

```c=2444
p->se->fixed_offset = 0;
```

`set_process_priority.c`

```c=17
current->se.fixed_offset = (prio - 100) * 50000;
```

### Create stress for scheduler

Using only one cpu on virtual machine to create "stress" for scheduler

#### Use `yes` command

```bash
$ yes > /dev/null
```

#### Use `stress_creator.c`

```c=
#include <stdio.h>
#include <stdlib.h>

#define TOTAL_ITERATION_NUM 10000000000

int main(int argc, char** argv) {
    for (int i = 1; i <= TOTAL_ITERATION_NUM; i++)
        rand();
    return 0;
}
```

## Result

### CPU at low load

```
The process spent 1079174 uses to execute when priority is not adjusted.
The process spent 1155926 uses to execute when priority is 101.
The process spent 1047048 uses to execute when priority is 102.
The process spent 1066393 uses to execute when priority is 103.
The process spent 1036407 uses to execute when priority is 104.
The process spent 982956 uses to execute when priority is 105.
The process spent 572699 uses to execute when priority is 106.
The process spent 533393 uses to execute when priority is 107.
The process spent 563944 uses to execute when priority is 108.
The process spent 535227 uses to execute when priority is 109.
The process spent 591870 uses to execute when priority is 110.
The process spent 524807 uses to execute when priority is 111.
The process spent 532673 uses to execute when priority is 112.
The process spent 522444 uses to execute when priority is 113.
The process spent 565203 uses to execute when priority is 114.
The process spent 527621 uses to execute when priority is 115.
The process spent 548666 uses to execute when priority is 116.
The process spent 525487 uses to execute when priority is 117.
The process spent 529936 uses to execute when priority is 118.
The process spent 641523 uses to execute when priority is 119.
The process spent 525272 uses to execute when priority is 120.
The process spent 532524 uses to execute when priority is 121.
The process spent 525018 uses to execute when priority is 122.
The process spent 546420 uses to execute when priority is 123.
The process spent 548799 uses to execute when priority is 124.
The process spent 530139 uses to execute when priority is 125.
The process spent 522808 uses to execute when priority is 126.
The process spent 528194 uses to execute when priority is 127.
The process spent 564610 uses to execute when priority is 128.
The process spent 525043 uses to execute when priority is 129.
The process spent 525323 uses to execute when priority is 130.
The process spent 525028 uses to execute when priority is 131.
The process spent 521975 uses to execute when priority is 132.
The process spent 569005 uses to execute when priority is 133.
The process spent 525671 uses to execute when priority is 134.
The process spent 519912 uses to execute when priority is 135.
The process spent 525946 uses to execute when priority is 136.
The process spent 632583 uses to execute when priority is 137.
The process spent 542963 uses to execute when priority is 138.
The process spent 526273 uses to execute when priority is 139.
```


### CPU under load

```
The process spent 1668031 uses to execute when priority is not adjusted.
The process spent 1602958 uses to execute when priority is 101.
The process spent 1632944 uses to execute when priority is 102.
The process spent 1539541 uses to execute when priority is 103.
The process spent 1371543 uses to execute when priority is 104.
The process spent 1016420 uses to execute when priority is 105.
The process spent 1027287 uses to execute when priority is 106.
The process spent 983335 uses to execute when priority is 107.
The process spent 1017136 uses to execute when priority is 108.
The process spent 969084 uses to execute when priority is 109.
The process spent 998883 uses to execute when priority is 110.
The process spent 963568 uses to execute when priority is 111.
The process spent 935731 uses to execute when priority is 112.
The process spent 978286 uses to execute when priority is 113.
The process spent 929712 uses to execute when priority is 114.
The process spent 1027544 uses to execute when priority is 115.
The process spent 908916 uses to execute when priority is 116.
The process spent 904867 uses to execute when priority is 117.
The process spent 935364 uses to execute when priority is 118.
The process spent 881940 uses to execute when priority is 119.
The process spent 925743 uses to execute when priority is 120.
The process spent 882755 uses to execute when priority is 121.
The process spent 876832 uses to execute when priority is 122.
The process spent 976451 uses to execute when priority is 123.
The process spent 852702 uses to execute when priority is 124.
The process spent 892315 uses to execute when priority is 125.
The process spent 842311 uses to execute when priority is 126.
The process spent 837832 uses to execute when priority is 127.
The process spent 855269 uses to execute when priority is 128.
The process spent 828921 uses to execute when priority is 129.
The process spent 808374 uses to execute when priority is 130.
The process spent 959327 uses to execute when priority is 131.
The process spent 816900 uses to execute when priority is 132.
The process spent 799565 uses to execute when priority is 133.
The process spent 813625 uses to execute when priority is 134.
The process spent 784343 uses to execute when priority is 135.
The process spent 773005 uses to execute when priority is 136.
The process spent 813271 uses to execute when priority is 137.
The process spent 747597 uses to execute when priority is 138.
The process spent 761395 uses to execute when priority is 139.
```
