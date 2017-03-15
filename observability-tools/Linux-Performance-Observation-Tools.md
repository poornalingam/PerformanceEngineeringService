## Linux Performance Observation tools                                

Layers | Performance analysis tools
-----:| :----------:
One-liners|  ######  Many  #########
Frond-end tools| ######   [perf](#perf); trace-cmd; [perf-tools](#opensource-tools)  ######
Tracing frameworks| ######### [ftrace](#ftrace); perf_events; [eBPF](#extended-bpf),..  #########
Back-end instrumentation| ############   tracepoints; kprobes; uprobes  ############

### ftrace
ftrace is introduced in linux from 2.6+ (2005). Its not effectively utilized so far because of difficult to understand. There are few front end tools which wraps ftrace. Example: trace-cmd and [perf-tools](#opensource-tools). Use ftrace when you don't have any other option where you don't have any other tracing tools are available.

**Note**: Netflex is heavy using ftrace and perf tools for performance analysis.

From the tracing perspective its well matured tool.

### perf
The perf tool can be used to collect profiles on per-thread, per-process and per-cpu basis. Measure what's going on inside a CPU. Have a look this [perf one-liner](http://www.brendangregg.com/perf.html) for examples.

```bash
# perf stat gzip file1
 Performance counter stats for 'gzip file1':
       1920.159821 task-clock                #    0.991 CPUs utilized
                13 context-switches          #    0.007 K/sec
                 0 CPU-migrations            #    0.000 K/sec
               258 page-faults               #    0.134 K/sec
     5,649,595,479 cycles                    #    2.942 GHz                     [83.43%]
     1,808,339,931 stalled-cycles-frontend   #   32.01% frontend cycles idle    [83.54%]
     1,171,884,577 stalled-cycles-backend    #   20.74% backend  cycles idle    [66.77%]
     8,625,207,199 instructions              #    1.53  insns per cycle
                                             #    0.21  stalled cycles per insn [83.51%]
     1,488,797,176 branches                  #  775.351 M/sec                   [82.58%]
        53,395,139 branch-misses             #    3.59% of all branches         [83.78%]

       1.936842598 seconds time elapsed
```

### dtrace - Dynamic tracing
Dynamic tracing allows all software to be instrumented, live and in production. It is a technique of taking in-memory CPU instructions and dynamically building instrumentation upon them. It is so different from traditional observation that it can be difficult, at first, to grasp its role. It is available Linux and Mac OS X.

Prior to dtrace, system tracing was commonly performed using static probes. Their visibility was limited, and their usage was often time-consuming, requiring a cycle of configuration, tracing, dumping data, and then analysis. DTrace provides both static and dynamic tracing of user and kernel-level software and can provide data in real time.The following simple example traces process execution during an ssh login. Tracing is system-wide (not associated with a particular process ID):

```bash
# dtrace -n 'exec-success { printf("%d %s", timestamp, curpsinfo->pr_psargs); }'
dtrace: description 'exec-success ' matched 1 probe
CPU     ID               FUNCTION:NAME
  2   1425    exec_common:exec-success 732006240859060 sh -c /usr/bin/locale -a
  2   1425    exec_common:exec-success 732006246584043 /usr/bin/locale -a
  5   1425    exec_common:exec-success 732006197695333 sh -c /usr/bin/locale -a
  5   1425    exec_common:exec-success 732006202832470 /usr/bin/locale -a
  0   1425    exec_common:exec-success 732007379191163 uname -r
  0   1425    exec_common:exec-success 732007449358980 sed -ne /^# START exclude/,/^#
FINISH exclude/p /etc/bash/bash_completion
  1   1425    exec_common:exec-success 732007353365711 -bash
  1   1425    exec_common:exec-success 732007358427035 /usr/sbin/quota
  2   1425    exec_common:exec-success 732007368823865 /bin/mail -E
 12   1425    exec_common:exec-success 732007374821450 uname -s
 15   1425    exec_common:exec-success 732007365906770 /bin/cat -s /etc/motd
 ```

In this example, DTrace was instructed to print timestamps (nanoseconds) with process names and arguments. Much more sophisticated scripts can be written in the D language, allowing us to create and calculate custom measures of latency.

[more](/playbooks/DStat-SystemAdmin.md)

### SAR - System Activity Reporter

SAR can be used analyze traditional resource monitoring during runtime and historically. Very useful for historical server resource utilization analysis.

![SAR](/Images/linux_observability_sar.jpg)

### extended BPF
eBPF is introduced in 2014. It is pretty fast in kernel aggregations and programs. It can generate in-kernel latency heat-map. Yet to explore BPF capabilities. **TBD**. Read more [here](http://www.brendangregg.com/blog/2015-05-15/ebpf-one-small-step.html) and [here](http://www.brendangregg.com/ebpf.html)

Here's an example of tracing and showing block (disk) I/O as a latency heat map:

![Heat-map](/Images/tracex3_heatmap_01.png)

The passage of time is on the y-axis (going downwards), and latency is on the x-axis. The color depth shows how many I/O fell into a time and latency range: darker for more. This particular example shows that most I/O took around 9 ms, but with a wide distribution.

Lots of companies are using BPF in reason time. Some of those are github, facebook and netflex.

### Opensource tools

Linux 2.8+ kernel has 1200 tracepoints in kernel. Use these tracepoints to analyze blocks, waits, latency in details. Will list most of the useful tracepoints in sometime. Both ftrace and perf are core Linux tracing tools, included in the kernel source. Your system probably has ftrace already, and perf is often just a package add.  Custom tools can be built to get more clarity in analysis.

One among these open-source tools is [perf-tools](https://github.com/brendangregg/perf-tools). It has very good features to analyze latency. It build on Linux 3.2+ Kernels. Would recommend you to have a look this tool.

The two tools are most interesting tools
[iolatency example](https://github.com/brendangregg/perf-tools/blob/master/examples/iolatency_example.txt);
[iosnoop example](https://github.com/brendangregg/perf-tools/blob/master/examples/iosnoop_example.txt)

```bash
# ./iosnoop -ts
Tracing block I/O. Ctrl-C to end.
STARTs         ENDs           COMM             PID    TYPE DEV      BLOCK        BYTES     LATms
5982800.302061 5982800.302679 supervise        1809   W    202,1    17039600     4096       0.62
5982800.302423 5982800.302842 supervise        1809   W    202,1    17039608     4096       0.42
5982800.304962 5982800.305446 supervise        1801   W    202,1    17039616     4096       0.48
5982800.305250 5982800.305676 supervise        1801   W    202,1    17039624     4096       0.43
5982800.308849 5982800.309452 supervise        1810   W    202,1    12862464     4096       0.60
5982800.308856 5982800.309470 supervise        1806   W    202,1    17039632     4096       0.61
5982800.309206 5982800.309740 supervise        1806   W    202,1    17039640     4096       0.53
5982800.309211 5982800.309805 supervise        1810   W    202,1    12862472     4096       0.59
5982800.309332 5982800.309953 supervise        1812   W    202,1    17039648     4096       0.62
5982800.309676 5982800.310283 supervise        1812   W    202,1    17039656     4096       0.61
[...]
```
Measuring block device I/O latency from queue insert to completion:

```bash
# ./iolatency -Q
Tracing block I/O. Output every 1 seconds. Ctrl-C to end.

  >=(ms) .. <(ms)   : I/O      |Distribution                          |
       0 -> 1       : 1913     |######################################|
       1 -> 2       : 438      |#########                             |
       2 -> 4       : 100      |##                                    |
       4 -> 8       : 145      |###                                   |
       8 -> 16      : 43       |#                                     |
      16 -> 32      : 43       |#                                     |
      32 -> 64      : 1        |#                                     |

[...]
```

### Visual tools

#### Google Heapster and cAdvisor
In a cloud cluster, application performance can be examined at many different levels: containers, pods, services, and whole clusters. It gives users deep insights into how their applications are performing and where possible application bottlenecks may be found. In comes Heapster, a project meant to provide a base resource monitoring platform on Kubernetes. It uses cAdvisor to collect metrics. Its a open-source tool as well. Metrics are stored in InfluxDB and exposed through REST API.

[More](K8-Container-performance-monitoring.md)

Life cycle monitoring is only restricted to container's poststart and prestop. Don't see cAdvisor as kernel level performance analysis tool as of now.

#### Kernelshark - for ftrace

KernelShark is a front end reader of trace-cmd output. Kernelshark can read this file and produce a graph view,list view and simple and Advance filtering of its data.

trace-cmd is a binary tool to read ftrace's buffers and records into trace.dat file.

![KernelShark Screenshot](/Images/kernelShark-plot-task-result.jpg)

#### Trace Compass - to visualize LTTng time service trace data

Eclipse Trace Compass is an open source java application for viewing and analyzing any type of logs or traces. Its goal is to provide views, graphs, metrics, and more to help extract useful information from traces, in a way that is more user-friendly and informative than huge text dumps. It supports Linux, Windows and Mac OS X.

###### Key features
Offline analysis of complex issues
- Real-time deadline investigation
- Latency analysis
- Log correlation with operating system traces
- Network packet correlation accross layers
- Identification of relevant information in large amounts of trace data
- Causes of high processor usage and memory leaks
- Correlation of hardware and software components execution traces
- Symbol name resolution using debug information


1. The Kernel Analysis displays the states of processes and resources over time, using information from Linux kernel traces.
![TraceCompass](/Images/TraceCompass-KernelAnalysis.png)

2. If you can define trace events representing function entries and exits, you can display the call stack of your application over time. [Sample Screenshot](/Images/TraceCompass-callstack.png)
3. Using LTTng-UST's C standard library wrapper, all calls to malloc() and free() can be instrumented without recompiling the application. This allows plotting the memory usage over time. [Sample Screenshot](/Images/TraceCompass-memory.png)
4. The base framework can be extended to add support for new trace types. Support for libpcap traces (the format used by Wireshark) was added this way. [Sample Screenshot](/Images/TraceCompass-pcap.png)


##### LTTng - Linux Trace Toolkit, next generation
One of the main features of Trace Compass is the LTTng integration. [LTTng](http://lttng.org/) is an open source tracing framework for Linux.  LTTng (Linux Trace Toolkit, next generation) is a highly efficient tracing tool for Linux that can be used to track down kernel and application performance issues as well as troubleshoot problems involving multiple concurrent processes and threads. It consists of a set of kernel modules, daemons - to collect the raw tracing data - and a set of tools to control, visualize and analyze the generated data. It also provides support for user space application instrumentation.

At present, the LTTng plug-ins support the following kernel-oriented views:
- Control Flow - to visualize processes state transitions
- Resources - to visualize system resources state transitions
- CPU Usage - to visualize the usage of the processor with respect to the time in traces
- Kernel Memory Usage - to visualize the relative usage of system memory
- IO Usage - to visualize the usage of input/output devices
- System Calls - presents all the system calls in a table view
- System Call Statistics - present all the system calls statistics
- System Call Density - to visualize the system calls displayed by duration
- System Call vs Time - to visualize when system calls occur

Also, the LTTng plug-ins supports the following User Space traces views:
- Memory Usage - to visualize the memory usage per thread with respect to time in the traces
- Call Stack - to visualize the call stack's evolution over time
- Function Duration Density - to visualize function calls displayed by duration
- Flame Graph - to visualize why the CPU is busy

Finally, the LTTng plug-ins supports the following Control views:
- Control - to control the tracer and configure the tracepoints

#### Flame graphs - for any profiles with stack traces
TBD

#### Heat maps - to show distributions over time.
TBD

#### Vector - Opensource

**Keep this in TO BE WATCH LIST**. Important to note that its a on-demand instance analysis in kernel level detail. You can generate Flame graphs, heat maps for quicker analysis. **Best suitable for container monitoring.** It gives the capability to monitor application stack and kernel stack in mix mode.

[Vector](http://vectoross.io/) is an open source (Apache License v2), [NetflexOSS](https://github.com/Netflix/vector), on-host performance monitoring framework which exposes hand picked high resolution system and application metrics to every engineer’s browser. Having the right metrics available on-demand and at a high resolution is key to understand how a system behaves and correctly troubleshoot performance issues.
- Vector provides access to high-resolution metrics, up to 1 second.
- Metrics come directly from the monitored host, in near real-time.
- The monitoring overhead is close to zero and the agent is completely idle while not being used.
- Only the necessary metrics are collected for monitoring, keeping the framework very light.
- Completely configurable dashboards provide simple cross-metric correlation and analysis.
- Vector is open source and can be easily extended to include more metrics and widgets.
- Cloud agnostic

**Agent**: Vector depends on Performance Co-Pilot (PCP) to collect metrics on each host you plan to monitor. **PCP offers a multitude of APIs and libraries to extract and make use of performance metrics from your own application.** PCP’s stateless model makes it lightweight and robust. Its overhead on hosts is negligible, as clients are responsible for keeping track of state, sampling rate, and computation. Additionally, metrics are not aggregated across hosts or persisted outside of the user’s browser session, keeping the framework light.
**Application**: Vector is just a static application, you should be able to easily deploy it to any web server. For example, using Ubuntu with Apache 2. Once the Vector UI is loaded, just enter the hostname or IP from the host you installed PCP on, and start collecting metrics!

<B>Vector Architecture </B>
![Architecture](/Images/Vector-architecture.png)

<B> Vector Screenshot </B>
![Vector screenshot](/Images/Vector-screenshot.png)

<B> Flame graph in mixed mode <B>
![Flame-graph](/Images/Flame-Graph-Mixed-Mode.png)

[Back to Performance Analysis Tools](/Performance_Analysis_Tools.md#Linux Performance Observation tools)
