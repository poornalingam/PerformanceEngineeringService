#  System Profiling and Tuning
- [Application profiling](./Application-Performance-Analysis.md)
- [CPU profiling](./CPU-Profiling.md)
- [Memory profiling](./Memory-profiling.md)
- [File Systems profiling](./filesystem-profiling.md)
  - [FS basics](#basics)
  - [Architecture](#architecture)
  - [Methodologies](#methodologies)
  - [Analysis](#analysis)
  - [Benchmark](#benchmark)
  - [Tuning](#tuning)
- [Disk profiling](./disk-profiling.md)
- [Network profiling](./Network-profiling.md)
- [Cloud environment profiling](./Cloud-environment-profiling.md)

## File Systems profiling and tuning
When studying application I/O performance, the **performance of the file system matters more than disk performance. File systems use caching, buffering, and asynchronous I/O to avoid subjecting applications to disk-level (or remote system) latency.** Nevertheless, performance analysis and the available toolsets have historically focused on the performance of the disks.

This section shows how file system requests can be examined in detail, including the use of dynamic tracing to measure start to completion time from the application context.

## Basics

### Key Terminology
- **File system cache**: an area of main memory (usually DRAM) used to cache file system contents, which may include different caches for various data and metadata types.
- **Logical I/O**: I/O issued by the application to the file system.
- **Physical I/O**: I/O issued directly to disks by the file system (or via raw I/O).
- **Throughput**: the current data transfer rate between applications and the file system, measured in bytes per second
- **VFS**: virtual file system, a kernel interface to abstract and support different file system types.
- **Volume manager**: software for managing physical storages devices in a flexible way, creating virtual volumes from them for use by the OS.

File system is a location where logical and physical operations occur.

### File System Cache
A generic file system cache stored in main memory is pictured, servicing a read operation.

![Mail Memory cache](/Images/FS-Main-Memory-cache.jpg)

The read returns either from cache (cache hit) or from disk (cache miss). Cache misses are stored in the cache, populating the cache (warming it up).

### File System Latency

File system latency is the primary metric of file system performance, measured as the time from a logical file system request to its completion. It is **inclusive of time spent in the file system, kernel disk I/O subsystem, and waiting on disk devices—the physical I/O**. Application threads often block during an application request to wait for file system requests to complete, for which file system latency directly and proportionally affects application performance.

Cases where applications may not be directly affected include the use of non-blocking I/O, or when I/O is issued from an asynchronous thread.

### Caching
For applications, this process is transparent: their logical I/O latency becomes much lower, as it can be served from main memory rather than the much slower disk devices.

The principle is: If there is spare main memory, remember something useful. When applications need more memory, the kernel should quickly free it from the file system cache for use.

### Prefetch or Read-Ahead
Prefetch can detect a sequential read workload based on the current and previous file I/O offsets, and then predict and issue disk reads before the application has requested them.

When prefetch detection works well, applications show significantly improved sequential read performance; the disks keep ahead of application requests. When prefetch detection works poorly, unnecessary I/O is issued that the application does not need, polluting the cache and consuming disk and I/O transport resources. File systems typically allow prefetch to be tuned as needed.

### Synchronous Writes
Synchronous are much slower than asynchronous writes (write-back caching), since synchronous writes incur disk device I/O latency.

Individual Synchronous Writes: Write I/O is synchronous when a file is opened using the flag O_SYNC or one of the variants.

**Direct I/O** allows applications to use a file system but bypass the file system cache.

**Raw I/O** is issued directly to disk offsets, bypassing the file system altogether.

### non-blocking I/O
non-blocking I/O (async) was also discussed in [Applications section.](/Application-Performance-Analysis.md#non-blocking-io)

### Memory-Mapped Files
For some applications and workloads, **file system I/O performance can be improved by mapping files to the process address space and accessing memory offsets directly**. This avoids the syscall execution and context switch overheads incurred when calling read() and write() syscalls to access file data. It can also avoid double copying of data, if the kernel supports direct copying of the file data buffer to the process address space.

Memory mappings are created using the mmap() syscall and removed using munmap(). Mappings can be tuned using madvise(). **Some applications provide an option to use the mmap syscalls in their configuration. For example, the Riak database can use mmap for its in-memory data store.**

### Logical versus Physical I/O
File systems do much more than present persistent storage (the disks) as a file-based interface. They cache reads, buffer writes, and create additional I/O to maintain the on-disk physical layout metadata that they need to record where everything is.

Sometime file system (logical I/O) may not match disk I/O (physical I/O), for several reasons. Unrelated: other applications, other tenants and other other kernel tasks. Indirect: File system prefetch, file system buffering. File system I/O smaller than physical I/O: File System Caching, File System write cancellation, Compression, coalescing (Merging sequential I/O before issuing them to desk), In-memory file system. All of these can happen concert.

### Access Timestamps
Many file systems support access timestamps, which record the time that each file and directory was accessed (read). This causes file metadata to be updated whenever files are read, creating a write workload that consumes disk I/O resources. Turn off these updates or deferring and grouping them to reduce active workload.

### Capacity
When file systems fill, performance may degrade for a couple of reasons. When writing new data, it may take more time to locate the free blocks on disk for computation, and any disk I/O needed. Areas of free space on disk are likely to be smaller and more sparsely located, degrading performance due to smaller I/O or random I/O.

How much of a problem this is depends on the file system type, its on-disk layout, and its storage devices.

## Architecture

### File System I/O Stack
A general model of the file system I/O stack. Specific components and layers depend on the operating system type, version, and file systems used. Operating Systems, for the full diagram.

![Generic-Filesystem-IO-Stack.jpg](/Images/Generic-Filesystem-IO-Stack.jpg)

This shows the path of I/O through the kernel. The path from system calls direct to the disk device subsystem is raw I/O. The path via VFS and the file system is file system I/O, including direct I/O which skips the file system cache.

### File System Caches
Unix originally had only the buffer cache to improve the performance of block device access. Nowadays, Linux have multiple different cache types.

 File system caches, showing generic caches available for standard file system types.

![Linux FS Cache](/Images/Linux-FS-Cache.jpg)

#### Buffer Cache
The buffer cache functionality is being used to improve the performance of block device I/O. The size of the buffer cache is dynamic and is observable from /proc.

#### Page Cache
The page cache caches virtual memory pages, including file system pages, improving the performance of file and directory I/O. The size of the page cache is dynamic, and it will grow to use available memory, freeing it again when applications need it.

Pages of memory that are dirty (modified) and are for use by a file system are flushed to disk by kernel threads (named flush). This topic discussed in [page scanner in memory](./Memory-profiling.md#page-scanning)

## Methodologies
This section describes various strategies and exercises for file system analysis and tuning.

Methodology | types
---- | ----
Workload Characterization | observation analysis
Performance monitoring | observation analysis, capacity planning
Static performance tuning| observation Analysis, capacity planning
Cache tuning | Observation analysis, tuning
Workload separation | tuning

### Workload Characterization
Characterizing the load applied is an important exercise when capacity planning, benchmarking, and simulating workloads.

Here are the basic attributes for characterizing the file system workload.

- Operation rate and operation types
- File I/O throughput
- File I/O size
- Read/write ratio
- Synchronous write ratio
- Random versus sequential file offset access

These characteristics can vary from second to second, especially for timed application tasks that execute at intervals. To better characterize the workload, capture maximum values as well as averages. Better still, examine the full distribution of values over time.

Here is an example workload description, to show how these attributes can be expressed together:

On a financial trading database, the file system has a random read workload, averaging 18,000 reads/s with an average read size of 2 Kbytes. The total operation rate is 21,000 ops/s, which includes reads, stats, opens, closes, and around 200 synchronous writes/s. The write rate is steady while the read rate varies, up to a peak of 39,000 reads/s.

#### Advanced Workload Characterization/Checklist
Additional details may be included to characterize the workload. These have been listed here as questions for consideration, which may also serve as a checklist when studying file system issues thoroughly.

- What is the file system cache hit ratio? Miss rate?
- What are the file system cache capacity and current usage?
- What other caches are present (directory, inode, buffer) and what are their statistics?
- Which applications or users are using the file system?
- What files and directories are being accessed? Created and deleted?
- Have any errors been encountered? Was this due to invalid requests, or issues from the file system?
- Why is file system I/O issued (user-level call path)?
- To what degree is the file system I/O application synchronous?
- What is the distribution of I/O arrival times?

Many of these questions can be posed per application or per file. Any of them may also be checked over time, to look for maximums and minimums, and time-based variations

#### Performance Characterization
The following questions (contrast with the previous workload characterization questions) characterize the resulting performance of the workload:

- What is the average file system operation latency?
- Are there any high-latency outliers?
- What is the full distribution of operation latency?
- Are system resource controls for file system or disk I/O present and active?

The first three questions may be asked for each operation type separately.

### Performance Monitoring
Performance monitoring can identify active issues and patterns of behavior over time. Key metrics for file system performance are

- Operation rate
- Operation latency

The operation rate is the most basic characteristic of the applied workload, and the latency is the resulting performance. The value for normal or bad latency depends on your workload, environment, and latency requirements.

The operation latency metric may be monitored as a per-second average and can include other values such as the maximum and standard deviation. Ideally, it would be possible to inspect the full distribution of latency, to look for outliers and other patterns.

Both rate and latency may also be recorded for each operation type (read, write, stat, open, close, etc.). Doing this will greatly help investigations of workload and performance changes, by identifying differences in particular operation types.

### Static Performance Tuning
Static performance tuning focuses on issues of the configured environment. For file system performance, examine the following aspects of the static configuration:

- How many file systems are mounted and actively used?
- What is the file system record size?
- Are access timestamps enabled?
- What other file system options are enabled (compression, encryption, . . .)?
- How has the file system cache been configured? Maximum size?
- How have other caches (directory, inode, buffer) been configured?
- Is a second-level cache present and in use?
- How many storage devices are present and in use?
- What is the storage device configuration? RAID?
- Which file system types are used?
- What is the version of the file system (or kernel)?
- Are there file system bugs/patches that should be considered?
- Are there resource controls in use for file system I/O?

Answering these questions can reveal configuration choices that have been overlooked. Sometimes a system has been configured for one workload, and then repurposed for another. This method will revisit those choices.

### Cache Tuning
The kernel and file system may use many different caches, including a buffer cache, directory cache, inode cache, and file system (page) cache. Various caches were described in [File System Caches Section, Architecture](#file-system-caches), which can be tuned as described in [Cache Tuning](/Method-PerformanceAnalysis.md#cache-tuning). **In summary, check which caches exist, check that they are working, check how well they are working, check their sizes, then tune the workload for the cache and tune the cache for the workload.**

### Workload Separation
Some types of workloads can perform better when configured to use their own file systems and disk devices. **Creating random I/O by seeking between two different workload locations is particularly bad for rotational disks.**

For example, a database may benefit from having separate file systems and disks for its log files and its database files.

## Analysis
This section introduces file system performance analysis tools.

Tools | Description
----| ----
strace | System call debuggers
Dtrace | Dynamic tracing of file system operations, latency
free | cache capacity statistics
vmstat| Virtual memory statistics
sar | Various statistics, including historic
slabtop | kernel slab allocator statistics
/proc/meminfo| Kernel memory breakdowns

This is a selection of tools and capabilities to support the preceding methodology section, beginning with system-wide and per-file-system observability, then operation and latency analysis, and finishing with cache statistics.

### strace
Previous operating system tools for measuring file system latency in detail included the debuggers for the syscall interface, such as strace. Such debuggers can hurt performance and may be suitable for **use only when the performance overhead is acceptable and other methods to analyze latency are not possible.**

This example shows strace timing reads on an ext4 file system:

```bash
$ strace -ttT -p 845
[...]
18:41:01.513110 read(9, "\334\260/\224\356k..."..., 65536) = 65536 <0.018225>
18:41:01.531646 read(9, "\371X\265|\244\317..."..., 65536) = 65536 <0.000056>
18:41:01.531984 read(9, "\357\311\347\1\241..."..., 65536) = 65536 <0.005760>
18:41:01.538151 read(9, "*\263\264\204|\370..."..., 65536) = 65536 <0.000033>
18:41:01.538549 read(9, "\205q\327\304f\370..."..., 65536) = 65536 <0.002033>
18:41:01.540923 read(9, "\6\2738>zw\321\353..."..., 65536) = 65536 <0.000032>
```
**The -tt option prints the relative timestamps on the left, and -T prints the syscall times on the right**. Each read() was for 64 Kbytes, the first taking 18 ms, followed by 56 μs (likely cached), then 5 ms. The reads were to file descriptor 9. To check that this is to a file system (and isn’t a socket), either the open() syscall will be visible in earlier strace output, or another tool such as lsof can be used.

### Dtrace
File system operations can be observed from the syscall and fbt providers, until fsinfo is available. For example, using fbt to trace kernel vfs functions:
```bash
# dtrace -n 'fbt::vfs_*:entry { @[execname] = count(); }'
dtrace: description 'fbt::vfs_*:entry ' matched 39 probes
^C
[...]
  sshd                                                            913
  ls                                                             1367
  bash                                                           1462
  sysbench                                                      10295
```
The largest number of file system operations during this trace was called by applications with the name sysbench.

Counting the type of operation by aggregating on probefunc.

```bash
# dtrace -n 'fbt::vfs_*:entry /execname == "sysbench"/ { @[probefunc] = count(); }'
dtrace: description 'fbt::vfs_*:entry ' matched 39 probes
^C
  vfs_write                                                      4001
  vfs_read                                                       5999
```
This matched a sysbench process while it performed a random read-write benchmark, showing the ratio of operations. To strip the vfs_ from the output, instead of @[probefunc], use @[probefunc + 4].

#### File opens

one-liners used DTrace to summarize event counts. The following demonstrates printing all event data separately, in this case, details for the open() system call, system-wide:

```bash
# opensnoop -ve
STRTIME                UID    PID COMM        FD ERR PATH
2012 Sep 13 23:30:55 45821  24218 ruby        23   0 /var/run/name_service_door
2012 Sep 13 23:30:55 45821  24218 ruby        23   0 /etc/inet/ipnodes
2012 Sep 13 23:30:55    80   3505 nginx       -1   2 /public/dev-3/vendor/
2012 Sep 13 23:30:56    80  25308 php-fpm      5   0 /public/etc/config.xml
2012 Sep 13 23:30:56    80  25308 php-fpm      5   0 /public/etc/local.xml
2012 Sep 13 23:30:56    80  25308 php-fpm      5   0 /public/etc/local.xml
[...]
```
#### System call Latency

This one-liner measures file system latency at the system call interface, summarizing it as a histogram in units of nanoseconds:

```bash
# dtrace -n 'syscall::read:entry /fds[arg0].fi_fs == "zfs"/ { self->start = timestamp; } syscall::read:return/self->start/ { @["ns"] = quantize(timestamp - self->start); self->start = 0; }'
dtrace: description 'syscall::read:entry ' matched 2 probes
^C
  ns
           value  ------------- Distribution ------------- count
            1024 |                                         0
            2048 |                                         2
            4096 |@@@@@@                                   103
            8192 |@@@@@@@@@@                               162
           16384 |                                         3
           32768 |                                         0
           65536 |                                         0
          131072 |                                         1
          262144 |                                         0
          524288 |                                         1
         1048576 |                                         3
         2097152 |@@@                                      48
         4194304 |@@@@@@@@@@@@@@@@@@@@@                    345
         8388608 |                                         0
```
The distribution shows two peaks, the first between 4 and 16 μs (cache hits), and the second between 2 and 8 ms (disk reads). Instead of quantize(), the avg() function could be used to show the average (mean). However, that would average the two peaks, which would be misleading.

### VFS Latency
The VFS interface can be traced, either via a static provider (if one exists) or via dynamic tracing (the fbt provider).
```bash
# dtrace -n 'fbt::vfs_read:entry /stringof(((struct file *)arg0)-> f_path.dentry->d_sb->s_type->name) == "ext4"/ { self->start = timestamp; } fbt::vfs_read:return /self->start/ { @["ns"] = quantize(timestamp - self->start); self->start = 0; }'
dtrace: description 'fbt::vfs_read:entry ' matched 2 probes
^C
  ns
           value  ------------- Distribution ------------- count
            1024 |                                         0
            2048 |@                                        13
            4096 |@@@@@@@@@@@@                             114
            8192 |@@@                                      26
           16384 |@@@                                      32
           32768 |@@@                                      29
           65536 |@@                                       23
          131072 |@                                        9
          262144 |@                                        5
          524288 |@@                                       14
         1048576 |@                                        6
         2097152 |@@@                                      31
         4194304 |@@@@@@                                   55
         8388608 |@@                                       14
        16777216 |                                         0
```
This time the predicate matches on the ext4 file system. Peaks for both cache hits and misses can be seen, with expected latency.

Listing VFS function entry probes:
```bash
# dtrace -ln 'fbt::vfs_*:entry'
   ID   PROVIDER            MODULE                          FUNCTION NAME
15518        fbt            kernel                        vfs_llseek entry
15552        fbt            kernel                         vfs_write entry
15554        fbt            kernel                          vfs_read entry
15572        fbt            kernel                        vfs_writev entry
15574        fbt            kernel                         vfs_readv entry
15678        fbt            kernel                    vfs_kern_mount entry
15776        fbt            kernel                       vfs_getattr entry
15778        fbt            kernel                       vfs_fstatat entry
[...31 lines truncated...]
```

### LatencyTop
LatencyTOP is a tool for reporting sources of latency, aggregated system-wide and per process.

File system latency is reported by LatencyTOP.

```
Cause                                                Maximum     Percentage
Reading from file                                 209.6 msec         61.9 %
synchronous write                                  82.6 msec         24.0 %
Marking inode dirty                                 7.9 msec          2.2 %
Waiting for a process to die                        4.6 msec          1.5 %
Waiting for event (select)                          3.6 msec         10.1 %
Page fault                                          0.2 msec          0.2 %

Process gzip (10969)                       Total: 442.4 msec
Reading from file                                 209.6 msec         70.2 %
synchronous write                                  82.6 msec         27.2 %
Marking inode dirty                                 7.9 msec          2.5 %

```
### vmstat
The vmstat command, like top, also may include details on the file system cache.

```bash
$ vmstat 1
procs -----------memory---------- ---swap-- -----io---- --system-- -----cpu-----
 r  b   swpd   free   buff  cache   si   so    bi    bo   in   cs us sy id wa st
 0  0      0  70296 134024 623100    0    0     1     1    7    6  0  0 100  0  0
 0  0      0  68900 134024 623100    0    0     0     0   46   96  1  2 97  0  0
[...]
```
The buff column shows the buffer cache size, and cache shows the page cache size, both in kilobytes.

### slabtop
The Linux slabtop command prints information about the kernel slab caches, some of which are used for file system caches:

```bash
# slabtop -o
 Active / Total Objects (% used)    : 151827 / 165106 (92.0%)
 Active / Total Slabs (% used)      : 7599 / 7599 (100.0%)
 Active / Total Caches (% used)     : 68 / 101 (67.3%)
 Active / Total Size (% used)       : 44974.72K / 47255.53K (95.2%)
 Minimum / Average / Maximum Object : 0.01K / 0.29K / 8.00K

  OBJS ACTIVE  USE OBJ SIZE  SLABS OBJ/SLAB CACHE SIZE NAME
 35802  27164  75%    0.10K    918       39      3672K buffer_head
 26607  26515  99%    0.19K   1267       21      5068K dentry
 26046  25948  99%    0.86K   2894        9     23152K ext4_inode_cache
 12240  10095  82%    0.05K    144       85       576K shared_policy_node
 11228  11228 100%    0.14K    401       28      1604K sysfs_dir_cache
  9968   9616  96%    0.07K    178       56       712K selinux_inode_security
  6846   6846 100%    0.55K    489       14      3912K inode_cache
  5632   5632 100%    0.01K     11      512        44K kmalloc-8
[...]
```
Without the -o output mode, slabtop will refresh and update the screen.

Slabs may include

- dentry: dentry cache
- inode_cache: inode cache
- ext3_inode_cache: inode cache for ext3
- ext4_inode_cache: inode cache for ext4

slabtop uses /proc/slabinfo, which exists if CONFIG_SLAB is enabled.

### Visaulizations
TBD.

## benchmark

### dd
The dd command (device-to-device copy) can be used to perform ad hoc tests of sequential file system performance. The following commands write, then read a 1 Gbyte file named file1 with a 1 Mbyte I/O size:

```bash
write: dd if=/dev/zero of=file1 bs=1024k count=1k
read: dd if=file1 of=/dev/null bs=1024k
```
### Micro benchmark

There are many file system benchmark tools available, including Bonnie, Bonnie++, iozone, tiobench, SysBench, fio, and FileBench. A few are discussed here, in order of increasing complexity.

The Bonnie tool is a simple C program to test several workloads on a single file, from a single thread.

```bash
# ./Bonnie -h
usage: Bonnie [-d scratch-dir] [-s size-in-Mb] [-html] [-m machine-label]
```
Use -s to set the size of the file to test. By default, Bonnie uses 100 Mbytes, which has entirely cached on this system:

```bash
$ ./Bonnie
File './Bonnie.9598', size: 104857600
Writing with putc()...done
Rewriting...done
Writing intelligently...done
Reading with getc()...done
Reading intelligently...done
Seeker 1...Seeker 3...Seeker 2...start 'em...done...done...done...'
              -------Sequential Output-------- ---Sequential Input-- --Random--
              -Per Char- --Block--- -Rewrite-- -Per Char- --Block--- --Seeks---
Machine    MB K/sec %CPU K/sec %CPU K/sec %CPU K/sec %CPU K/sec %CPU  /sec %CPU
          100 123396 100.0 1258402 100.0 996583 100.0 126781 100.0 2187052 100.0 164190.1 299.0
```
The output includes the CPU time during each test, which at 100% is an indicator that Bonnie never blocked on disk I/O, instead always hitting from cache and staying on-CPU.

### Cache Flushing
Linux provides a way to flush (drop entries from) file system caches, which may be useful for benchmarking performance from a consistent and “cold” cache state, such as after system boot. This mechanism is described very simply in the kernel source documentation (Documentation/sysctl/vm.txt) as

```bash
To free pagecache:
        echo 1 > /proc/sys/vm/drop_caches
To free dentries and inodes:
        echo 2 > /proc/sys/vm/drop_caches
To free pagecache, dentries and inodes:
        echo 3 > /proc/sys/vm/drop_caches
```
## Tuning
Many tuning approaches have already been covered in Section  Methodology, including cache tuning and workload characterization. The latter can lead to the highest tuning wins by identifying and eliminating unnecessary work.

The specifics of tuning—the options available and what to set them to—depend on the operating system version, the file system type, and the intended workload. The following sections provide examples of what may be available and why they may need to be tuned. Covered are application calls and two example file system types: ext3 and ZFS. For tuning of the page cache Memory.

### zfs
ZFS supports a large number of tunable parameters (called properties) per file system, with a smaller number that can be set system-wide (/etc/system).

The file system properties can be listed using the zfs(1) command. For example
```bash
# zfs get all zones/var
NAME       PROPERTY              VALUE                  SOURCE
zones/var  type                  filesystem             -
zones/var  creation              Sat Nov 19  0:37 2011  -
zones/var  used                  60.2G                  -
zones/var  available             1.38T                  -
zones/var  referenced            60.2G                  -
zones/var  compressratio         1.00x                  -
zones/var  mounted               yes                    -
zones/var  quota                 none                   default
zones/var  reservation           none                   default
zones/var  recordsize            128K                   default
zones/var  mountpoint            legacy                 local
zones/var  sharenfs              off                    default
zones/var  checksum              on                     default
zones/var  compression           off                    inherited from zones
zones/var  atime                 off                    inherited from zones
[...]
```
The (truncated) output includes columns for the property name, current value, and source. The source shows how it was set: whether it was inherited from a higher-level ZFS dataset, the default, or set locally for that file system.

The parameters can also be set using the zfs(1M) command and are described in the zfs(1M) man page. Key parameters related to performance are listed in Table 8.

parameter |options | Description
--- | --- | ---
recordsize | 512 to 128 k | Suggested block size for files
compression | on/off/zjb/gzip/zle/lz4 | lightweight alogrithms can improve performance in some situation, by relieving back-end i/o congestion
atime| on/off| access timestamp updates
primarycache | all/none/metadata | ARC policy; Cache population to low priority file systems(archives) can be reduced by using "none" or "metadata" (only)
secondarycache| all/none/metadata'L2ARC policy
logbias | latency/throughput | adivce for synchronous writes: " Latency" uses log devices, whereas "throughput" uses pool devices
Sync | standard/always/disabled | synchronous write behavior

The most important parameter to tune is usually record size, to match the application I/O. It usually defaults to 128 Kbytes, which can be inefficient for small random I/O. Note that this does not apply to files that are smaller than the record size, which are saved using a dynamic record size equal to their file length.

Disabling atime can also improve performance (although its update behavior is already optimized), if those timestamps are not needed.

[Back to Top](#system-profiling-and-tuning)
