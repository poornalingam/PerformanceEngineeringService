# System Profiling and Tuning

- [Application profiling](./Application-Performance-Analysis.md)
- [CPU profiling](./CPU-Profiling.md)
  - [CPU basics](#Basics)
  - [Observation methodologies](#methodologies)
  - [Analysis](#analysis)
- [Memory profiling](./Memory-profiling.md)
- [File Systems profiling](./filesystem-profiling.md)
- [Disk profiling](./disk-profiling.md)
- [Network profiling](./Network-profiling.md)
- [Cloud environment profiling](./Cloud-environment-profiling.md)

## CPU profiling and tuning

## Basics
CPU resources are resources available, process threads (or tasks) will queue, waiting their turn. Waiting can add significant latency during the runtime of applications, degrading performance.

At a high level, CPU usage by process, thread, or task can be examined. At a lower level, the code path within applications and the kernel can be profiled and studied. At the lowest level, CPU instruction execution and cycle behavior can be studied.

This page consists of five parts:

- **Background** CPU-related terminology, basic models of CPUs, and key CPU performance concepts.
- **Architecture** processor and kernel scheduler architecture.
- **Methodology** performance analysis methodologies, both observational and experimental.
- **Analysis** CPU performance analysis tools on Linux systems, including profiling, tracing, and visualizations.
- **Tuning** examples of tunable parameters.

The effects of memory I/O on CPU performance are covered, including CPU cycles stalled on memory and the performance of CPU caches.

## Methodologies
Various Methodology and exercises for CPU analysis and Tuning

Methodology | Type
---|---
Tools method | Observational Analysis
USE method | Observational Analysis
Workload Characterization| Observational Analysis, capacity planing
Profiling |Observational Analysis
Cycle analysis |Observational Analysis, capacity planing
Performance Monitoring| Observational Analysis, capacity planing
Priority tuning | tuning
Resource Controls| tuning
CPU binding| tuning
Micro-benchmarking | experimental analysis
Scaling | capacity planing, tuning


### Tools method
The tools method is a process of iterating over available tools, examining key metrics they provide. While this is a simple methodology, it can **overlook issues for which the tools provide poor or no visibility, and it can be time-consuming to perform.**

For CPUs, the tools method can involve checking the following:

- **uptime**: Check load averages to see if CPU load is increasing or decreasing over time. A load average over the number of CPUs in the system usually indicates saturation.
- **vmstat**: Run vmstat per second, and check the idle columns to see how much headroom there is. Less than 10% can be a problem.
- **mpstat**: Check for individual hot (busy) CPUs, identifying a possible thread scalability problem.
- **top/prstat**: See which processes and users are the top CPU consumers.
- **pidstat/prstat**: Break down the top CPU consumers into user- and system-time.
- **perf/dtrace/stap/oprofile**: Profile CPU usage stack traces for either user- or kernel-time, to identify why the CPUs are in use.
- **perf/cpustat**: Measure CPI.

### USE method
The USE method is for identifying bottlenecks and errors across all components, early in a performance investigation, before deeper and more time-consuming strategies are followed.

For each CPU, check for

- **Utilization**: the time the CPU was busy (not in the idle thread)
- **Saturation**: the degree to which runnable threads are queued waiting their turn on-CPU
- **Errors**: CPU errors, including correctable errors

### Workload Characterization
Characterizing the load applied is important in capacity planning, benchmarking, and simulating workloads. It can also lead to some of the largest performance gains by identifying unnecessary work that can be eliminated.

Basic attributes for characterizing CPU workload are
- Load averages (utilization + saturation)
- User-time to system-time ratio
- Syscall rate
- Voluntary context switch rate
- Interrupt rate

High system-time shows time spent in the kernel instead, which may be further understood by the syscall and interrupt rate. **I/O-bound workloads have higher system-time, syscalls, and also voluntary context switches as threads block waiting for I/O.**

On **busiest application server, _the load average varies between 2 and 8 out of 10 during the day depending on the number of active_ clients. The _user/system ratio is 60/40_, as this is an I/O-intensive workload performing around 100 K syscalls/s, and a high rate of voluntary context switches.**

#### Advanced Workload Characterization/Checklist
These are listed here as questions for consideration, which may also serve as a checklist when studying CPU issues thoroughly:
- What is the CPU utilization system-wide? Per CPU?
- How parallel is the CPU load? Is it single-threaded? How many threads?
- Which applications or users are using the CPUs? How much?
- Which kernel threads are using the CPUs? How much?
- What is the CPU usage of interrupts?
- What is the CPU interconnect utilization?
- Why are the CPUs being used (user- and kernel-level call paths)?
- What types of stall cycles are encountered?

### Profiling
Profiling builds a picture of the target for study. CPU usage can be profiled by sampling the state of the CPUs at timed intervals, following these steps:

Some profiling tools, including DTrace, systemtap, perf allow real-time processing of the captured data, which can be analyzed while sampling is still occurring.

### Cycle Analysis
By using the CPU performance counters (CPCs), CPU utilization can be understood at the cycle level. This may **reveal that cycles are spent stalled on Level 1, 2, or 3 cache misses, memory I/O, or resource I/O, or spent on floating-point operations or other activity**. This information may lead to performance wins by adjusting compiler options or changing the code.

Begin cycle analysis by measuring CPI. If **CPI is high, continue to investigate types of stall cycles. If CPI is low, look for ways in the code to reduce instructions performed**. The values for “high” or “low” CPI depend on your processor: low could be less than one, and high could be greater than ten. Assess the load in healthy sample system.

Apart from measuring counter values, CPC can be configured to interrupt the kernel on the overflow of a given value. For example, at every 10,000 Level 2 cache misses, the kernel could be interrupted to gather a stack backtrace.

### Priority Tuning
The nice value is useful for adjusting process priority.

Unix has always provided a nice() system call for adjusting process priority, which sets a nice-ness value. Positive nice values result in lower process priority (nicer), and negative values—which can be set only by the superuser (root)—result in higher priority. A nice() command became available to launch programs with nice values, and a renice() command was later added to adjust the nice value of already running processes.

**The value of 16 is recommended to users who wish to execute long-running programs without flak from the administration.** Best suitable for batch process server.

This is **most effective when there is contention for CPUs, causing scheduler latency for high-priority work**. _Your task is to identify low-priority work, which may **include monitoring agents and scheduled backups**, that can be modified to start with a nice value._ Analysis may also be performed to check that the tuning is effective, and that the scheduler latency remains low for high-priority work.

### CPU Binding
Another way to tune CPU performance involves binding processes and threads to individual CPUs, or collections of CPUs. This can increase CPU cache warmth for the process, improving its memory I/O performance.

On Linux-based systems, the exclusive CPU sets approach can be implemented using cpusets.

## Analysis
CPU performance analysis tools.

Tool | Description
--- | ---
uptime; w | load averages
vmstat | includes system wide CPU averages
mpstat | per-CPU statistics
sar  | historical statistics
top | monitoring per-process/thread cpu usage
pidstat | per-process/thread CPU breakdowns
time | time a command, with CPU breakdowns
Dtrace, perf | CPU profiling and tracing
perf | CPU performance counter analysis

The list begins with tools for CPU statistics, and then drills down to tools for deeper analysis including code-path profiling and CPU cycle analysis.

### uptime
uptime is one of several commands that print the system load averages:

```bash
$ uptime
  9:04pm  up 268 day(s), 10:16,  2 users,  load average: 7.76, 8.32, 8.60
```
The **load average indicates the demand for CPU resources and is calculated by summing the number of threads running (utilization) and the number that are queued waiting to run (saturation).**

As a modern example, **a system with 64 CPUs has a load average of 128. This means that on average there is always one thread running on each CPU, and one thread waiting for each CPU.** The same system with a load average of ten would indicate significant headroom, as it could run another 54 CPU-bound threads before all CPUs are busy.

Incorporate other resource load is to use separate load averages for each resource type.

### vmstat
The virtual memory statistics command, vmstat, prints system-wide CPU averages in the last few columns, and a count of runnable threads in the first column.

```bash
$ vmstat 1
procs -----------memory---------- ---swap-- -----io---- -system-- ----cpu----
 r  b   swpd   free   buff  cache   si   so    bi    bo   in   cs us sy id wa
15  0   2852 46686812 279456 1401196    0    0     0     0    0    0  0  0 100  0
16  0   2852 46685192 279456 1401196    0    0     0     0 2136 36607 56 33 11  0
15  0   2852 46685952 279456 1401196    0    0     0    56 2150 36905 54 35 11  0
15  0   2852 46685960 279456 1401196    0    0     0     0 2173 36645 54 33 13  0
[...]
```
The **first line of output is the summary-since-boot**, with the exception of r on Linux—which begins by showing current values.

- r: **run-queue length** —the total number of runnable threads (see below)
- wa: wait I/O, which **measures CPU idle when threads are blocked on disk I/O**
- **st: stolen (not shown in the output), which for virtualized environments shows CPU time spent servicing other tenants**

**All of these values are system-wide averages across all CPUs, with the exception of r, which is the total. The r column is the total number of tasks waiting plus those running.** R description in man page is error.

### mpstat
The multiprocessor statistics tool, mpstat, can report statistics per CPU.

```bash
$ mpstat -P ALL 1
02:47:49   CPU    %usr  %nice   %sys %iowait   %irq  %soft %steal %guest  %idle
02:47:50   all   54.37   0.00  33.12    0.00   0.00   0.00   0.00   0.00  12.50
02:47:50     0   22.00   0.00  57.00    0.00   0.00   0.00   0.00   0.00  21.00
02:47:50     1   19.00   0.00  65.00    0.00   0.00   0.00   0.00   0.00  16.00
02:47:50     2   24.00   0.00  52.00    0.00   0.00   0.00   0.00   0.00  24.00
02:47:50     3  100.00   0.00   0.00    0.00   0.00   0.00   0.00   0.00   0.00
02:47:50     4  100.00   0.00   0.00    0.00   0.00   0.00   0.00   0.00   0.00
02:47:50     5  100.00   0.00   0.00    0.00   0.00   0.00   0.00   0.00   0.00
02:47:50     6  100.00   0.00   0.00    0.00   0.00   0.00   0.00   0.00   0.00
02:47:50     7   16.00   0.00  63.00    0.00   0.00   0.00   0.00   0.00  21.00
02:47:50     8  100.00   0.00   0.00    0.00   0.00   0.00   0.00   0.00   0.00
02:47:50     9   11.00   0.00  53.00    0.00   0.00   0.00   0.00   0.00  36.00
02:47:50    10  100.00   0.00   0.00    0.00   0.00   0.00   0.00   0.00   0.00
02:47:50    11   28.00   0.00  61.00    0.00   0.00   0.00   0.00   0.00  11.00
02:47:50    12   20.00   0.00  63.00    0.00   0.00   0.00   0.00   0.00  17.00
02:47:50    13   12.00   0.00  56.00    0.00   0.00   0.00   0.00   0.00  32.00
02:47:50    14   18.00   0.00  60.00    0.00   0.00   0.00   0.00   0.00  22.00
02:47:50    15  100.00   0.00   0.00    0.00   0.00   0.00   0.00   0.00   0.00
[...]
```

The -P ALL option was used to print the per-CPU report. By default, mpstat(1) prints only the system-wide summary line (all). The columns are

- **CPU**: logical CPU ID, or all for summary
- **%nice**: user-time for processes with a nice’d priority
- **%steal**: time spent servicing other tenants (Cloud specific)
- **%guest**: CPU time spent in guest virtual machines

Key columns are %usr, %sys, and %idle.

### sar
The system activity reporter, sar, can be used to observe current activity and can be configured to archive and report historical statistics.

- -P ALL: same as mpstat -P ALL
- -u: same as mpstat’s default output: system-wide average only
- -q: includes run-queue size as runq-sz (waiting + running, the same as vmstat’s r) and load averages

### pidstat
pidstat tool prints CPU usage by process or thread, including user- and system-time breakdowns. By default, a rolling output is printed of only active processes.

```bash
$ pidstat 1
Linux 2.6.35-32-server (dev7)   11/12/12        _x86_64_        (16 CPU)

22:24:42          PID    %usr %system  %guest    %CPU   CPU  Command
22:24:43         7814    0.00    1.98    0.00    1.98     3  tar
22:24:43         7815   97.03    2.97    0.00  100.00    11  gzip

22:24:43          PID    %usr %system  %guest    %CPU   CPU  Command
22:24:44          448    0.00    1.00    0.00    1.00     0  kjournald
22:24:44         7814    0.00    2.00    0.00    2.00     3  tar
22:24:44         7815   97.00    3.00    0.00  100.00    11  gzip
22:24:44         7816    0.00    2.00    0.00    2.00     2  pidstat
[...]
```
This example captured a system backup, involving a tar command to read files from the file system, and the gzip command to compress them. The user-time for gzip is high, as expected, as it becomes CPU-bound in compression code. The tar command spends more time in the kernel, reading from the file system.

The -p ALL option can be used to print all processes, including those that are idle. -t prints per-thread statistics.

### DTrace

Previous tools, including mpstat and top, showed system-time—CPU time spent in the kernel. DTrace can be used to identify what the kernel is doing.

### systemtap
SystemTap can also be used on Linux systems for tracing of scheduler events.

### Perf

The perf command has evolved and become a collection of tools for profiling and tracing, now called Linux Performance Events (LPE). Each tool is selected as a subcommand. For example, perf stat executes the stat command, which provides CPC-based statistics. These commands are listed in the USAGE message, and a page is reproduced here.

Command | Description
--- | ---
annotate | Read perf.data (created by perf record) and display annotated code.
diff | Read two perf.data and display the differential profile
evlist | list the event names in a perf.data file
inject | Filter to augment the events stream with additional information
kmem | Tool to trace/measure kernel memory (slab) properties
kvm | tool to trace/measure kvm gust os
list | list all symbolic event types
lock | Analyze lock events
probe | Define new dynamic tracepoints
record | Run a command and record its profile into perf.data
report | Read perf.data and display the profile
sched | Tool to trace/measure scheduler properties (latencies)
script | Read perf.data and display trace output
stat | Run a command and gather performance counter statistics
timechart | Tool to visualize total system behavior during a workload
top | system profiling tool

Key commands are demonstrated in the following sections.

#### System Profiling
perf can be used to profile CPU call paths, summarizing where CPU time is spent in both kernel- and user-space. This is performed by the record command, which captures samples at regular intervals to a perf.data file. A report command is then used to view the file.

In the following example, all CPUs (-a) are sampled with call stacks (-g) at 997 Hz (-F 997) for 10 s (sleep 10). The --stdio option is used to print all the output, instead of operating in interactive mode.

```bash
# perf record -a -g -F 997 sleep 10
[ perf record: Woken up 44 times to write data ]
[ perf record: Captured and wrote 13.251 MB perf.data (~578952 samples) ]

# perf report --stdio
[...]
# Overhead      Command      Shared Object                              Symbol
# ........  ...........  .................  ..................................
#
    72.98%      swapper  [kernel.kallsyms]  [k] native_safe_halt
                |
                --- native_safe_halt
                    default_idle
                    cpu_idle
                    rest_init
                    start_kernel
                    x86_64_start_reservations
                    x86_64_start_kernel

     9.43%           dd  [kernel.kallsyms]  [k] acpi_pm_read
                     |
                     --- acpi_pm_read
                         ktime_get_ts
                        |
                        |--87.75%-- __delayacct_blkio_start
                        |          io_schedule_timeout
                        |          balance_dirty_pages_ratelimited_nr
                        |          generic_file_buffered_write
                        |          __generic_file_aio_write
                        |          generic_file_aio_write
                        |          ext4_file_write
                        |          do_sync_write
                        |          vfs_write
                        |          sys_write
                        |          system_call
                        |          __GI___libc_write
                        |
[...]

```

These sample counts are given as percentages, which show where the CPU time was spent. This example indicates that 72.98% of time was spent in the idle thread, and 9.43% of time in the dd process. Out of that 9.43%, 87.5% is composed of the stack shown, which is for ext4_file_write().

These **kernel and process symbols are available only if their debuginfo files are available**; otherwise hex addresses are shown.

#### Process Profiling

Apart from profiling across all CPUs, individual processes can be targeted. The following command executes the command and creates the perf.data file:

`# perf record -g command`
As before, debuginfo must be available for perf(1) to translate symbols when viewing the report.

#### Scheduler Latency

The sched command records and reports scheduler statistics. For example. Note: it creates larger volume of data.

```bash
# perf sched record sleep 10
[ perf record: Woken up 108 times to write data ]
[ perf record: Captured and wrote 1723.874 MB perf.data (~75317184 samples) ]

# perf sched latency

 ---------------------------------------------------------------------------------------------------------------
  Task                  |   Runtime ms  | Switches | Average delay ms | Maximum delay
ms | Maximum delay at     |
 ---------------------------------------------------------------------------------------------------------------
  kblockd/0:91          |      0.009 ms |        1 | avg:    1.193 ms | max:    1.193
ms | max at: 105455.615096 s
  dd:8439               |   9691.404 ms |      763 | avg:    0.363 ms | max:   29.953
ms | max at: 105456.540771 s
  perf_2.6.35-32:8440   |   8082.543 ms |      818 | avg:    0.362 ms | max:   29.956
ms | max at: 105460.734775 s
  kjournald:419         |    462.561 ms |      457 | avg:    0.064 ms | max:   12.112
ms | max at: 105459.815203 s
[...]
  INFO: 0.976% lost events (167317 out of 17138781, in 3 chunks)
  INFO: 0.178% state machine bugs (4766 out of 2673759) (due to lost events?)
  INFO: 0.000% context switch bugs (3 out of 2673759) (due to lost events?)
  ```

This shows the average and maximum scheduler latency while tracing.

Scheduler events are frequent, so this type of tracing incurs CPU and storage overhead. The perf.data file in this example was 1.7 Gbytes for 10 s of tracing.

#### Stat

The stat command provides a high-level summary of CPU cycle behavior based on CPC. In the following example it launches a gzip command:

```bash
$ perf stat gzip file1

 Performance counter stats for 'gzip perf.data':

       62250.620881  task-clock-msecs         #      0.998 CPUs
                 65  context-switches         #      0.000 M/sec
                  1  CPU-migrations           #      0.000 M/sec
                211  page-faults              #      0.000 M/sec
       149282502161  cycles                   #   2398.089 M/sec
       227631116972  instructions             #      1.525 IPC
        39078733567  branches                 #    627.765 M/sec
         1802924170  branch-misses            #      4.614 %
           87791362  cache-references         #      1.410 M/sec
           24187334  cache-misses             #      0.389 M/sec

       62.355529199  seconds time elapsed
  ```
  The statistics include the cycle and instruction count, and the IPC (inverse of CPI). As described earlier, this is an extremely useful high-level metric for determining the types of cycles occurring and how many of them are stall cycles.

  The following lists other counters that can be examined:

```bash
# perf list

List of pre-defined events (to be used in -e):

  cpu-cycles OR cycles                       [Hardware event]
  instructions                               [Hardware event]
  cache-references                           [Hardware event]
  cache-misses                               [Hardware event]
  branch-instructions OR branches            [Hardware event]
  branch-misses                              [Hardware event]
  bus-cycles                                 [Hardware event]
[...]
  L1-dcache-loads                            [Hardware cache event]
  L1-dcache-load-misses                      [Hardware cache event]
  L1-dcache-stores                           [Hardware cache event]
  L1-dcache-store-misses                     [Hardware cache event]
[...]
```
Look for both “Hardware event” and “Hardware cache event.”

These events can be specified using –e. For example (this is from an Intel Xeon):

```bash
$ perf stat -e instructions,cycles,L1-dcache-load-misses,LLC-load-misses,dTLB-load-
misses gzip file1

 Performance counter stats for 'gzip file1':

        12278136571  instructions             #      2.199 IPC
         5582247352  cycles
           90367344  L1-dcache-load-misses
            1227085  LLC-load-misses
             685149  dTLB-load-misses

        2.332492555  seconds time elapsed
```
Apart from instructions and cycles, this example also measured the following:

- **L1-dcache-load-misses**: Level 1 data cache load misses. This gives you a measure of the memory load caused by the application, after some loads have been returned from the Level 1 cache. It can be compared with other L1 event counters to determine cache hit rate.

- **LLC-load-misses**: Last level cache load misses. After the last level, this accesses main memory, and so this is a measure of main memory load. The difference between this and L1-dcache-load-misses gives an idea (other counters are needed for completeness) of the effectiveness of the CPU caches beyond Level 1.

- **dTLB-load-misses**: Data translation lookaside buffer misses. This shows the **effectiveness of the MMU to cache page mappings** for the workload and can measure the size of the memory workload (working set).

#### Software Tracing

perf record -e can be used with various software instrumentation points for tracing activity of the kernel scheduler. These include software events and tracepoint events (static probes), as listed by perf list.

`# perf list`

The following example uses the context switch software event to trace when applications leave the CPU and collects call stacks for 10 s:

```bash
# perf record -f -g -a -e context-switches sleep 10
[ perf record: Woken up 1 times to write data ]
[ perf record: Captured and wrote 0.417 MB perf.data (~18202 samples) ]
# perf report --stdio
# ========
# captured on: Wed Apr 10 19:52:19 2013
# hostname : 9d219ce8-cf52-409f-a14a-b210850f3231
[...]
#
# Events: 2K context-switches
#
# Overhead    Command      Shared Object      Symbol
# ........  .........  .................  ..........
#
    47.60%       perl  [kernel.kallsyms]  [k] __schedule
                 |
                 --- __schedule
                     schedule
                     retint_careful
                    |
                    |--50.11%-- Perl_pp_unstack
                    |
                    |--26.40%-- Perl_pp_stub
                    |
                     --23.50%-- Perl_runops_standard

    25.66%        tar  [kernel.kallsyms]  [k] __schedule
                  |
                  --- __schedule
                     |
                     |--99.72%-- schedule
                     |          |
                     |          |--99.90%-- io_schedule
                     |          |          sleep_on_buffer
                     |          |          __wait_on_bit
                     |          |          out_of_line_wait_on_bit
                     |          |          __wait_on_buffer
                     |          |          |
                     |          |          |--99.21%-- ext4_bread
                     |          |          |          |
                     |          |          |          |--99.72%-- htree_dirbl...
                     |          |          |          |          ext4_htree_f...
                     |          |          |          |          ext4_readdir
                     |          |          |          |          vfs_readdir
                     |          |          |          |          sys_getdents
                     |          |          |          |          system_call
                     |          |          |          |          __getdents64
                     |          |          |           --0.28%-- [...]
                     |          |          |
                     |          |           --0.79%-- __ext4_get_inode_loc
[...]
```

This truncated output shows two applications, perl and tar, and their call stacks when they context switched. Reading the stacks shows the tar program was sleeping on file system (ext4) reads. The perl program was involuntary context switched as it is performing heavy compute, although that isn’t clear from this output alone.

More information can be found using the sched tracepoint events.

[Disks, includes another example of static tracing with perf: block I/O tracepoints.]()

[Network, includes an example of dynamic tracing with perf for the tcp_sendmsg() kernel function.]()

## Other tools
Other CPU performance tools include

- **htop**: includes ASCII bar charts for CPU usage and has a more powerful interactive interface than the original top.

- **atop**: includes many more system-wide statistics and uses process accounting to catch the presence of short-lived processes.

- **/proc/cpuinfo**: This can be read to see processor details, including clock speed and feature flags.

- **getdelays.c**: This is an example of delay accounting observability and includes CPU scheduler latency per process.

- **Image valgrind**: a memory debugging and profiling toolkit . It contains callgrind, a tool to trace function calls and gather a call graph, which can be visualized using kcachegrind; and cachegrind for analysis of hardware cache usage by a given program.

- **SysBench**: The SysBench system benchmark suite has a simple CPU benchmark tool that calculates prime numbers.

```bash
# sysbench --num-threads=8 --test=cpu --cpu-max-prime=100000 run
sysbench 0.4.12:  multi-threaded system evaluation benchmark

Running the test with following options:
Number of threads: 8

Doing CPU performance benchmark

Threads started!
Done.

Maximum prime number checked in CPU test: 100000


Test execution summary:
    total time:                          30.4125s
    total number of events:              10000
    total time taken by event execution: 243.2310
    per-request statistics:
         min:                                 24.31ms
         avg:                                 24.32ms
         max:                                 32.44ms
         approx.  95 percentile:              24.32ms

Threads fairness:
    events (avg/stddev):           1250.0000/1.22
    execution time (avg/stddev):   30.4039/0.01
```

This executed eight threads, with a maximum prime number of 100,000. The runtime was 30.4 s, which can be used for comparison with the results from other systems or configurations.

- **perf_events**: Use perf_events for **CPU profiling**. The profile can be visualized as a flame graph.  [More details](http://www.brendangregg.com/FlameGraphs/cpuflamegraphs.html)

```bat
git clone --depth 1 https://github.com/brendangregg/FlameGraph
perf record -F 99 -a -g -- sleep 30
perf script | ./FlameGraph/stackcollapse-perf.pl | ./FlameGraph/flamegraph.pl > perf.svg
```

![Flame Graph](/Images/cpu-bash-flamegraph.png)

#### Documentation
For more on perf, see its man pages, documentation in the Linux kernel source under tools/perf/Documentation, the [“Perf Tutorial”](https://perf.wiki.kernel.org/index.php/Tutorial) , and [“The Unofficial Linux Perf Events Web-Page”](www.eece.maine.edu/~vweaver/projects/perf_events).



[Back to Top](#cpu-profiling-and-tuning)
