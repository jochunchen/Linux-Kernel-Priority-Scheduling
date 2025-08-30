# Linux-Kernel-Priority-Scheduling
This repository contains the source code and documentation for two projects focused on modifying the Linux kernel's scheduler. The projects were completed on an Ubuntu Server environment, demonstrating the ability to compile a custom kernel, add new system calls, and alter core scheduling behavior to meet specific performance goals.

## Project 1: Modifying static_prio
This project's goal was to add a new system call, set_process_priority, to allow a user-space program to directly modify a process's static priority. The static_prio value is a key parameter used by the scheduler to determine a process's CPU share.

### Key Contributions
- Kernel Compilation: Successfully compiled and booted a custom Linux kernel from source.
- New System Call: Implemented a new system call (sys_set_process_priority) and integrated it into the kernel's system call table for both 32-bit and 64-bit architectures.
- Scheduler Integration: Modified the __switch_to function to ensure that a process's static_prio is updated with the new value during a context switch.
- Testing: Developed a user-space program to test the new system call and measure its effect on process execution time under varying priority levels.

## Project 2: Modifying vruntime
This project expanded upon the first by taking a more direct approach to scheduler manipulation. Instead of simply changing the static priority, this project focused on modifying the virtual runtime (vruntime), a core metric used by the Completely Fair Scheduler (CFS) to track how much CPU time a process has received.

### Key Contributions

- Manipulating vruntime: Implemented logic within the kernel/sched/fair.c file to directly subtract a custom offset from a process's vruntime. This effectively gives the process a "head start" in the scheduling queue, granting it more CPU time.
- New task_struct Field: Added a fixed_offset field to the task_struct to store the value used for the vruntime adjustment.
- Enhanced System Call: Modified the set_process_priority system call from Project 1 to set this new fixed_offset field instead of static_prio.
- Stress Testing: Conducted tests under high CPU load to demonstrate the significant performance benefits of directly manipulating vruntime compared to the standard nice value.
