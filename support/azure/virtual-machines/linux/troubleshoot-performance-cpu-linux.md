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

Monitoring CPU utilization is straightforward. From a percentage of CPU utilization in top command output, to the more in-depth statistics reported by ps / sar, it is possible to accurately determine how much CPU power is being consumed and by what.
top is the first resource monitoring tool to provide an in-depth representation of CPU utilization. Here is a top report from a 2-processor VM:

### Obtain performance pointers

You can obtain performance CPU metric using the tools in the following table.

|Resource|Tool|
|---|---|
|CPU|`top`, `vmstat`, `sysstat`, `htop`, `mpstat`, `pidstat`|

The followng sections discuss CPU related metrics.

## CPU resource

A certain percentage of CPU is either used or not. Similarly, processes either spend time in CPU (such as 80 percent `usr` usage) or do not (such as 80 percent idle). The main tool to confirm CPU usage is `top`.

The `top` tool runs in interactive mode by default. It refreshes every second and shows processes as sorted by CPU usage:

```output
top - 06:33:18 up  4:10,  2 users,  load average: 2.26, 0.83, 0.32
Tasks: 193 total,   1 running, 192 sleeping,   0 stopped,   0 zombie
%Cpu(s):  1.2 us,  1.9 sy,  0.0 ni, 16.8 id, 79.7 wa,  0.3 hi,  0.2 si,  0.0 st
MiB Mem :   7697.8 total,   2597.6 free,    580.2 used,   4519.9 buff/cache
MiB Swap:      0.0 total,      0.0 free,      0.0 used.   6831.5 avail Mem

    PID USER      PR  NI    VIRT    RES    SHR S  %CPU  %MEM     TIME+ COMMAND
  28459 root      20   0    8476   2860   1844 D   2.7   0.0   0:01.58 dd
  28460 root      20   0    8476   2984   1984 D   2.7   0.0   0:01.55 dd
    861 root      20   0 1176588  30280   4524 S   0.7   0.4   1:26.21 auoms
  22338 root      20   0       0      0      0 D   0.7   0.0   0:00.27 kworker+
  28422 root      20   0   54400   4276   3652 R   0.7   0.1   0:00.17 top
    814 root      16  -4  131380   3316   1984 S   0.3   0.0   0:06.97 auditd
    817 root       0 -20  641596  26824   3656 S   0.3   0.3   0:34.73 auomsco+
   1073 root      20   0 1139916 135968  57960 S   0.3   1.7   0:48.70 wdavdae+
   1528 root      20   0 1071884  79200  51168 S   0.3   1.0   0:40.29 wdavdae+
   1535 omsagent  20   0  342040  66408   9972 S   0.3   0.8   0:43.01 omsagent
   2005 root      20   0  513792  33172  11812 D   0.3   0.4   0:45.47 platfor+
  28247 root      20   0       0      0      0 I   0.3   0.0   0:00.37 kworker+
      1 root      20   0  241300  14192   9060 S   0.0   0.2   0:10.62 systemd
      2 root      20   0       0      0      0 S   0.0   0.0   0:00.23 kthreadd
      3 root       0 -20       0      0      0 I   0.0   0.0   0:00.00 rcu_gp
      4 root       0 -20       0      0      0 I   0.0   0.0   0:00.00 rcu_par+

```

Now, look at the `dd` process line from that output:

```output
PID USER      PR  NI    VIRT    RES    SHR S  %CPU  %MEM     TIME+ COMMAND
28459 root      20   0    8476   2860   1844 D   2.7   0.0   0:01.58 dd
```


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

process state output
```
[root@rhel8 ~]# ps -eo pcpu,pmem,pid,user,args | sort -r -k1 | head -6
%CPU %MEM     PID USER     COMMAND
25.5  0.0   28984 root     dd if=/dev/zero of=/cases/file1.bin bs=1M count=2048 status=progress
17.5  0.0   28983 root     dd if=/dev/zero of=/cases/file2.bin bs=1M count=2048 status=progress
 0.5  0.3     861 root     /opt/microsoft/auoms/bin/auoms
 0.3  1.7    1073 root     /opt/microsoft/mdatp/sbin/wdavdaemon
 0.3  0.4    2005 root     /usr/libexec/platform-python -u bin/WALinuxAgent-2.11.1.4-py3.9.egg -run-exthandlers


```
> [!NOTE]
> - You can display per-CPU usage in the `top` tool by selecting <kbd>1</kbd>.
>
> - The `top` tool displays a total usage of more than 100 percent if the process is multithreaded and spans more than one CPU.

Another useful reference is load average. The load average shows an average system load in 1-minute, 5-minute, and 15-minute intervals. The value indicates the level of load of the system. Interpreting this value depends on the number of CPUs that are available. For example, if the load average is 2 on a one-CPU system, then the system is so loaded that the processes start to queue up. If there's a load average of 2 on a four-CPU system, there's about 50 percent overall CPU usage.

> [!NOTE]
> You can quickly obtain the CPU count by running the `nproc` command.

In the previous example, the load average is at 1.04. This is a two-CPU system, meaning that there's about 50 percent CPU usage. You can verify this result if you notice the 48.5 percent idle CPU value. (In the `top` command output, the idle CPU value is shown before the `id` label.)

Use the load average as a quick overview of how the system is performing.

Run the `uptime` command to obtain the load average.
