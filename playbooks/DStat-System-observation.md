### DTrace - Dynamic tracing

For performance observation, you may be looking for that one tool that can give your a good amount of the information provided by above tools, even more, a single and powerful tool that has additional features and capabilities, then look no further than dstat.

DTrace provides both static and dynamic tracing of user- and kernel-level software and can provide data in real time.

dstat is a powerful, flexible and versatile tool for generating Linux system resource statistics, that is a replacement for all the tools mentioned above. It comes with extra features, counters and it is highly extensible, users with Python knowledge can build their own plugins.

### Features of dstat:
- Joins information from vmstat, netstat, iostat, ifstat and mpstat tools
- Displays statistics simultaneously
- Orders counters and highly-extensible
- Supports summarizing of grouped block/network devices
- Displays interrupts per device
- Works on accurate timeframes, no timeshifts when a system is stressed
- Supports colored output, it indicates different units in different colors
- Shows exact units and limits conversion mistakes as much as possible
- Supports exporting of CSV output to Gnumeric and Excel documents

### Few useful options
- `dstat --list`
- `dstat -c --top-cpu -d --top-bio --top-latency`
```bash
----total-cpu-usage---- -most-expensive-  -dsk/total- ----most-expensive--- --highest-total--
usr sys idl wai hiq siq|  cpu process   | read  writ|  block i/o process   | latency process
5    0   94   0   0   0|firefox      3.6| 148k   81k|init [5]     98k   50B|pdflush        21
2    1   98   0   0   0|wnck-applet  0.5|   0     0 |                      |at-spi-regist   5
2    1   98   0   0   0|firefox      0.5|   0     0 |                      |Xorg            1
1    2   97   0   0   1|                |   0     0 |                      |Xorg            1
1    1   98   0   0   0|                |   0     0 |                      |ksoftirqd/1    10
1    1   97   0   0   0|firefox      0.5|   0     0 |                      |ksoftirqd/0     5
2    1   97   0   0   0|firefox      0.5|   0     0 |firefox       0    28k|ksoftirqd/0     5
2    1   97   0   0   0|firefox      0.5|   0     0 |                      |Xorg            1
1    1   97   0   0   0|firefox      0.5|   0     0 |                      |ksoftirqd/0     6
2    1   98   0   0   0|firefox      0.5|   0     0 |                      |ksoftirqd/0     6
1    2   98   0   0   0|                |   0     0 |                      |ksoftirqd/1     8
2    1   98   0   0   0|iwlagn       0.5|   0    72k|kjournald     0    32k|ksoftirqd/1    12
1    1   97   0   0   0|                |   0     0 |                      |iwlagn/0        1
1    1   98   0   0   0|firefox      0.5|   0     0 |                      |ksoftirqd/1     8
```

- `dstat --vmstat`
- `dstat --list`
- `dstat -c --top-cpu -dn --top-mem`
  - Additionally, you can also store the output of dstat in a .csv file for analysis at a latter time by enabling the  --output option as in the example below
  - `$ dstat --time --cpu --mem --load --output report.csv 1 5`
- `dstat -tcmsn -N eth0`
  - Show information about cpu, memory, eth0 activity and system resources related to time
- `dstat -cdl -D sda1`
  - Show information about cpu, disk (sda1) utilization and system load
- `dstat -tn -N eth0 --tcp 5`
  - Using delay for statistics
  - Let say you want to display statistic every 5 seconds about network and tcp activity
- `$ dstat -tn eth0 --tcp 2 10`
  - we want to limit the number of stats updates to 10 with delay every 2 second
- `dtrace -n 'syscall:::entry /execname == "postgres"/ { @[probefunc] = count(); }'`
  - one-liner counts syscalls (using an aggregation) for processes named "postgres" (PostgreSQL database)
- `dtrace -n 'syscall::read:entry /execname == "postgres"/ {
    @[ustack()] = count(); }'`
  - this one-liner traces PostgreSQL read() syscalls, gathers the user-level stack trace, and aggregates them:

[More dtrace one liner command to analyze the system in kernel level](https://wiki.freebsd.org/DTrace/One-Liners)  
[Dtrace tutorials](https://wiki.freebsd.org/DTrace/Tutorial?highlight=%28%5CbCategoryDTrace%5Cb%29)

[^ Top](/DStat-SystemAdmin.md#dtrace---dynamic-tracing)  |   [^ Metrics](/Performance-Metric.md)  |  [^Tools](/Performance_Analysis_Tools.md)
