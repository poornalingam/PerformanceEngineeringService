# How and What will be analyzed
- [Performance Metrics for all software and physical stack](/Performance-Metric.md)
- [Tools for performance analysis](/Performance_Analysis_Tools.md)
- [Application level performance analysis](#application-level-performance-analysis)
- [Database level performance analysis](#database-level-performance-analysis)
- [K8S and Container level performance analysis](/K8S-Container-performance-monitoring.md)
- [MicroServices performance analysis](/MicroService_performance_assessment.md)
- [Infrastructure level performance analysis](#infrastructure-level-performance-analysis)
- [Techniques for performance analysis](/Method-PerformanceAnalysis.md)
- Scripting language specific performance analysis

# Performance Metrics for all software and physical stack

## Performance Tuning efforts

**Performance tuning is most effective when done closest to where the work is performed**. For workloads driven by applications, this means within the application itself. Table shows an example software stack, with tuning possibilities.

Layer | Tuning Targets
:-----| :--------------
**Application** | database queries performed
**Database** | database table layout, indexes, buffering
**System Calls** | memory-mapped or read/write, sync or async I/O flags
**File System** | record size, cache size, file system tunable
**Storage** | RAID level, number and type of disks, storage tunable

By tuning at the application level, it may be possible to eliminate or reduce database queries and improve performance by a large factor (e.g., 20x). Tuning down to the storage device level may eliminate or improve storage I/O, but a tax has already been paid executing higher-level OS stack code, so this may improve resulting application performance by only percentages (e.g., 20%).

Remember that **operating system performance analysis can also identify application-level issues, not just OS-level issues**, in some cases more easily than from the application alone.

## Application level performance analysis
Analysis level | Metrics | Suitable Tools
:--- | :---:| :---
Client Application – HTML |TBD| Log analysis
Client Application - AJAX |TBD| Log analysis, HttpWatch
Application | GC type, GC Page type and Size | Manual approach to choose the right GC to meet your requirements, GC details can be sent to Log
Application - JVM | Time taken for Minor and Major| Garbage collection; Log analysis
Heap memory | Long living objects, Objects without reference; Large objects; Small objects with larger number of reference; Weak links;| JMC;JFR; IBM heap analyzer;
Database call| number of database call made per request and its response time| APM tools, Log analysis,
Database call | Network Latency if db chatty application| DCRUM, dtrace

### Application performance analysis playbooks
  - TBD

## Database level performance analysis
Analysis level | Metrics | Suitable Tools
:--------- | :----:| :-----
ORA DB Level | Avg Active user Session, TPS, IOs/txn, Avg wait, **Logical and Physical I/O**, Foreground wait events, buffer pool, flash cash, Table & Row locks (TM & TX - contention), Long running Query and Larger no of calls, Leakages | ADDM, ASH, AWR (<=1hr window), OEM, Spotlight, **Debug**: SQL tracing/tkprof, Statpack, **Work with your DBA for detailed study**
DB level: Latency | File system cache hit rate (<95%), IOPS, throughput, average disk I/O latency, and the read/write ratio. **Debug**: disk I/O level tracing | Linux observation tools (mpstat, iostat -x 1 (>10ms), dstat)
DB level - Scheduler |Schedule, type of backup, backup level, Bkp elapse time | log analysis: Backup logs  
SGA and PGA projection  |TBD|
Cassandra| TBD|

### Database analysis Playbooks
  - [Database - disk read slowness](/playbooks/db-read-slow-disk.md)
  - More to come

## Infrastructure level performance analysis

Depends on performance analysis and depth of analysis choose the right metrics to collect live analysis and historical analysis. Use the below play books to get yourself familiar.
- [System performance issue quick guide](/playbooks/system-performance-issue-quickguide.md)
- System performance issue analysis guide
- System performance issue debug

![Linux perf tools](/Images/linux_perf_tools_full.jpg)

Analysis level |      Metrics        | Suitable Tools
:------------ | :--------------------| :---------------
Info | CPU – load average, system, user, wait, | Sensu, Metricbeat, [Linux observation tools](/Images/linux_perf_tools_full.jpg), nmon
Info | Memory – used, cached, Paging, Scan Rate, Swap out, | Sensu, Metricbeat, [Linux observation tools](/Images/linux_perf_tools_full.jpg), nmon
Info | Network – Packet receive, sent and dropped | Sensu, Metricbeat, [Linux observation tools](/Images/linux_perf_tools_full.jpg), nmon
Info | Process| Sensu, Metricbeat, [Linux observation tools](/Images/linux_perf_tools_full.jpg), nmon
Info | Disk I/O – tps, Read/Write trans per sec, block read and write per sec, | Sensu, Metricbeat, [Linux observation tools](/Images/linux_perf_tools_full.jpg), [sar](/Images/linux_observability_sar.jpg), nmon
Analysis | CPU - Content Switch, individual CPU statistics, fork, exec| [Linux observation tools](/Images/linux_perf_tools_full.jpg), nmon
Analysis | Latency| Sensu, Metricbeat, [Linux observation tools](/Images/linux_perf_tools_full.jpg), nmon
Analysis | Throughput | Sensu, Metricbeat, [Linux observation tools](/Images/linux_perf_tools_full.jpg), nmon
Analysis | Queue length; Run Queue Length| Sensu, Metricbeat, [Linux observation tools](/Images/linux_perf_tools_full.jpg), nmon
Analysis | Memory - VM efficiency, Page Steal, Page fault – Major,  cache hit ration, cache miss rate| Sensu, Metricbeat, [Linux observation tools](/Images/linux_perf_tools_full.jpg), nmon
Analysis | Network – send-Q, Recv-Q |
Analysis | Open, read, close file | Sensu, Metricbeat, [Linux observation tools](/Images/linux_perf_tools_full.jpg), nmon
Analysis | Thread level resource usages | [Linux observation tools](/Images/linux_perf_tools_full.jpg)
Analysis | Process – fork, exec | [Linux observation tools](/Images/linux_perf_tools_full.jpg)
Debug | CPU – Cycle, Instructions, PC, Cache –misses, LLC, cache hit ration, affinity | **ftrace**; Perf stat/record/lock; DTrace, SystemTap; [perf-tool](https://github.com/brendangregg/perf-tools); **eBPF**
Debug | Desk - ftrace |  **ftrace**; perf; Dtrace; [perf-tool](https://github.com/brendangregg/perf-tools);**eBPF**
Debug| Network -  tcpdump | [Linux observation tools](/Images/linux_perf_tools_full.jpg)
Debug | IO - IO traces dumped | Blktrace;[perf-tool](https://github.com/brendangregg/perf-tools);**eBPF**

### Infrastructure performance analysis playbooks
 - [System performance issue quick guide](/playbooks/system-performance-issue-quickguide.md)
 - [Linux performance tools tutorial](/observability-tools/Linux-Performance-Observation-Tools.md)
 - [Dynamic system observation](/observability-tools/Dynamic-System-observation.md)
 - Extended BPF
 - Dtrace
 - Vector
 - Flame Graph
 - Many more to come

[Back to Top](#how-and-what-will-be-analyzed)
