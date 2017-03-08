# Database Read - Slow Disks issue

The Application team has complained of “slow disks” on one of their database servers.

Performance analyst (PA) first task is to learn more about the issue, gathering details to form a problem statement. The problem statement claims that the disks are slow, but it doesn’t explain if this is causing a database issue or not. As a PA responds by asking these questions:
	•  Is there currently a database performance issue? How is it measured?
	•  How long has this issue been present?
	•  Has anything changed with the database recently?
	•  Why were the disks suspected?

The application team replies: “We have a log for queries slower than 1,000 ms. These usually don’t happen, but during the past week they have been growing to dozens per hour. Server monitoring system showed that the disks were busy.”

PA login to basic server monitoring system (like Sensu, metricbeat), providing historical performance graphs based on operating system tools: mpstat(1), iostat(1), and others.

This confirms that there is a real database issue, but it also shows that the disk hypothesis is likely a guess. PA wants to check the disks, but he also wants to check other resources quickly in case that guess was wrong.

PA begins with a methodology called the USE method to quickly check for resource bottlenecks.

### Step 1: Utilization
As the database team reported, utilization for the disks is high, around 80%, while for the other resources (CPU, network) utilization is much lower. **The historical data shows that disk utilization has been steadily increasing during the past week, while CPU utilization has been steady**. Basic monitoring doesn’t provide saturation or error statistics for the disks, so to complete the USE method PA must log in to the server and run some commands.

### Step 2: Error
He checks disk error counters from /proc; they are zero.

### Step 3: saturation
PA runs "iostat -x 1" with an interval of one second and watches utilization and saturation metrics over time. Basic monitoring reported 80% utilization but uses a one-minute interval. At one-second granularity, **PA can see that disk utilization fluctuates, often hitting 100% and causing levels of saturation and increased disk I/O latency.**

To further confirm that this is blocking the database—and isn’t asynchronous with respect to the database queries—he uses a **dynamic tracing-based script to capture timestamps and database stack traces** whenever the database was descheduled by the kernel (off cpu or on and off dispatch queue). This shows that the database is often blocking during a file system read, during a query, and for many milliseconds. This is enough evidence for PA.

![Kernel-Tracing](/Images/Kernel-tracing.png)

**The next question is why.** The disk performance statistics appear to be consistent with high load. PA performs workload characterization to understand this further, using iostat(1) to measure IOPS, throughput, average disk I/O latency, and the read/write ratio.
  - From these, he also calculates the average I/O size and estimates the access pattern: random or sequential.
  - For more details, PA can use disk I/O level tracing; however, he is satisfied that this already points to a case of high disk load, and not a problem with the disks.

His summary so far is that the disks are under high load, which increases I/O latency and is slowing the queries. However, the disks appear to be acting normally for the load and CPU utilization was also steady. The rate of queries  has been steady.

PA thinks it could be file system fragmentation, which is expected when the file system approaches 100% capacity. He finds that it is only at 30%.

He remembers that this **disk I/O is largely caused by file system cache (page cache) misses**. He checks the file system cache hit rate and finds it is currently at 91%. This sounds high (good), but he has no historical data to compare it to. He logs in to other database servers that serve similar workloads and finds their cache hit rate to be over 97%. He also finds that the file system cache size is much larger on the other servers.

PA contacts the application development team and asks them to shut down the application and move it to a different server, referring to the database issue. After they do this, PA watches disk utilization creep downward, as the file system cache recovers to its original size. The slow queries return to zero.

[^ Back to top](#database-read---slow-disks-issue)
