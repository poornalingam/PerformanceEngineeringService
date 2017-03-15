# System Profiling and Tuning
- [Application profiling](./Application-Performance-Analysis.md)
- [CPU profiling](./CPU-Profiling.md)
- [Memory profiling](./Memory-profiling.md)
  - [Memory Basics](#memory-basics)
  - [Methods of analysis](#methodology)
    - [Tools method](#tools-method)
    - [USE method](#use-method)
    - [Workload Characterizing](#Characterizing)
    - [Leak analysis](#leak-analysis)
    - [Static performance tuning](#static-performance-tuning)
  - [Analysis](#analysis)
  - [Tuning](#tuning)
- [File Systems profiling](./filesystem-profiling.md)
- [Disk profiling](./disk-profiling.md)
- [Network profiling](./Network-profiling.md)
- [Cloud environment profiling](./Cloud-environment-profiling.md)


## Memory profiling and fine tuning

## Memory Basics
Once main memory has filled, the system may begin switching data between main memory and the storage devices. This is a slow process that will often become a system bottleneck, dramatically decreasing performance. The system may also terminate the largest memory-consuming process.

- **Swapping**: the transfer of pages between main memory and the storage devices.
- **File System Paging**: File system paging is caused by the reading and writing of pages in memory-mapped files. This is normal behavior for applications that use file memory mappings (mmap()), and on file systems that use the page cache.
- **Anonymous Paging**: Anonymous paging involves data that is private to processes:  the process heap and stacks. Anonymous paging hurts performance and has therefore been referred to as “bad” paging.
- **Page fault**: an invalid memory access. Minor faults are normal occurrences when using on-demand virtual memory.
- **Major faults**: Page faults that require storage device access, such as accessing an uncached memory-mapped file, are called major faults.
- **Resident memory**: memory that currently resides in main memory.

Performance is best when there is no anonymous paging (or swapping). This can be achieved by configuring applications to remain within the main memory available and by monitoring page scanning, memory utilization, and anonymous paging, to ensure that there are no longer indicators of a memory shortage.

### hardware
Memory hardware includes main memory, busses, CPU caches, and the MMU.

#### Main Memory
DRAM is a today's common type of main memory.

CPU 1 can perform I/O to DRAM A directly, via its memory bus. This is referred to as local memory. CPU 1 performs I/O to DRAM B via CPU 2 and the CPU interconnect (two hops) in NUMA. This is referred to as remote memory and has a higher access latency.

#### busses
How main memory is physically connected to the system depends on the main memory architecture. The actual implementation may involve additional controllers and busses between the CPUs and memory and be accessed in one of the following ways:

- **Shared system bus**: single or multiprocessor, via a shared system bus, a memory bridge controller, and finally a memory bus in UMA (uniform memory access).
-  **Interconnect**: multiprocessor, each with directly attached memory via a memory bus, and processors connected via a CPU interconnect in NUMA (non-UMA).

#### MMU

The memory management unit is responsible for virtual-to-physical address translations. These are performed per page, and offsets within a page are mapped directly. MMU is nearby CPU Caches.

A generic MMU is pictured, with levels of CPU caches and main memory.
![MMU](/Images/Memory-model.jpg)

Linux has a feature called huge pages, which sets aside a portion of physical memory for use with a particular large page size, such as 2 Mbytes. It avoids a problem of memory fragmentation preventing larger pages being dynamically allocated.

The MMU pictured in Figure uses a TLB as the first level of address translation cache, followed by the page tables in main memory. The TLB may be divided into separate caches for instruction and data pages.

### software

#### Freeing memory
**Reaping**: When a low-memory threshold is crossed, kernel modules and the kernel slab allocator can be instructed to immediately free any memory that can easily be freed. This is also known as shrinking. Reaping mostly involves freeing memory from the kernel slab allocator caches.

Linux kernel modules can also call register_shrinker() to register specific functions for reaping their own memory.

#### Page Scanning
The page-out daemon is called kswapd(), which scans LRU page lists of inactive and active memory to free pages. It is woken up based on free memory and two thresholds to provide hysteresis

The rate at which pages are scanned is dynamic, based on the available free memory. This is pictured in Figure for an example 128 Gbyte system, along with the tunable names

![page scan rate](/Images/Memory-page-scan-rate.jpg)

When available memory drops below desfree, and then minfree, the page-out daemon is woken up more frequently to scan pages. If available memory drops below desfree for 30 s, the kernel also begins swapping.


## Methodology
This describes various memory performance methodologies and exercises for memory analysis and tuning.

Methodology | Type
---|---
Tools method | Observational Analysis
USE method | Observational Analysis
Characterizing usage| Observational Analysis, capacity planing
Leak detection | tuning
Static performance tuning | tuning

These methods may be followed individually or used in combination. Suggestion is to use the following strategies to start with, in this order: the USE method, and characterizing usage.

### Tools method

For memory, the tools method can involve checking the following:

- **Page scanning**: Look for continual page scanning (more than 10 s) as a sign of memory pressure. This can be done using `sar -B` and checking the pgscan columns.
- **Swapping**: The paging of memory is a further indication that the system is low on memory. You can use `vmstat` and check the si and so columns (here, the term swapping means anonymous paging). Run vmstat per second and check the free column for available memory.
- **OOM killer**: These events can be seen in the system log /var/log/messages, or from dmesg. Search for “Out of memory.”
- **top/prstat**: See which processes and users are the top physical memory consumers (resident) and virtual memory consumers. Resident memory refers to memory that currently resides in main memory.
- **dtrace/stap/perf**: Trace memory allocations with stack traces, to identify the cause of memory usage.

### USE method

Check system-wide for

- **Utilization**: how much memory is in use, and how much is available. Both physical memory and virtual memory should be checked.
- **Saturation**: the degree of page scanning, paging, swapping, and Linux OOM killer sacrifices performed, as measures to relieve memory pressure.
- **Errors**: failed memory allocations.

Monitoring memory usage over time, especially by process, can help identify the presence and rate of memory leaks.

**Saturation may be checked first**, as continual saturation is a sign of a memory issue. These metrics are usually readily available from operating system tools, including vmstat, sar, and dmesg, for OOM killer sacrifices. For systems configured with a separate disk swap device, any activity to the swap device is also a sign of memory pressure.

 Different tools may report this differently, depending on whether they account for unreferenced file system cache pages or inactive pages. Check the tool documentation to see details.

For environments that implement memory limits or quotas (resource controls), as occurs in some cloud computing environments, memory saturation may need to be measured differently.

### Characterizing Usage
Characterizing memory usage is an important exercise when capacity planning, benchmarking, and simulating workloads. It can also lead to some of the largest performance gains by identifying misconfigurations. For example, a database cache may be configured too small and have low hit rates, or too large and cause system paging.

For memory, this involves identifying where and how much memory is used:

- System-wide physical and virtual memory utilization
- Degree of saturation: paging, swapping, OOM killing
- Kernel and file system cache memory usage
- Per-process physical and virtual memory usage
- Usage of memory resource controls, if present

Here is an example description to show how these attributes can be expressed together (profile):

**The system has 256 Gbytes of main memory, which is only at 1% utilization, with 30% in a file system cache. The largest process is a database, consuming 2 Gbytes of main memory (RSS), which is its configured limit from the previous system it was migrated from.**

#### Advanced usage analysis/Checklist
Use these questioner as a checklist when studying memory issues thoroughly:

- Where is the process memory used?
- Where is the kernel memory used? Per slab?
- How much of the file system cache (or page cache) is active as opposed to inactive?
- Why are processes allocating memory (call paths)?
- Why is the kernel allocating memory (call paths)?
- What processes are actively being paged/swapped out?
- What processes have previously been paged/swapped out?
- May processes or the kernel have memory leaks?
- In a NUMA system, how well is memory distributed across memory nodes?
- What are the CPI and memory stall cycle rates?
- How much local memory I/O is performed as opposed to remote memory I/O?
- How balanced are the memory busses?

Memory bus load can be determined by inspecting the CPU performance counters (CPCs), which can be programmed to count memory stall cycles. They can also be used to measure cycles per instruction (CPI), as a measure of how memory-dependent the CPU load is.

### Leak detection
Memory growth issues are often misidentified as memory leaks. The first question to ask is: Is it supposed to do that? Check the configuration.

This may be first noticed because the system is now paging, in response to the endless memory pressure. This type of issue is caused by either

- **A memory leak**: a type of software bug where memory is forgotten but never freed. This is fixed by modifying the software code, or by applying patches or upgrades (which modify the code).
- **Memory growth**: The software is consuming memory normally, but at a much higher rate than is desirable for the system. This is fixed either by changing the software configuration, or by the software developer changing how the application consumes memory.

How memory leaks can be analyzed depends on the software and language type. There are also tools available to record in debug mode and the developer can use for memory leak investigations.

### Static Performance Tuning
Static performance tuning focuses on issues of the configured environment. For memory performance, examine the following aspects of the static configuration:

- How much main memory is there in total?
- How much memory are applications configured to use?
- What is the speed of main memory? Is it the fastest type available?
- What is the system architecture? NUMA, UMA?
- Is the operating system NUMA-aware?
- How many memory busses are present?
- What are the number and size of the CPU caches? TLB?
- Are large pages configured and used?
- Is overcommit available and configured?
- What other system memory tunables are in use?
- Are there software-imposed memory limits (resource controls)?

Answering these questions may reveal configuration choices that have been overlooked.

## Analysis
This section introduces memory analysis tools. See the previous section for strategies to follow when using them.

Tools|description
---|----
vmstat| virtual and physical memory statistics
sar | historical statistics
slabtop | kernel slab allocator statistics
ps | process status
top | monitor per-process/thread memory usage
pmap | process address space statistics
Dtrace | allocating tracing

### vmstat
The virtual memory statistics command, vmstat, provides a high-level view of system memory health, including current free memory and paging statistics.

```bash
$ vmstat 1
procs -----------memory---------- ---swap-- -----io---- -system-- ----cpu----
 r  b   swpd   free   buff  cache   si   so    bi    bo   in   cs us sy id wa
 4  0      0 34454064 111516 13438596   0   0    0    5    2    0  0  0 100  0
 4  0      0 34455208 111516 13438596   0   0    0    0 2262 15303 16 12 73  0
 5  0      0 34455588 111516 13438596   0   0    0    0 1961 15221 15 11 74  0
 4  0      0 34456300 111516 13438596   0   0    0    0 2343 15294 15 11 73  0
[...]
```

The columns are in kilobytes by default and are
- **swpd**: amount of swapped-out memory
- **free**: free available memory
- **buff**: memory in the buffer cache
- **cache**: memory in the page cache
- **si**: memory swapped in (paging)
- **so**: memory swapped out (paging)

The buffer and page caches are described in [File Systems](./filesystem-profiling.md#file-system-caches) page. It is normal for the free memory in the system to drop after boot and be used by these caches to improve performance. It can be released for application use when needed.

If the si and so columns are continually non-zero, the system is under memory pressure and is paging to a swap device or file. Other tools, including memory by process, can be used to investigate what is consuming memory.

using the -S option if you experience unaligned issue.

`$ vmstat 1 -Sm`

There is also a -a option for printing a breakdown of inactive and active memory from the page cache:

```bash
$ vmstat -a 1
procs -----------memory---------- ---swap-- -----io---- -system-- ----cpu----
 r  b   swpd   free  inact active   si   so    bi    bo   in   cs us sy id wa
 5  0      0 34453536 10358040 3201540   0   0    0    5    2    0  0  0 100  0
 4  0      0 34453228 10358040 3200648   0   0    0    0 2464 15261 16 12 71  0
[...]
```
These memory statistics can be printed as a list using the -s option.

### sar
The system activity reporter, sar, can be used to observe current activity and can be configured to archive and report historical statistics.

Memory statistics via the following options:
- -B: paging statistics
- -H: huge pages statistics
- -r: memory utilization
- -R: memory statistics
- -S: swap space statistics
- -W: swapping statistics

![SAR observability](/Images/linux_observability_sar.jpg)

SAR statistics.
![SAR options](/Images/Linux-sar.jpg)
Many of the statistic names include the units measured: pg for pages, kb for kilobytes, % for a percentage, and /s for per second.

**The %vmeff metric is an interesting measure of page reclaim efficiency. High means pages are successfully stolen from the inactive list (healthy); low means the system is struggling.** The man page describes near 100% as high, and less than 30% as low.

### slaptop
The Linux slabtop command prints kernel slab cache usage from the slab allocator. Like top, it refreshes the screen in real time.

```bash
$ slabtop -sc
 Active / Total Objects (% used)    : 3590651 / 3682877 (97.5%)
 Active / Total Slabs (% used)      : 94610 / 94610 (100.0%)
 Active / Total Caches (% used)     : 58 / 83 (69.9%)
 Active / Total Size (% used)       : 432643.91K / 477592.84K (90.6%)
 Minimum / Average / Maximum Object : 0.01K / 0.13K / 12.75K

  OBJS ACTIVE  USE OBJ SIZE  SLABS OBJ/SLAB CACHE SIZE NAME
3345069 3334148  99%    0.10K  85771       39    343084K buffer_head
151728  77833  51%    0.55K   5232       29     83712K radix_tree_node
  5520   4495  81%    2.00K    345       16     11040K kmalloc-2048
 11193  11185  99%    0.82K    287       39      9184K ext3_inode_cache
  9464   9464 100%    0.61K    182       52      5824K inode_cache
 29064  28977  99%    0.19K    692       42      5536K dentry
  4896   4734  96%    0.66K    102       48      3264K proc_inode_cache
   380    344  90%    5.73K     76        5      2432K task_struct
 20094  20094 100%    0.08K    394       51      1576K sysfs_dir_cache
[...]
```

The output has a summary at the top and a list of slabs, including their object count (OBJS), how many are active (ACTIVE), percent used (USE), the size of the objects (OBJ SIZE, bytes), and the total size of the cache (CACHE SIZE, bytes).

In this example, the -sc option was used to sort by cache size, with the largest at the top.

The slab statistics are from /proc/slabinfo and can also be printed using `vmstat -m`.

#### ps
The process status command, ps(1), lists details on all processes, including memory usage statistics.

```bash
$ ps aux
USER       PID %CPU %MEM    VSZ   RSS TTY  STAT START   TIME COMMAND
[...]
bind      1152  0.0  0.4 348916 39568 ?    Ssl  Mar27  20:17 /usr/sbin/named -u bind
root      1371  0.0  0.0  39004  2652 ?    Ss   Mar27  11:04 /usr/lib/postfix/master
root      1386  0.0  0.6 207564 50684 ?    Sl   Mar27   1:57 /usr/sbin/console-kit-daemon --no-daemon
rabbitmq  1469  0.0  0.0  10708   172 ?    S    Mar27   0:49 /usr/lib/erlang/erts-5.7.4/bin/epmd -daemon
rabbitmq  1486  0.1  0.0 150208  2884 ?    Ssl  Mar27 453:29 /usr/lib/erlang/erts-5.7.4/bin/beam.smp -W w -K true -A30 ...
```

- **%MEM**: main memory usage (physical memory, RSS) as a percentage of the total in the system
- **RSS**: resident set size (Kbytes)
- **VSZ**: virtual memory size (Kbytes)

While RSS shows current main memory usage, it includes shared segments such as system libraries, which may be mapped by dozens of processes. If you were to sum the RSS column, you may find it exceeds the memory available in the system, due to over counting of this shared memory. See the later pmap command for analysis of shared memory usage.

Select the required column use the SVR4-style -o option,

```bash
# ps -eo pid,pmem,vsz,rss,comm
  PID %MEM  VSZ  RSS COMMAND
[...]
13419  0.0 5176 1796 /opt/local/sbin/nginx
13879  0.1 31060 22880 /opt/local/bin/ruby19
13418  0.0 4984 1456 /opt/local/sbin/nginx
15101  0.0 4580   32 /opt/riak/lib/os_mon-2.2.6/priv/bin/memsup
10933  0.0 3124 2212 /usr/sbin/rsyslogd
[...]
```
You can also print columns for major and minor faults (maj_flt, min_flt).

You can also use top command to display virtual memory, Resident set size, Shared memory and main memory for each process.

#### pmap

The pmap command lists the memory mappings of a process, showing their sizes, permissions, and mapped objects. This allows process memory usage to be examined in more detail, and shared memory to be quantified.

```bash
# pmap -x 13504
13504:  /opt/local/bin/postgres -D /var/pgsql/data90
 Address  Kbytes     RSS    Anon  Locked Mode   Mapped File
08027000     132     132       4       - rw---    [ stack ]
08050000    4204    1880       -       - r-x--  postgres
0847A000      28      28       -       - rwx--  postgres
08481000     260      48       -       - rwx--  postgres
084C2000     248     212      20       - rwx--    [ heap ]
FC400000   36112   36112       -   36112 rwxsR    [ ism shmid=0x1 ]
FE8E0000     904      68       -       - r-x--  libiconv.so.2.5.0
FE9D1000       4       4       -       - rwx--  libiconv.so.2.5.0
FE9E0000    1220    1220       -       - r-x--  libc_hwcap1.so.1
FEB21000      36      36      12       - rwx--  libc_hwcap1.so.1
FEB2A000       8       8       -       - rwx--  libc_hwcap1.so.1
FEB30000     416     416       -       - r-x--  libnsl.so.1
FEBA8000       8       8       -       - rw---  libnsl.so.1
FEBAA000      20      20       -       - rw---  libnsl.so.1
FEBD0000     304     304       -       - r-x--  libm.so.2
FEC2B000      16      16       -       - rwx--  libm.so.2
FEC81000       4       4       -       - rwxs-    [ anon ]
FEC90000      64       8       -       - rwx--    [ anon ]
FECAC000      76      32       -       - r----  LCL_DATA
[...]
```
For most of the mappings, very little memory is anonymous, and much of it is read-only (r-x), meaning those pages can be shared with other processes.

#### DTrace
DTrace can be used to trace user- and kernel-level allocations, minor and major page faults, and the operation of the page-out daemon. These abilities support characterizing usage and drill-down analysis.

##### Allocation Tracing
User-level allocators can be traced using the pid provider, if available. This is a dynamic tracing provider, which means software can be instrumented at any moment, without restarting, and without needing to configure allocators to run in debug mode beforehand.

The following example summarizes the requested size of malloc() calls, for PID 15041.

```bash
# dtrace -n 'pid$target::malloc:entry { @["requested bytes"] = quantize(arg0); }' -p 15041
dtrace: description 'pid$target::malloc:entry ' matched 3 probes
^C

  requested bytes
           value  ------------- Distribution ------------- count
             256 |                                         0
             512 |@@@@                                     3824
            1024 |@@@@@@@@@@@@@@@@@                        17807
            2048 |@@@@@@@@@@@@@                            13564
            4096 |@@@@@@                                   5907
            8192 |@                                        1040
           16384 |                                         0
```
This one-liner summarizes the requested bytes for malloc(), which is the first argument, by passing it (arg0) to the power-of-two quantize() aggregating function. If desired, the return value of malloc() can be traced as well, to check that the allocation succeeded.

This key can include the user-level stack trace using the ustack() action.

```bash
# dtrace -n 'pid$target::malloc:entry { @["requested bytes, for:", ustack()] = quantize(arg0); }' -p 15041
dtrace: description 'pid$target::malloc:entry' matched 3 probes
[...]
  requested bytes, for:
              libumem.so.1`malloc
              libstdc++.so.6.0.13`_Znwm+0x1e
              libstdc++.so.6.0.13`_Znam+0x9

              eleveldb.so`_ZN7leveldb9ReadBlockEPNS_16RandomAccessFileERKNS_...

              eleveldb.so`_ZN7leveldb5Table11BlockReaderEPvRKNS_11ReadOption...

              eleveldb.so`_ZN7leveldb12_GLOBAL__N_116TwoLevelIterator13InitD...

              eleveldb.so`_ZN7leveldb12_GLOBAL__N_116TwoLevelIterator4SeekER...

              eleveldb.so`_ZN7leveldb12_GLOBAL__N_115MergingIterator4SeekERK...

              eleveldb.so`_ZN7leveldb12_GLOBAL__N_16DBIter4SeekERKNS_5SliceE...
              eleveldb.so`eleveldb_iterator_move+0x24b
              beam.smp`process_main+0x6939
              beam.smp`sched_thread_func+0x1cf
              beam.smp`thr_wrapper+0xbe
              0xfffffd7fe4d7b862
              0xb8c0000000000000

           value  ------------- Distribution ------------- count
             256 |                                         0
             512 |@@@@@                                    1
            1024 |@@@@@@@@@@                               2
            2048 |@@@@@@@@@@@@@@@@@@@@@@@@@                5
            4096 |                                         0

```

In this case, the output was many pages long and has been truncated to fit. It shows user-level stack traces that led to the allocation, along with a distribution of the requested allocation size.

Other internals of user-level allocators can be investigated. For example, listing the entry probes for the libumem allocator.

`# dtrace -ln 'pid$target:libumem::entry' -p 15041`

#### One-Liners
- Summarize user-level malloc() request size for process PID.
`dtrace -n 'pid$target::malloc:entry { @["request"] = quantize(arg0); }' -p PID`

- Summarize user-level malloc() request size with call stack for process PID.
`dtrace -n 'pid$target::malloc:entry { @[ustack()] = quantize(arg0); }' -p PID`

- Count libumem function calls.
`dtrace -n 'pid$target:libumem::entry { @[probefunc] = count(); }' -p PID`

- Count user-level stacks for heap growth (via brk()):
`dtrace -n 'syscall::brk:entry { @[execname, ustack()] = count(); }'`

#### SystemTap
SystemTap can also be used on Linux systems for dynamic tracing of file system events.

#### Other tools
Other Linux memory performance tools include the following:

- **free**: report free memory, with buffer cache and page cache.
- **dmesg**: check for “Out of memory” messages from the OOM killer.
- **valgrind**: a performance analysis suite, including memcheck, a wrapper for user-level allocators for memory usage analysis including leak detection. This costs significant overhead; the manual advises that it can cause the target to run 20 to 30 times slower.
- **swapon**: to add and observe physical swap devices or files.
- **iostat**: If the swap device is a physical disk or slice, device I/O may be observable using iostat, which indicates that the system is paging.
- **perf**: Introduced in CPUs, this can be used to investigate CPI, MMU/TSB events, and memory bus stall cycles from the CPU performance instrumentation counters. It also provides probes for page faults and several kernel memory (kmem) events.
- **/proc/zoneinfo**: statistics for memory zones (NUMA nodes).
- **/proc/buddyinfo**: statistics for the kernel buddy allocator for pages.

Applications and virtual machines (e.g., the Java VM) may also provide their own memory analysis tools. [Applications](/Application-Performance-Analysis.md).

## Tuning
The most important memory tuning is ensuring that the applications remain in main memory, and that paging and swapping do not occur frequently. Identifying this problem was covered in Methodology, and Analysis. This section discusses other memory tuning: kernel tunable parameters, configuring large pages, allocators, and resource controls.

The specifics of tuning the options available and what to set them to—depend on the operating system version and the intended workload.

This section describes tunable parameter examples.
Various memory tunable parameters are described in the kernel source documentation in Documentation/sysctl/vm.txt and can be set using sysctl.

Option| Default | Description
----| ----|----
vm.dirty_background_bytes| 0 | amount of dirty memory to trigger pdflush background write-back
vm.dirty_background_ration | 10 | percentage of dirty system memory to trigger pdflush background write-back
vm.dirty_bytes | 0 | amount of dirty memory that causes a writing process to start write-back
vm.dirty_ration | 20 | ration of dirty system mempry to cause a writing process to begin write-back
vm.dirty_expires_centisecs | 3,000| minimum time for dirty memory to be eligible for pdflush (promotes write cancellation)
vm.dirty_writeback_centisecs|500|pdflush wake-up interval (0 to disable)
vm.min_free_kbytes|dynamic|set the desired free memory amount
vm.overcommit_memory|0| 0=use heuristic to allow reasonable over-commit; 1=always overcommit; 2- don\'t overcommit
vm.swappiness | 60 | the degree to favour swapping (paging) for freeing memory over reclaiming it from the page cache
vm.vfs_cache_pressure | 100 | the degree to reclaim cached directory and inode objects; lower values retain then more; 0 means never reclaim - can easily lead to out-of-memory conditions.

Note: dirty_background_bytes and dirty_background_ratio are mutually exclusive, as are dirty_bytes and dirty_ratio (only one may be set).

### Multiple page sizes
Large page sizes can improve memory I/O performance by improving the hit ratio of the TLB cache (increasing its reach). Most modern processors support multiple page sizes, such as a 4 Kbyte default and a 2 Mbyte large page.

Large pages (called huge pages) can be configured in a number of ways. For reference, see Documentation/vm/hugetlbpage.txt.

```bash
# echo 50 > /proc/sys/vm/nr_hugepages
# grep Huge /proc/meminfo
AnonHugePages:         0 kB
HugePages_Total:      50
HugePages_Free:       50
HugePages_Rsvd:        0
HugePages_Surp:        0
Hugepagesize:       2048 kB
```
One way for an application to consume huge pages is via the shared memory segments, and the SHM_HUGETLBS flag to shmget().

Another way involves creating a huge-page-based file system for applications to map memory from.

```bash
# mkdir /mnt/hugetlbfs
# mount -t hugetlbfs none /mnt/hugetlbfs -o pagesize=2048K
```
Other ways include the MAP_ANONYMOUS|MAP_HUGETLB flags to mmap() and use of the libhugetlbfs API.

### Resource controls
Basic resource controls, including setting a main memory limit and a virtual memory limit, may be available using ulimit(1).

For Linux, the container groups (cgroups) memory subsystem provides various additional controls. These include

- **memory.memsw.limit_in_bytes**: the maximum allowed memory and swap space, in bytes
- **memory.limit_in_bytes**: the maximum allowed user memory, including file cache usage, in bytes
- **memory.swappiness**: similar to vm.swappiness described earlier but can be set for a cgroup
- **memory.oom_control**: can be set to 0, to allow the OOM killer for this cgroup, or 1, to disable it.

[Back to top](#memory-basics)
