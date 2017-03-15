# Dynamic system observability

## Introduction
For performance observation, you may be looking for that one tool that can give your a good amount of the information provided by above tools, even more, a single and powerful tool that has additional features and capabilities, then look no further than dstat.

**dstat** is a powerful, flexible and versatile tool for generating Linux system resource statistics, that is a replacement for all the tools mentioned above. It comes with extra features, counters and it is highly extensible, users with Python knowledge can build their own plugins.

**DTrace** provides both static and dynamic tracing of user- and kernel-level software and can provide data in real time. It can be used in Solaris, Mac OS X. Linux version is being developed. In Linux, DTrace capabilities are available in Systemtap, perf and eBPF with some short coming. But understanding dtrace capability will help the performance engineer to look different approaches to observe performance.

A key difference of DTrace from other tracing frameworks (e.g., syscall tracing) is that **DTrace is designed to be production-safe, with minimized performance overhead.**

**Systemtap**
SystemTap also provides static and dynamic tracing for user- and kernel-level code and was conceived for Linux. SystemTap sources other kernel frameworks for tracing: **tracepoints for static probes, kprobes for dynamic probes, and uprobes for user-level probes**. These sources are also used by other tools (perf, LTTng).

**Perf**: Linux Performance Events (LPE), perf for short, has been evolving to support a wide range of performance observability activities. While it doesn’t currently have the real-time programmatic capabilities of DTrace or SystemTap, it can perform static and dynamic tracing (based on tracepoints, kprobes, and uprobes), as well as profiling. It can also inspect stack traces, local variables, and data types.

## Dstat
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

- Show information about cpu, memory, eth0 activity and system resources related to time
- `dstat -tcmsn -N eth0`

- Show information about cpu, disk (sda1) utilization and system load
- `dstat -cdl -D sda1`

- Using delay for statistics
  - Let say you want to display statistic every 5 seconds about network and tcp activity
  - `dstat -tn -N eth0 --tcp 5`
  - we want to limit the number of stats updates to 10 with delay every 2 second
  - `$ dstat -tn eth0 --tcp 2 10`

## dtrace
DTrace currently available for Solaris, Mac OS X as part of their base version. [Oracle release Linux version](git://oss.oracle.com/git/dtrace-linux-kernel.git) as OSS under CDDL license. 

DTrace is an observability framework that includes a programming language and a tool. This section summarizes DTrace basics, including dynamic and static tracing, probes, providers, D, actions, variables,s one-liners, and scripting.

DTrace can observe all user- and kernel-level code via instrumentation points called probes. When probes are hit, arbitrary actions may be performed in its D language. Actions can include counting events, recording timestamps, performing calculations, printing values, and summarizing data. These dynamic tracing actions can be performed in real time, while tracing is still enabled.

### probe
DTrace probes are named with a four-tuple:
`provider:module:function:name`

The provider is the collection of related probes, similar to a software library. The module and function are dynamically generated and specify the code location of the probe. The name is the name of the probe itself.

When specifying these, wildcards (“\*”) may be used. Leaving a field blank (“::”) is equivalent to a wildcard (“:\*:”). Blank left fields may also be dropped from the probe specification (e.g., “:::BEGIN” == “BEGIN”).
`io:::start`
is the start probe from the io provider. The module and function fields are left blank, so these will match all locations of the start probe.

### providers
The DTrace providers available depend on your DTrace and operating system version. They may include
- **syscall** system call trap table
- **vminfo** virtual memory statistics
- **sysinfo** system statistics
- **profile** sampling at arbitrary rates
- **sched** kernel scheduling events
- **proc** process-level events: create, exec, exit
- **io** block device interface tracing (disk I/O)
- **pid** user-level dynamic tracing
- **tcp** TCP protocol events: connections, send and receive
- **ip** IP protocol events: send and receive
- **fbt** kernel-level dynamic tracing

There are many additional providers for higher-level languages: **Java, JavaScript, Node.js, Perl, Python, Ruby**, Tcl, and others.

### D language
The D language is awk-like and can be used in one-liners or scripts (the same as awk)
` probe_description /predicate/ { action }`

The action is a series of optional semicolon-delimited statements that are executed when the probe fires. The predicate is an optional filtering expression.
`proc:::exec-success /execname == "httpd"/ { trace(pid); }`

traces the exec-success probe from the proc provider and performs the printing action trace(pid) if the process name is equal to "httpd". The exec-success probe is commonly used to trace the creation of new processes and instruments a successful exec() system call. The current process name is retrieved using the built-in variable execname, and the current process ID via pid.

### Built-in variables
Built-in variables can be used in calculations and predicates and can be printed using actions such as trace() and printf(). Commonly used built-ins are

Variables|Description
---- | ----
execname | on-CPU process name
uid| on-CPU user ID
pid| on-CPU process ID
timestamp| current time, nanoseconds since boot
vtimestamp| time thread was on-CPU, nanoseconds
arg0..N|probe arguments (unit64_t)
args[0]..[N]|probe arguments(typed)
curthread|pointer to current thread kernal structure
probefunc |function component of probe Description
probename | name component of probe Description
curpsinfo| current process information

### Actions
Commonly used actions include those listed

Action|Description
----|----
trace (arg)|print arg
printf(format,arg,...)|print formatted string
stringof(addr)| return a string from a kernel address
coptinstr(addr)|return a string from a use-space address
stack(count)|print kernel-level stack trace
ustack(count)|print user-level stack trace
func(pc)| return a kernal function name, from kernel program counter (PC)
exit (status)|exit DTrace and return status
trunc (@agg, count)| truncate the aggregation, either fully or to the number of keys specified (count)
clear(@agg)|delete values from an aggregation (keep keys)
printa(format, @agg)|print aggregation, formatted

### Variable types
the types of variables, listed in order of usage preference (aggregations are chosen first, then low to high overhead).

Type|Prefix|Scope|Overhead|Multi-CPU Safe|Example Assignment
----|----|----|----|-----|----
Aggregation| @|global|Low|yes|@x=count();
Aggregation| @[]|global|low|yes|@x[pid]=count();
Clause-local | this->| clause instance|Very low| yes| this->x=1;
Thread-local | self-> | thread|medium| yes|self->x=1;
Scalar| none | global | low-medium|no|x=1;
Associative array|none|global|medium-high|no|x[y]=1;


An aggregation is a special variable type that can be tallied per CPU and combined later for passing to user-land. These have the lowest overhead and are used for summarizing data in different ways.

Aggregation Action|Description
----|----
Count, Sum, Min, Max| Self explanatory
**quantize(value)** | record value as a power-of-two histogram
**lquantize(Value,min,max,step)** |record values as a linear histogram, with minimum, maximum, and step provided
llquantize(value,factor,min_,magnitude,max_magnitude,steps)|record value as a hybrid log/linear historgram

Example of an aggregation and a historgram action, quantize(), the following shows and returned sizes for the read () syscall:

```bash
# dtrace -n 'syscall::read:return { @["rval (bytes)"] = quantize(arg0); }'
dtrace: description 'syscall::read:return ' matched 1 probe
^C
  rval (bytes)
           value  ------------- Distribution ------------- count
              -1 |                                         0
               0 |@@@@@@@@@@@@@@                           447
               1 |@@@                                      100
               2 |                                         5
               4 |                                         0
               8 |                                         2
              16 |                                         2
              32 |@@                                       53
              64 |                                         1
             128 |                                         0
             256 |                                         0
             512 |                                         4
            1024 |@                                        19
            2048 |                                         10
            4096 |@                                        34
            8192 |@@@@                                     130
           16384 |@@@@@@                                   170
           32768 |@@@@                                     125
           65536 |@@@@                                     114
          131072 |                                         5
          262144 |                                         5
          524288 |                                         0
```
In this case, the most frequently returned size was zero bytes,

#### One-Liners

- `dtrace -ln 'syscall:::'`
  - list all the syscall
- `dtrace -n 'syscall:::entry /execname == "postgres"/ { @[probefunc] = count(); }'`
  - one-liner counts syscalls (using an aggregation) for processes named "postgres" (PostgreSQL database)
- `dtrace -n 'syscall::read:entry /execname == "postgres"/ {
    @[ustack()] = count(); }'`
  - this one-liner traces PostgreSQL read() syscalls, gathers the user-level stack trace, and aggregates them:
- `dtrace -n 'syscall::open:entry { printf("%s %s", execname, copyinstr(arg0)); }'`
  - Trace open() system calls, printing the process name and file path name:
- `dtrace -n 'sysinfo:::xcalls { @[execname] = count(); }'`
  - Summarize CPU cross calls by process name:

[More dtrace one liner command to analyze the system in kernel level](https://wiki.freebsd.org/DTrace/One-Liners)  
[Dtrace tutorials](https://wiki.freebsd.org/DTrace/Tutorial?highlight=%28%5CbCategoryDTrace%5Cb%29)

#### scripting
DTrace statements can be saved to a file for execution, allowing much longer DTrace programs to be written.

For example, the bitesize.d **script shows requested disk I/O sizes by process name:**

```bash
#!/usr/sbin/dtrace -s

#pragma D option quiet

dtrace:::BEGIN
{
        printf("Tracing... Hit Ctrl-C to end.\n");
}

io:::start
{
        this->size = args[0]->b_bcount;
        @Size[pid, curpsinfo->pr_psargs] = quantize(this->size);
}

dtrace:::END
{
        printf("\n%8s  %s\n", "PID", "CMD");
        printa("%8d  %S\n%@d\n", @Size);
}
```
The #pragma line sets quiet mode, which suppresses the default DTrace output.

Example Out:
```bash
# ./bitesize.d
Tracing... Hit Ctrl-C to end.
^C

     PID  CMD
    3424  tar cf /dev/null .\0

           value  ------------- Distribution ------------- count
             512 |                                         0
            1024 |@@@                                      39
            2048 |@@@@@@                                   71
            4096 |@@@@@@@@@                                111
            8192 |@@@@@@@@@@@@@@@@@@@@@                    259
           16384 |                                         6
           32768 |@                                        8
           65536 |                                         0
```

While tracing, most of the disk I/O was requested by the tar command, with sizes shown above.

bitesize.d is from a collection of DTrace scripts called the DTraceToolkit, which can be found online.

[The Dynamic Tracing Guide, originally by Sun Microsystem](http://dtrace.org/guide)

[These scripts are available online](www.dtracebook.com)

### profiling
Profiling can also be based on untimed hardware events, such as CPU hardware cache misses or bus activity. It can also show which code paths are responsible. DTrace does programmatic profiling, timer-based using its profile provider, and hardware-event-based using its cpc provider.


[^ Top](/DStat-SystemAdmin.md#dtrace---dynamic-tracing)  |   [^ Metrics](/Performance-Metric.md)  |  [^Tools](/Performance_Analysis_Tools.md)
