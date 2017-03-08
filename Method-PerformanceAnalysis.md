
# Performance Analysis Methodologies
Numerous methodologies and procedures for system performance analysis and tuning are available. We will be discussing  few methodologies to analyze performance in this section which are relevant to us.
- [Tools Methods](#tools-methods)
- [USE Methods](#use-methods)
- [Workload Characterization](#workload-characterization)
- [Latency Methods](#latency-methods)
- [Event Tracing](#event-tracing)

## Tools Methods
A tools-oriented approach is as follows:
- List available performance tools (optionally, install or purchase more).
- For each tool, list useful metrics it provides.
- For each metric, list possible rules for interpretation.

The result of this is a prescriptive checklist showing which tool to run, which metrics to read, and how to interpret them. While this can be fairly effective, it relies exclusively on available (or known) tools, which can provide an incomplete view of the system.

## USE methods
The utilization, saturation, and errors (USE) method should be used early in a performance investigation, to identify systemic bottlenecks. It can be summarized this way: For every resource, check utilization, saturation, and errors.

These terms are defined as follows:
- **Resource**: all physical server functional components (CPUs, busses, . . .). Some software resources can also be examined, provided the metrics make sense.
- **Utilization**: for a set time interval, the percentage of time that the resource was busy servicing work. While busy, the resource may still be able to accept more work; the degree to which it cannot do so is identified by saturation.
- **Saturation**: the degree to which the resource has extra work that it can’t service, often waiting on a queue.
- **Errors**: the count of error events.

### Procedures
Errors are placed first before utilization and saturation are checked. Errors are usually quick and easy to interpret, and it can be time-efficient to rule them out before investigating the other metrics.

![USE](/Images/USE-method.jpg)

The USE method metrics are usually expressed as follows:

- **Utilization**: as a percent over a time interval (e.g., “One CPU is running at 90% utilization”). Look for 5 minutes interval.
- **Saturation**: as a wait-queue length (e.g., “The CPUs have an average run-queue length of four”)
- **Errors**: number of errors reported (e.g., “This network interface has had 50 late collisions”)

### Resource list

The first step in the USE method is to create a list of resources. Try to be as complete as possible.

- CPUs: sockets, cores, hardware threads (virtual CPUs)
- Main memory: DRAM
- Network interfaces: Ethernet ports
- Storage devices: disks
- Controllers: storage, network
- Interconnects: CPU, memory, I/O

The most common issues are usually found with the easier metrics (e.g., CPU saturation, memory capacity saturation, network interface utilization, disk utilization), so these can be checked first.

Resource | Type | Metrics
:--------|:--------:|:-------
CPU | Utilization| CPU utilization (either per CPU or System-wide avg)
CPU|saturation| dispatcher-queue length (run-queue length)
Memory|utilization|available free memory (System-wide)
Memory|Saturation|anonymous paging or thread swapping (page scanning is another indicator), out-of-memory events
Network interface|utilization|receive throughput/max bandwidth, transmit throughput/max bandwidth
Storage device I/O| utilization|device busy percent
Storage device I/O|saturation|wait-queue-length
storage device I/O| errors| device errors ("soft","hard")
CPU | errors | Example: correctable CPU cache error-correcting code (ECC) events or faulted CPUs
Memory| errors| Example: failed malloc()s (although this is usually due to virtual memory exhaustion, not physical)
Network | saturation | saturation-related network interface or OS errors, e.g., Linux "overruns"
Storage controller| utilization | depends on the controller;it may have a maximum IOPS or throughput that can be checked against current activity
CPU interconnects| utilization | per-port throughput/maximum bandwidth (CPU performance counters)
Memory interconnect| saturation | memory stall cycles, high cycles per instruction (CPU performance counters)
I/O interconnect | utilization | bus throughput/maximum bandwidth (performance counters may exist on your HW, e.g., Intel "uncore" events)

Some of these may not be available from standard operating system tools and may require the use of dynamic tracing or the CPU performance counter facility.

### Software Resources
Some software resources can be similarly examined. This usually applies to smaller components of software, not entire applications, for example:

- **Mutex locks:** Utilization may be defined as the time the lock was held, saturation by those threads queued waiting on the lock.
- **Thread pools:** Utilization may be defined as the time threads were busy processing work, saturation by the number of requests waiting to be serviced by the thread pool.
- **Process/thread capacity:** The system may have a limited number of processes or threads, whose current usage may be defined as utilization; waiting on allocation may be saturation; and errors are when the allocation failed (e.g., “cannot fork”).
- **File descriptor capacity:** similar to process/thread capacity, but for file descriptors.

If the metrics work well in your case, use them; otherwise, alternative methodologies such as latency analysis can be applied.

### Cloud Computing

In a cloud computing environment, software resource controls may be in place to limit or throttle tenants who are sharing one system. OS virtualization (SmartOS Zones) imposes memory limits, CPU limits, and storage I/O throttling. Each of these resource limits can be examined with the USE method, similarly to examining the physical resources.

## Workload Characterization

Workload characterization is a simple and effective method for identifying a class of issues: those due to the load applied. It focuses on the input to the system, rather than the resulting performance. Your system may have no architectural or configuration issues present, but it is under more load than it can reasonably handle.

Workloads can be characterized by answering the following questions:
- Who is causing the load? Process ID, user ID, remote IP address?
- Why is the load being called? Code path, stack trace?
- What are the load characteristics? IOPS, throughput, direction (read/write), type? Include variance (standard deviation) where appropriate.
- How is the load changing over time? Is there a daily pattern?

It can be useful to check all of these, even when you have strong expectations about what the answers will be, because you may be surprised. The best performance wins are the result of eliminating unnecessary work. These questions helps identify those. Sometimes unnecessary work is caused by applications malfunctioning.

Analysis of the workload also helps separate problems of load from problems of architecture, by identifying the former.

The specific tools and metrics for performing workload characterization depend on the target. Some applications record detailed logs of client activity, which can be the source for statistical analysis. They may also already provide daily or monthly reports of client usage, which can be mined for details.  Use Elastic stack for log aggregation if the data is not confidential or SHR.

## Latency Methods

Latency analysis examines the time taken to complete an operation, then breaks it into smaller components, continuing to subdivide the components with the highest latency so that the root cause can be identified and quantified. Similarly to drill-down analysis, latency analysis may drill down through layers of the software stack to find the origin of latency issues.

Analysis can begin with the workload applied, examining how that workload was processed in the application, then drilling down into the operating system libraries, system calls, the kernel, and device drivers.

For example, analysis of MySQL query latency could involve answering the following questions (example answers are given here):
1. Is there a query latency issue? (yes)
2. Is the query time largely spent on-CPU or waiting off-CPU? (off-CPU)
3. What is the off-CPU time spent waiting for? (file system I/O)
4. Is the file system I/O time due to disk I/O or lock contention? (disk I/O)
5. Is the disk I/O time likely due to random seeks or data transfer time? (transfer time)

For this example, each step of the process posed a question that divided the latency into two parts, and then proceeded to analyze the larger part: a binary search of latency, if you will.

![Latency Method](/Images/PA-Latency-Method.jpg)

Latency analysis of database queries is the target of method R. Method R is a performance analysis methodology developed for Oracle databases that focuses on finding the origin of latency, based on Oracle trace events. It is described as “a response time-based performance improvement method that yields maximum economic value to your business” and focuses on identifying and quantifying where time is spent during queries. While this is used for the study of databases, its approach could be applied to any system and is worth mentioning here as an avenue of possible study.

## Event Tracing
Systems operate by processing discrete events. These include CPU instructions, disk I/O and other disk commands, network packets, system calls, library calls, application transactions, database queries, and so on. Performance analysis usually studies summaries of these events, such as operations per second, bytes per second, or average latency. Sometimes important detail is lost in the summary, and the events are best understood when inspected individually.

#### Network layer

**Network troubleshooting often requires packet-by-packet inspection, with tools such as tcpdump**. This example summarizes packets as single lines of text.

```bash
# tcpdump -ni eth4 -ttt
tcpdump: verbose output suppressed, use -v or -vv for full protocol decode
listening on eth4, link-type EN10MB (Ethernet), capture size 65535 bytes
00:00:00.000000 IP xx.yy.zzz.x.22 > xx.yy.zzz.x.33986: Flags [P.], seq 1182098726:1182098918, ack 4234203806, win 132, options [nop,nop,TS val 1751498743 ecr 1751639660], length 192
00:00:00.000392 IP xx.yy.zzz.x.33986 > xx.yy.zzz.x.22: Flags [.], ack 192, win 501, options [nop,nop,TS val 1751639684 ecr 1751498743], length 0
00:00:00.009561 IP xx.yy.zzz.x.22 > xx.yy.zzz.x.33986: Flags [P.], seq 192:560, ack 1, win 132, options [nop,nop,TS val 1751498744 ecr 1751639684], length 368
00:00:00.000351 IP xx.yy.zzz.x.33986 > xx.yy.zzz.x.22: Flags [.], ack 560, win 501, options [nop,nop,TS val 1751639685 ecr 1751498744], length 0
00:00:00.010489 IP xx.yy.zzz.x.22 > xx.yy.zzz.x.33986: Flags [P.], seq 560:896, ack 1, win 132, options [nop,nop,TS val 1751498745 ecr 1751639685], length 336
00:00:00.000369 IP xx.yy.zzz.x.33986 > xx.yy.zzz.x.22: Flags [.], ack 896, win 501, options [nop,nop,TS val 1751639686 ecr 1751498745], length 0
```
#### Storage layer
Storage device I/O at the block device layer can be traced using iosnoop(1M) (DTrace-based: Disks)

```bash
# ./iosnoop -Dots
STIME(us)    TIME(us)      DELTA  DTIME  UID   PID D    BLOCK   SIZE  COMM ...
722594048435 722594048553  117    130      0   485 W 95742054   8192 zpool-...
722594048879 722594048983  104    109      0   485 W 95742106   8192 zpool-...
722594049335 722594049552  217    229      0   485 W 95742154   8192 zpool-...
722594049900 722594050029  128    137      0   485 W 95742178   8192 zpool-...
722594050336 722594050457  121    127      0   485 W 95742202   8192 zpool-...
722594050760 722594050864  103    110      0   485 W 95742226   8192 zpool-...
722594051190 722594051262  72     80       0   485 W 95742250   8192 zpool-...
722594051613 722594051678  65     72       0   485 W 95742318   8192 zpool-...
722594051977 722594052067  90     97       0   485 W 95742342   8192 zpool-...
722594052417 722594052515  98     105      0   485 W 95742366   8192 zpool-...
722594052840 722594052902  62     68       0   485 W 95742422   8192 zpool-...
722594053220 722594053290  69     77       0   485 W 95742446   8192 zpool-...
```
Multiple timestamps are printed here, including the start time (STIME), end time (TIME), delta time between request and completion (DELTA), and estimated time to service this I/O (DTIME).

#### System call layer

The system call layer is another common location for tracing, with tools including strace(1) on Linux. These tools also have options to print timestamps.

When performing event tracing, look for the following information:

- Input: all attributes of an event request: type, direction, size, and so on
- Times: start time, end time, latency (difference)
- Result: error status, result of event (size)

[^Top](#performance-analysis-methodologies) |  [^ Metrics](/Performance-Metric.md)  |  [^Tools](/Performance_Analysis_Tools.md)
