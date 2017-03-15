# Latency assessment

## Latency assessment in every layer of the application and system.
The Latency can be observed from System level and Process level
- Application latency
- Database Latency
- CPU Latency
    - Schedule Latency
    - Run Queue Latency
- File System Latency
- Disk I/O Latency
  - Write Latency (Buffer size)
  - Read Latency (I/O size)
- Network latency

Watch out this area ..

### Process level Latency observation
1. Process level latency can be observed from /proc/<Process ID>/schedstat

2. The scheduler latency statistic is sourced from /pro/schedstats
CONFIG_TASK_DELAY_ACCT option track time per task in the following states:
- **Scheduler latency**: waiting for a turn on-CPU
- **Block I/O**: waiting for a block I/O to complete
- **Swapping**: waiting for paging (memory pressure)
- **Memory reclaim**: waiting for the memory reclaim routine

The scheduler latency statistic is sourced from schedstats (in /proc).

These statistics can be read by user-level tools using taskstats, which is a netlink-based interface for fetching per-task and process statistics. The kernel source Documentation/accounting directory has both the documentation, delay-accounting.txt, and an example consumer, getdelays.c:

```bash
$ ./getdelays -dp 17451
print delayacct stats ON
PID    17451
CPU             count     real total  virtual total    delay total  delay average
                  386     3452475144    31387115236     1253300657          3.247ms
IO              count    delay total  delay average
                  302     1535758266              5ms
SWAP            count    delay total  delay average
                    0              0              0ms
RECLAIM         count    delay total  delay average
                    0              0              0ms
```
It was taken from a heavily CPU-loaded system, and the process inspected was suffering scheduler latency.

### File System Latency

File system latency is the primary metric of file system performance, measured as the time from a logical file system request to its completion. It is **inclusive of time spent in the file system, kernel disk I/O subsystem, and waiting on disk devices—the physical I/O**. Application threads often block during an application request to wait for file system requests to complete, for which file system latency directly and proportionally affects application performance.

Cases where applications may not be directly affected include the use of non-blocking I/O, or when I/O is issued from an asynchronous thread.

For applications, this process is transparent: their logical I/O latency becomes much lower, as it can be served from main memory rather than the much slower disk devices.


????

`$ perf sched`
