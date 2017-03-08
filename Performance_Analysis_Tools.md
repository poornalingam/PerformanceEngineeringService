# How and What will be analyzed
- [Performance Metrics for all software and physical stack](/Performance-Metric.md)
- [K8S and Container level performance analysis](/K8S-Container-performance-monitoring.md)
- [MicroServices performance analysis](/MicroService_performance_assessment.md)
- [Techniques for performance analysis](/Method-PerformanceAnalysis.md)
- Scripting language specific performance analysis

# Tools for Performance Analysis

Following tools are useful for Performance analysis. Choosing the right tools is a first step.

Tools | License| Home grown Appl | Vendor Appl/ Tools | Web Appl | RESTfull Appl | Container | Micro Services | External Comm.
:---- | :----:|:-----:| :----: | :----: | :-----:| :----:| :-----: | :-----: | :-----:|  
[Linux performance observation tools](/Images/linux_perf_tools_full.jpg) | Open Source| [x] | [x] | [x] | [x] | [x] | [x] | [x]
Sensu or Telegraf  | Open Source | [x]|[x]|[x]|[X]|||
nmon or perfmon | Open Source|[x]|[x]|[x]|[x]|[x]||
Elastic Stack  | Open Source|[x]|[x]|[x]|[x]|[x]||
Oracle JMC - Java Mission Control| Commercial License|[x]|[x]|[X]|[X]|[x]||
DynaTrace| JVM based License|[X]|[x]|[x]||||
InspectIT | Open Source|[x]|[x]|[x]||[x]||
PinPoint | Open Source|[x]|[x]|[x]|[?]||[X]|
Google Dapper  | Open Source||||||[X]|
ZipKin | Open Source||||||[X]|
DCRUM  | Ent License|[X]|[X]|[X]|[X]|[X]||[X]
Fiddler - packet analyzer |??| [X] | [x] | [x] | [x]||[x]|
Datatorrent RTS  |Ent License|[X]|[X]|[X]|[X]|||
HttpWatch | Open Source| [x]|[x]|[X]|[X]|||
Oracle AWR  |Ent License|[x]|[x]|[X]||||
Oracle Enterprise Manager|Ent License|[x]|[x]|[x]||||
Dell Spotlight on Oracle RAC |License|[x]|[x]|[x]||||
Heapdum analyzer| Open Source|[X]|[X]|[X]|[X]|||
Netflow | Ent License|[X]|[X]|[X]|[X]|||

## Linux Performance Observation tools
The bellow tools are primary to analysis system latency in every level and tracing. Analyzing the kernel level call will enable to find bottleneck, resource constrains, utilization, saturation, errors. These tools are basic need for USE method, Lantency method. Create the one liner commands using these tools and use it as when need. This article provides few one liner for quick analysis. It must be updated as we progress.

- [ftrace](Linux-Performance-Observation-Tools.md#ftrace)
- [perf](Linux-Performance-Observation-Tools.md#perf)
- [eBPF - extended BPF](Linux-Performance-Observation-Tools.md#extended-bpf)
- [dtrace - Dynamic tracing](Linux-Performance-Observation-Tools.md#dtrace---dynamic-tracing)
- [SAR - System Activity Reporter](Linux-Performance-Observation-Tools.md#sar---system-activity-reporter)
- [Opensource tools - perf-tool](Linux-Performance-Observation-Tools.md#opensource-tools)
- [Visual tools](Linux-Performance-Observation-Tools.md#visual-tools)
  1. Kernelshark - for ftrace
  2. Trace Compass - to visualize LTTng (Linux Trace Toolkit, next generation) time service trace data
  3. Flame graphs - for any profiles with stack traces
  4. Heat maps - to show distributions over time

Most of these tools are supporting CentOS as well.

## Sensu
TBD

## Elastic Stack - Metricbeat
TBD

## [Oracle JMC](http://www.oracle.com/technetwork/java/javase/2col/jmc-relnotes-2004763.html)
  The JDK 7 update 40 release includes the first release of Java Mission Control (JMC) that is bundled with the Hotspot JVM.

  * The JVM Browser now have subnodes for available server side services that show the state of services. The new JVM Browser can be viewed in two different modes; as a flat list, and as a tree.
  * The Mission Control client is now built to run on Eclipse 3.8.2/4.2.2 and later.
  * Java Flight Recorder (JFR), Event Convergence with JRockit; the same useful information that was provided by JRockit VM is now also available from Hotspot VM.
  * Method Profiling Events
  * DTrace Plug-in is only available for JDK running on Solaris, but client can run on JMC platforms. Watch out future release for Linux.

## APM Family Tools
### DynaTrace
TBD
### InspectIT
TBD
### PinPoint
TBD

## Container and MicroService Monitoring
### Google Dapper

  It’s a google white paper to build a tool to track microservices and its latency.
  ![Google Dapper](/Images/MicroService_Monitoring.jpg)

### ZipKin
TBD

## Packet Analysis
### Fiddler

Fiddler is a free and open-source packet analyzer. It is used for network troubleshooting, analysis, software, communications protocol development and education. Fiddler captures HTTP and HTTPS traffic data between browsers and servers. These data are extremely valuable for troubleshooting, performance turning and system monitoring.

### [HttpWatch](https://www.httpwatch.com/features/httpdebugger.aspx)

HttpWatch integrates with Internet Explorer and Firefox browsers to show you the HTTP and HTTPS traffic that is generated when you access a web page. Select a request in HttpWatch and everything you need to know is display in a tabbed window. Cookies, Headers, Query Strings and POST data can be quickly viewed, searched and exported to other formats.
* Understand http headers without being an expert
* Handle multi-page scenarios with page grouping
* Real time page level time charts
* Millisecond accurate request timings
* Page event timings
* Automatically detects performance issues
* Hassle-free access to https traffic

[Back to Top](#how-and-what-will-be-analyzed)
