---
title: Troubleshoot performance CPU issues in Linux
description: Troubleshoot CPU performance issues on Linux virtual machines in Azure.
ms.service: virtual-machines
ms.custom: sap:VM Performance, linux-related-content
author: paulxjp
ms.author: jianpingxi
---
# Troubleshoot performance CPU issues in Linux

**Applies to:** :heavy_check_mark: Linux VMs

## Introduction to CPU

Monitoring CPU utilization is straightforward. From a percentage of CPU utilization in top utility output, to the more in-depth statistics reported by ps or sar command, it is possible to accurately determine how much CPU power is being consumed and by what.

The top utility is the first resource monitoring tool to provide an in-depth representation of CPU utilization, it gives you a real-time look at what’s going on with the server. Here is a top report from a 2-processor VM:

```output
top - 03:12:38 up  1:53,  3 users,  load average: 1.72, 0.62, 0.25
Tasks: 198 total,   1 running, 197 sleeping,   0 stopped,   0 zombie
%Cpu(s):  0.3 us,  1.4 sy,  0.0 ni, 15.2 id, 82.5 wa,  0.2 hi,  0.3 si,  0.0 st
MiB Mem :   7697.8 total,   3672.6 free,    576.6 used,   3448.5 buff/cache
MiB Swap:      0.0 total,      0.0 free,      0.0 used.   6838.4 avail Mem

    PID USER      PR  NI    VIRT    RES    SHR S  %CPU  %MEM     TIME+ COMMAND
  17038 root      20   0    8476   2984   1984 D   3.7   0.0   0:00.77 dd
  17039 root      20   0    8476   2928   1916 D   3.7   0.0   0:01.14 dd
  16979 root      20   0       0      0      0 D   1.3   0.0   0:00.14 kworker+
    858 root      20   0 1176440  31088   4580 S   0.7   0.4   0:43.16 auoms
   1679 omsagent  20   0  350232  68976   9944 S   0.7   0.9   0:18.52 omsagent
    521 root      20   0       0      0      0 S   0.3   0.0   0:00.05 xfsaild+
  16018 root      20   0       0      0      0 I   0.3   0.0   0:00.29 kworker+
  16933 root      20   0   54400   4384   3756 R   0.3   0.1   0:00.21 top
      1 root      20   0  175948  14356   9180 S   0.0   0.2   0:04.02 systemd
      2 root      20   0       0      0      0 S   0.0   0.0   0:00.10 kthreadd
      3 root       0 -20       0      0      0 I   0.0   0.0   0:00.00 rcu_gp
      4 root       0 -20       0      0      0 I   0.0   0.0   0:00.00 rcu_par+
      5 root       0 -20       0      0      0 I   0.0   0.0   0:00.00 slub_fl+
      7 root       0 -20       0      0      0 I   0.0   0.0   0:00.00 kworker+
     10 root       0 -20       0      0      0 I   0.0   0.0   0:00.00 mm_perc+
     11 root      20   0       0      0      0 S   0.0   0.0   0:00.00 rcu_tas+
     12 root      20   0       0      0      0 S   0.0   0.0   0:00.00 rcu_tas+

```

Some things to look for in this view would be the load average (displayed on the right side of the top row), and the value of the following for each CPU:

**us**: This percentage represents the amount of CPU consumed by user processes.

**sy**: This percentage represents the amount of CPU consumed by system processes.

**id**: This percentage represents how idle each CPU is.

**wa**: This percentage represents the percentage of CPU time spent waiting for I/O operations to complete.

**Tasks**

Tasks: This section provides an overview of the total number of processes currently managed by the system.

**total**: Indicates the total count of processes currently being tracked by the system.

**running**: Represents the number of processes currently actively using CPU time.

**zombie**: Indicates processes that have completed execution but still have an entry in the process table.

Now, look at the `dd` processed line from above output:

```output
PID USER      PR  NI    VIRT    RES    SHR S  %CPU  %MEM     TIME+ COMMAND
17038 root      20   0    8476   2984   1984 D   3.7   0.0   0:00.77 dd
17039 root      20   0    8476   2928   1916 D   3.7   0.0   0:01.14 dd
```

Meanwhile listing the Top 5 CPU consuming processes:
```
[root@rhel8 ~]# ps -eo pcpu,pmem,pid,user,args | sort -r -k1 | head -6
%CPU %MEM     PID USER     COMMAND
33.3  0.0   17039 root     dd if=/dev/zero of=/cases/file1.bin bs=1M count=2048 status=progress
21.0  0.0   17038 root     dd if=/dev/zero of=/cases/file2.bin bs=1M count=2048 status=progress
 0.6  0.3     858 root     /opt/microsoft/auoms/bin/auoms
 0.3  1.7    1057 root     /opt/microsoft/mdatp/sbin/wdavdaemon
 0.3  0.4    2034 root     /usr/libexec/platform-python -u bin/WALinuxAgent-2.11.1.4-py3.9.egg -run-exthandlers
```

### Obtain CPU metric

You can obtain performance CPU metric using the tools in the following table.

|Resource|Tool|
|---|---|
|CPU|`top`, `vmstat`, `sysstat`, `htop`, `mpstat`, `pidstat`|

The followng sections discuss CPU related metrics.

## CPU resource

The utilization of a CPU is mainly dependent on which resource is trying to access it. A scheduler exists in the kernel which is responsible for scheduling resources, mainly two and those are threads (single or multi) and interrupts. The scheduler gives different priorities to the different resources. Below list explain the priorities from highest to lowest.

Hardware Interrupts - These are requests created by hardware on the system to process data. This interrupt does this without waiting for current program to finish. It is unconditional and immediate. For example, a key stroke, mouse movement, a NIC may signal that a packet has been received.

Soft Interrupts - These are kernel software interrupts to do maintenance of the kernel. For example, the kernel clock tick thread is a soft interrupt. On a regular interval it checks and make sure that a process has not passed its allotted time on a processor.

Real Time Threads - A real time process may come on the CPU and preempt (or “kick off) the kernel..

Kernel Threads - A kernel thread is a kernel entity, like processes and interrupt handlers; it is the entity handled by the system scheduler. Kernel-level threads are handled by the operating system directly and the thread management is done by the kernel.

User Threads - This space is often referred to as “user land” and all software applications run in the user space. This space has the lowest priority in the kernel scheduling mechanism. In order to understand how the kernel manages these different resources, we need understand some key concepts such as context switches, run queues, and utilization.


vmstat output:
```
[root@rhel8 ~]# vmstat 2
procs -----------memory---------- ---swap-- -----io---- -system-- ------cpu-----
 r  b   swpd   free   buff  cache   si   so    bi    bo   in   cs us sy id wa st
 2  3      0 3639696   9500 3634812    0    0    43   326   45  357  2  1 96  1  0
 0  3      0 3452428   9500 3821728    0    0     0 71700  127  972  1  2  1 96  0
 1  3      0 3266588   9500 4011400    0    0     0 68612  144 1266  2  4  6 88  0
 0  3      0 3096384   9500 4181596    0    0     0 107520  114  952  1  2  4 93  0
 0  3      0 2931908   9500 4346168    0    0     0 74770   81  759  0  1  0 98  0
 1  2      0 2765040   9500 4512992    0    0     0 61442   73  756  0  2  4 94  0
 0  3      0 2608596   9500 4669520    0    0     0 74758  111  862  1  1  6 92  0
 0  3      0 2440972   9500 4836968    0    0     0 108544   91  789  0  2  1 97  0

```


> [!NOTE]
> - You can display per-CPU usage in the `top` tool by selecting <kbd>1</kbd>.
>
> - The `top` tool displays a total usage of more than 100 percent if the process is multithreaded and spans more than one CPU.

Another useful reference is load average. The load average shows an average system load in 1-minute, 5-minute, and 15-minute intervals. The value indicates the level of load of the system. Interpreting this value depends on the number of CPUs that are available. For example, if the load average is 2 on a one-CPU system, then the system is so loaded that the processes start to queue up. If there's a load average of 2 on a four-CPU system, there's about 50 percent overall CPU usage.

> [!NOTE]
> You can quickly obtain the CPU count by running the `nproc` command.
```
[root@rhel8 ~]# nproc
2
```
>

In the previous example, the load average is at 2.26. This is a two-CPU system, meaning that the system load is approaching full. You can verify this result if you notice the 16.8 percent idle CPU value. (In the `top` command output, the idle CPU value is shown before the `id` label.)

Use the load average as a quick overview of how the system is performing.

Run the `uptime` command to obtain the load average.
