---
title: "Reading memory information under Linux systems"
date: 2013-06-15T13:24:00
summary: "Some bits to understand the Linux memory numbers"
---

Introduction
------------

Just a few notes about how to get informations how memory on linux, and common tools to read them.

Consultation
------------

It's possible to read about memory use under linux by multiple ways: */proc/meminfo*, *free*, *vmstat*, *top*...

Those tools allow to get information about physical memory as virtual memory.

Firstly, we can get general infos reading */proc/meminfo* or using *free*:

```sh
$ cat /proc/meminfo
MemTotal:        8058060 kB
MemFree:          757660 kB
Buffers:          401752 kB
Cached:          2958380 kB
SwapCached:       162336 kB
```

* MemTotal: Total usable ram (i.e. physical ram minus a few reserved bits and the kernel binary code)
* MemFree: Is sum of LowFree+HighFree (overall stat)
* Buffers: Memory in buffer cache
* Cached: Memory in the pagecache (diskcache) minus SwapCache (Doesn't include SwapCached though)
* SwapCached: Memory that once was swapped out, is swapped back in but still also is in the swapfile

```sh
$ free -m
             total       used       free     shared    buffers     cached
Mem:          7869       7106        763          0        392       2893
-/+ buffers/cache:       3820       4048
Swap:         8073        601       7472
```

Then, we have to understand the *top* output:

```sh
top - 11:11:11 up 23 days, 17:37, 10 users,  load average: 0.18, 0.16, 0.18
Tasks: 356 total,   1 running, 307 sleeping,   0 stopped,  48 zombie
%Cpu(s):  2.3 us,  1.4 sy,  0.0 ni, 96.3 id,  0.0 wa,  0.0 hi,  0.0 si,  0.0 st
KiB Mem:  16314188 total, 13358168 used,  2956020 free,   532416 buffers
KiB Swap: 16658428 total,      444 used, 16657984 free.  9372752 cached Mem

  PID USER      PR  NI    VIRT    RES    SHR S  %CPU %MEM     TIME+ COMMAND
18228 mycroft   20   0 1469652 398888  93480 S   5.0  2.4  81:13.74 chromium-browse
18183 mycroft   20   0 1426144 249424  94916 S   4.3  1.5  77:29.87 chromium-browse
 2124 root      20   0  365924  90976  56240 S   2.6  0.6 488:25.19 Xorg
18437 mycroft   20   0 1336024 364608  37372 S   1.7  2.2  34:40.92 chromium-browse
```

* Virtual: Virtual memory allocated, including all code, data and shared libraries
* Resident: Non swapped physical memory
* Shared: Amount of memory available potentially shared with other proccesses


*vmstat* reports virtual memory statistics:

```sh
$ vmstat 5
procs -----------memory---------- ---swap-- -----io---- -system-- ----cpu----
 r  b   swpd   free   buff  cache   si   so    bi    bo   in   cs us sy id wa
 1  0 615740 831376 401696 2970016    2    2    87   139   41   39  4  4 92  1
 0  0 615740 833200 401696 2967316    0    0     0    15  853 1576  3  2 95  0
 0  0 615740 832860 401696 2967932    0    0     0    83  704 1239  3  1 96  0
 0  0 615740 837280 401696 2963756    0    0     0    92  864 1362  2  1 97  0
```


Les noyaux Linux récents fournissent un mécanisme pour que le noyau libère certains éléments stockés en mémoire à la demande, ce qui permet de libérer une partie de la mémoire. Pour cela, il faut utiliser le fichier _/proc/sys/vm/drop_caches_:

Recent linux kernels provide a mechanism to drop the page cache, inodes and dentry caches, which can free up a lot of memory, using virtual file */proc/sys/vm/drop_caches*:

To free pagecache:
```sh
# echo 1 > /proc/sys/vm/drop_caches
```

To free dentries and inodes:
```sh
# echo 2 > /proc/sys/vm/drop_caches
```

To free pagecache, dentries and inodes:
```sh
# echo 3 > /proc/sys/vm/drop_caches
```

These operations are non destructive and will only free non used things. Think to *sync* to flush I/O buffers before, and you may free more memory.

[Source](http://linux-mm.org/Drop_Caches)

## Finding out what processes are swapping

More recently, I needed to find out what processes had some memory stored in swap (which is way slower than RAM).

The swap information for each processes can also be found in /proc/<pid>/status:

```sh
$ cat /proc/$$/status | grep VmSwap
VmSwap:        0 kB
```

## A few more links about memory

* [What Every Programmer Should Know About Memory](http://www.akkadia.org/drepper/cpumemory.pdf)
* [Understanding The Linux Virtual Memory Manager](https://www.kernel.org/doc/gorman/pdf/understand.pdf)
