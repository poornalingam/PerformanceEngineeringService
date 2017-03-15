#  System Profiling and Tuning
- [Application profiling](./Application-Performance-Analysis.md)
  - [Application basics](#application-basics)
  - [Application performance Techniques](#application-performance-techniques)
  - [Programming languages](#programming-languages)
  - [Analysis methodologies](#analysis-methodologies)
  - [Static Performance Tuning](#static-performance-tuning)
- [CPU profiling](./CPU-Profiling.md)
- [Memory profiling](./Memory-profiling.md)
- [File Systems profiling](./filesystem-profiling.md)
- [Disk profiling](./disk-profiling.md)
- [Network profiling](./Network-profiling.md)
- [Cloud environment profiling](./Cloud-environment-profiling.md)

## Application performance analysis and tuning

## Application basics
Before diving into application performance, Performance Analyst should familiarize yourself with the role of the application, its basic characteristics, and its ecosystem in the industry. This forms the context within which you can understand application activity. It also gives you opportunities to learn about common performance issues and tuning and provides avenues for further study.

- What type of functionality the application perform and What is the role of the application? Is it a database server, web server, load balancer, file server, object store?

- How is the application configured, and why? Check if any tunable parameters related to performance have been changed, including buffer sizes, cache sizes, parallelism (processes or threads), and other options.
- Are application metrics provided, such as an operation rate? They may be provided by bundled tools or third-party tools, via API requests, or by processing operation logs.

- What operation logs does the application create? What logs can be enabled? What performance metrics, including latency, are available from the logs? For example, MySQL supports a slow query log, providing valuable performance details for each query slower than a certain threshold.

- Is the application the latest version? Have performance fixes or improvements been noted in the release notes for recent versions? What are the “performance” bugs for your version of the application? Is there a community for the application where performance findings are shared? Who are the recognized performance experts for the application?

### Objectives
A performance goal provides direction for your performance analysis work and helps you select which activities to perform. The goal may be

- **Latency** : a low application response time
- **Throughput** : a high application operation rate or data transfer rate
- **Resource utilization** : efficiency for a given application workload

It is better if these can be quantified, using metrics that may be derived from business or quality-of-service requirements (NFR). Examples are
- An average application request latency of 5 ms
- 95% of requests at a latency of 100 ms or less
- Elimination of latency outliers : zero requests beyond 1,000 ms
- A maximum throughput of at least 10,000 application requests per second per server
- Average disk utilization under 50% for 10,000 application requests per second

Now you can work on the limiters for that goal
- For latency, the limiter may be disk or network I/O.
- For throughput, it may be CPU usage.

### Common Case
Common methods for improving application performance. Some of these may make sense for one goal but not another; for example, selecting a larger I/O size may improve throughput at the expense of latency.

Picking areas to optimize at random may involve a great deal of work for not much gain.

One way to efficiently improve application performance is to find the most common code path for the production workload and begin by improving that.
- If the application is CPU-bound, that may mean the code paths that are frequently on-CPU.
- If the application is I/O-bound, you should be looking at the code paths that frequently lead to I/O.

These can be determined by analysis and profiling of the application, including studying stack traces. A higher level of context for understanding the common case may also be provided by application observability tools.

## Application performance Techniques
Some commonly used techniques by which application performance can be improved: selecting an I/O size, caching, buffering, polling, concurrency and parallelism, non-blocking I/O (async), and processor binding. For package application refer the application documentation to see which of these are used, and for any additional application-specific features.

### Selecting appropriate I/O size
Costs associated with performing I/O includes
- initializing buffers
- making an appropriate system call
- context switching
- allocating kernel metadata
- checking process privileges and limits
- mapping addresses to devices
- executing kernel
- driver code to deliver the I/O
- freeing metadata and buffers

For efficiency, **the more data transferred by each I/O, the better but not all cases. It increases I/O latency**.

**Increasing the I/O size is a common strategy used by applications to improve throughput**. It’s usually much more efficient to transfer 128 Kbytes as a single I/O than as 128 x 1 Kbyte I/O, considering any fixed per-I/O costs. Disk I/O has historically had a high per-I/O cost due to seek time.

There’s a downside when the application doesn’t need larger I/O sizes. A **database performing 8 Kbyte random reads may run more slowly with a 128 Kbyte I/O size, as 120 Kbytes of data transfer is wasted**. This introduces I/O latency, which can be lowered by selecting a smaller I/O size that more closely matches what the application is requesting. **Unnecessarily larger I/O sizes can also waste cache space.**

### Caching
The operating system uses caches to improve file system read performance and memory allocation performance; applications often use caches for a similar reason. Instead of always performing an expensive operation, the results of commonly performed operations may be stored in a local cache for future use. Configure the sizes to suit to system.

### Buffering
To improve write performance, data may be combined in a buffer before being sent to the next level. This **increases the I/O size and efficiency of the operation**. Depending on the type of writes, it may also increase write latency, as the first write to a buffer waits for subsequent writes before being sent. To maintain atomicity database may not allow write buffering.

###  Concurrency and Parallelism

Another approach is **event-based concurrency, whereby an application services different functions and switches between them when events occur**. For example, the Node.js runtime uses this approach. This provides concurrency but may do so using a single thread or process, which can eventually become a scalability bottleneck as it can utilize only one CPU.

Apart from increased throughput of CPU work, multiple threads (or processes) allow I/O to be performed concurrently, as other threads can execute while a thread blocked on I/O waits.

###  Non-Blocking I/O
The Unix process life cycle shows processes blocking and entering the sleep state during I/O. There are a couple of performance problems with this model:

- For **many concurrent I/O**, each I/O consumes a thread (or process) while it is blocked. In order to support many concurrent I/O, the application must create many threads, which have a cost associated with thread creation and destruction.
- For **frequent short-lived I/O**, the overhead of frequent context switching can consume CPU resources and add application latency.

The non-blocking I/O model issues I/O asynchronously, without blocking the current thread, which can then perform other work. This has been a key feature of Node.js, a server-side JavaScript application environment that directs code to be developed in non-blocking ways.

### Processor Binding - CPU affinity
Some applications force this behavior by binding themselves to CPUs. This can significantly improve performance for some systems. It can also reduce performance when the bindings conflict with other CPU bindings, such as device interrupt mappings to CPUs.

In cloud computing this design lead to scheduler latency as the bound CPUs are busy with other tenants, even though other CPUs are idle.

## programming languages

Interpreters and language virtual machines also provide different levels of performance observability support via their own specific tools. For the system performance analyst, basic profiling using these tools can lead to some quick wins. For example, high CPU usage may be identified as a result of garbage collection (GC), and then fixed via some commonly used tunables. Or it may be caused by a code path that can be found as a known bug in a bug database and fixed by upgrading the software version (this happens a lot). Or it may be caused by custom code path that can be found by aggregating the response time and latency.

### Garbage Collection

- **Memory growth**: There is less control of the application’s memory usage, which may grow when objects are not identified automatically as eligible to be freed by asynchronous garbage collection process. If the application grows too large, it may either hit its own limits or encounter system paging, severely harming performance.
- **CPU cost**: GC will typically run intermittently (Minor GC) and involves searching or scanning objects in memory. This consumes CPU resources, reducing what is available to the application for short periods. As the memory of the application grows, CPU consumption by GC may also grow and it also invoke (Major GC). In some cases, this can reach the point where GC continually consumes an entire CPU for while.
- **Latency outliers**: Application execution may be paused while GC executes, causing occasional application responses with high latency. This depends on the GC type: stop-the-world, incremental, or concurrent.

**GC is a common target for performance tuning, to reduce CPU cost and occurrence of latency outliers**. For example, the Java VM provides many tunable parameters to set the GC type, number of GC threads, maximum heap size, target heap free ratio, and more.

## Analysis Methodologies
methodologies for application analysis and tuning. The tools used for analysis.

Methodology| type
----|----
Thread state analysis| observational analysis
CPU profiling | observation analysis
Syscall analysis | observation analysis
I/O profiling | observation analysis
Workload Characterization | observation analysis, capacity planing
USE method | observational analysis
Drill-down analysis | observation analysis
Lock analysis| observation analysis
Static performance tuning | observation analysis, tuning

In addition to these, look for custom analysis techniques for the specific application and the programming language in which it is developed. These may consider logical behavior of the application, including known issues, and lead to some quick performance wins.

#### Thread State Analysis
By dividing each application’s thread time into a number of meaningful states.

##### Two State
At a minimum, there are two thread states:

- **On-CPU**: executing

- **Off-CPU**: waiting for a turn on-CPU, or for I/O, locks, paging, work, and so on

If time is largely spent on-CPU, CPU profiling can usually explain this quickly. This is the case for many performance issues.

If time is found to be spent off-CPU, various other methodologies can be used, although without a better starting point this can be time-consuming.

#### Seven State
Better starting points for the off-CPU cases.
- **System (Kernel) executing**: on-CPU
- **User Execution**: on-CPU
- **Runnable**: and waiting for a turn on-CPU
- **Anonymous paging**: runnable, but blocked waiting for anonymous page-ins
- **Sleeping**: waiting for I/O, including network, block, and data/text page-ins
- **Lock**: waiting to acquire a synchronization lock (waiting on someone else)
- **Idle**: waiting for work

Performance is improved by reducing the time in the first six of these states, which increases the time spent in idl. This would mean that application requests have lower latency, and the application can handle more load.

Once you’ve established in which of the first six states the threads are spending their time, you can investigate them further:

- **System and User Executing**: Check whether this is user- or kernel-mode time and the reason for CPU consumption by using profiling. Profiling can determine which code paths are consuming CPU and for how long, which can include time spent spinning on locks. Refer CPU Profiling.
- **Runnable**: Spending time in this state means the application needs more CPU resources. Examine CPU load for the entire system, and any CPU limits present for the application.
- **Anonymous paging**: A lack of available main memory for the application can cause anonymous paging and delays. Examine memory usage for the entire system and any memory limits present for the application.
- **Sleeping**: Analyze the resource on which the application is blocked. Refer: Syscall Analysis, and Refer I/O Profiling.
- **Lock**: Identify the lock, the thread holding it, and the reason why the holder held it for so long. The reason may be that the holder was blocked on another lock, which requires further unwinding. This is an advanced activity, usually performed by the software developer who has intimate knowledge of the application and its locking hierarchy.

An **application worker thread may wait on a conditional variable for work (lock state), or for network I/O (sleeping state)**. So when you see large sleeping and lock state times, remember to drill down a little to check if this is really idle time.

#### Linux
Summarizes how these thread states may be measured on Linux.

**execting**:  The time spent executing is not hard to determine: top(1) reports this as %CPU.
**Runnable** is tracked by the kernel schedstats feature and is exposed via /proc/\*/schedstat. The perf sched tool can also provide metrics for understanding time spent runnable and waiting.

**Time waiting** for anonymous swapping can be measured by the kernel delay accounting feature, provided it is enabled. use the kernel documented program getdelays.c in dynamic system Observability page. Another approach is to use tracing tools such as DTrace or SystemTap.

**Time blocked** in the sleeping state can be loosely estimated using other tools, for example, pidstat -d to determine if a process is performing disk I/O, and probably sleeping.

If the **application is stuck in the sleeping state for very long intervals (seconds), you can try pstack(1) to determine why**. This takes a single snapshot of the threads and their user stack traces, which should include the sleeping threads and the reason they are sleeping. **Be warned**, however: pstack(1) may briefly pause the target while it does this, so use with caution.

Lock time can be investigated using tracing tools.

### CPU Profiling
Profiling is an important activity that is summarized here from the application perspective.

The intent is to determine why an application is consuming CPU resources. An effective technique is to sample the on-CPU user-level stack trace and consolidate the results. The stack traces shows the code path taken, which can reveal both high- and low-level reasons for the application consuming CPU.

Sampling stack traces can generate many thousands of lines of output to examine, even when summarizing the output to print only unique stacks. One way to understand the profile quickly is to visualize it using flame graphs.

CPU profiling can be performed using DTrace and perf tools. Refer CPU profiling page for [more details](/CPU-Profiling.md).

#### Syscall Analysis

This is performed using the strace command.

The traditional style of syscall tracing involves setting breakpoints for syscall entry and return. These are invasive, and for applications with high syscall rates their performance may be worsened by an order of magnitude.

Depending on application performance requirements, this style of tracing may be acceptable to use for short durations, to determine the syscall types being called.

```bash
$ strace -c -p 1884
Process 1884 attached - interrupt to quit
^CProcess 1884 detached
% time     seconds  usecs/call     calls    errors syscall
------ ----------- ----------- --------- --------- ----------------
 83.29    0.007994           9       911       455 wait4
 14.41    0.001383           3       455           clone
  0.85    0.000082           0      2275           ioctl
  0.68    0.000065           0       910           close
  0.63    0.000060           0      4551           rt_sigprocmask
  0.15    0.000014           0       455           setpgid
  0.00    0.000000           0       455           rt_sigreturn
  0.00    0.000000           0       455           pipe
------ ----------- ----------- --------- --------- ----------------
100.00    0.009598                 10467       455 total
```
This would be of greater use if the overhead was not such a problem.

This one-liner counts syscalls (using an aggregation) for processes named "postgres" (PostgreSQL database):
```bash
# dtrace -n 'syscall:::entry /execname == "postgres"/ { @[probefunc] = count(); }'
dtrace: description 'syscall:::entry ' matched 233 probes
^C

  setitimer                                                         4
  semsys                                                           22
  open64                                                           35
  kill                                                             79
  lwp_sigmask                                                      79
  setcontext                                                       79
  write                                                           126
  fcntl                                                           252
  pollsys                                                        2498
  read                                                           2750
  send                                                           9542
  recv                                                          12096
  llseek                                                        27925
```
During tracing, the llseek() syscall was executed the most—27,925 times.

####  I/O Profiling
Similar to the role of CPU profiling, I/O profiling determines why and how I/O-related system calls are being performed. This can be done using DTrace, examining the user-level stack traces for system calls.

For example, this one-liner traces PostgreSQL read() syscalls, gathers the user-level stack trace, and aggregates them:

`# dtrace -n 'syscall::read:entry /execname == "postgres"/ {
    @[ustack()] = count(); }'`

The output shows user-level stacks and then a count for the number of occurrences.

#### Lock Analysis
For multithreaded applications, locks can become a bottleneck, inhibiting parallelism and scalability. They can be analyzed by

- Checking for contention
- Checking for excessive hold times

The first identifies whether there is a problem now. Excessive hold times are not necessarily a problem, but they may be in the future, with more parallel load. For each, try to identify the name of the lock (if it exists) and the code path that led to using it.

## Static Performance Tuning
Static performance tuning focuses on issues of the configured environment. For application performance, examine the following aspects of the static configuration:

-  What known performance issues are there with the application? Is there a bug database that can be searched?
-  If it was configured or tuned differently from the defaults, what was the reason? (Was it based on measurements and analysis, or guesswork?)
-  Does the application employ a cache of objects? How is it sized?
-  Does the application run concurrently? How is that configured (e.g., thread pool sizing)?
-  What system libraries does the application use? What versions are they?
-  What memory allocator does the application use?
-  Is the application configured to use large pages for its heap?
-  Is the application compiled? What version of the compiler? What compiler options and optimizations? 64-bit?
-  Are there system-imposed limits or resource controls for CPU, memory, file system, disk, or network usage? (These are common with cloud computing.)

Answering these questions may reveal configuration choices that have been overlooked.

[Back to top](#application-performance-analysis-and-tuning)
