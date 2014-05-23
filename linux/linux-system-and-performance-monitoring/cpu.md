## 1.0 Performance Monitoring Introduction

* Performance tuning is the process of finding bottlenecks in a system and tuning the operation system to eliminate these bootlnecks.

* Performance tuning is about achieving balance between the diferent sub-systems of an OS. These sub-systems include:
  - CPU
  - Memory
  - IO
  - Network

* Example:
  - Large amounts of page-in IO requests can fill the memory queues
  - full gigabit throughout on an Ethernet controller may consume a CPU
  - a CPU may be consumed attempting to maintain free memory queues
  - a large number of disk write requests from memory may consume a CPU and IO channels

## 1.1 Determining Application Type

The application stack of any system is often broken down into two types:
  - __IO Bound__ - An IO bound application requires heavy use of memory and the underlying storage system. This is due to the large amounts of data. An IO bound application does not require much of the CPU or network (unless the storage system is on a network). IO bound applications use CPU resources to make IO requests and then often go into a sleep state. Database applications are often considered IO bound applications.
  - __CPU Bound__ - A CPU bound application requires heavy use of the CPU. CPU bound applications require the CPU for batch processing and/or mathematical calculations. High volume web servers, mail servers, and any kind of rendering server are often considered CPU bound applications.

## 1.2 Determining Baseline Statistics

  * Statistics must be available for a system under acceptable performance so it can be compared later against unacceptable performance.

## 2.0 Installing Monitoring Tools

* This paper describes how to monitor performance using the following tools.
  - `vmstat`: all purpose performance tool
  - `mpstat`: provides statistics per CPU
  - ` sar`: all purpose performance monitoring tool
  - `iostat`: provides disk statistics
  - `netstat`: provides network statistics
  - `dstat`: monitoring statistics aggregator
  - `iptraf`: traffic monitoring dashboard
  - `netperf`: Network bandwidth tool
  - `ethtool`: reports on Ethernet interface configuration
  - `iperf`: Network bandwidth tool
  - `iotop`: Displays IO per process

## 3.0 Introducing the CPU

* The utilization of a CPU is largely dependent on what resource is attempting to access it. The kernel has scheduler that is responsible for scheduling two kinds of resources: threads (single or multi) and interrupts.

* The scheduler gives different priorities to the different resources. The following list outlines the priorities from highest to lowest:
  - Interrupts
  - Kernel (System) Processes
  - User Processes

## 3.1 Context Switches

* Most modern processors can only run one process (single threaded) or thread at time. The Linux kernel views each processor core on a dual core chip as an independent processor. For example, a system with one dual core processor is reported as two individual processors by the Linux kernel.

* A standard Linux kernel can run anywhere from 50 to 50,000 process threads at once. With only one CPU, the kernel has to schedule and balance these process threads.

* Each thread has an allotted time quantum to spend on the processor. Once a thread has either passed the time quantum or has been preempted by something with a higher priority (a hardware interrupt, for example), the thread is placed back into a queue while the higher priority thread is placed on the processor. This switching of threads is referred to as a context switch.

* Every time the kernel conducts a context switch, resources are devoted to moving that thread off of the CPU registers and into a queue. The higher the volume of context switches on a system, the more work the kernel has to do in order to manage the scheduling of processes.

## 3.2 The Run Queue

* Each CPU maintains a run queue of threads.

* Process threads are either in a sleep state (blocked and waiting on IO) or they are runnable. If the CPU sub-system is heavily utilized, then it is possible that the kernel scheduler can’t keep up with the demand of the system. As a result, runnable processes start to fill up a run queue. The larger the run queue, the longer it will take for process threads to execute.

* The system load is a combination of the amount of process threads currently executing along with the amount of threads in the CPU run queue. If two threads were executing on a dual core system and 4 were in the run queue, then the load would be 6. Utilities such as top report load averages over the course of 1, 5, and 15 minutes.

## 3.3 CPU Utilization

* Most performance monitoring tools categorize CPU utilization into the following categories:
  - __User Time__ - The percentage of time a CPU spends executing process threads in the user space.
  - __System Time__ - The percentage of time the CPU spends executing kernel threads and interrupts.
  - __Wait IO__ - The percentage of time a CPU spends idle because ALL process threads are blocked waiting for IO requests to complete.
  - __Idel__ - The percentage of time a processor spends in a completely idle state.

## 4.0 CPU Performance Monitoring

* There are some general performance expectations on any system.
  - __Run Queues__ - A run queue should have no more than 1-3 threads queued per processor. For example, a dual processor system should not have more than 6 threads in the run queue.
  - __CPU Utilization__ - If a CPU is fully utilized, then the following balance of utilization should be achieved.
  - 65% - 70% User Time
  - 30% - 35% System Time
  - 0% - 5% Idle Time
  - __Context Switches__ - The amount of context switches is directly to CPU utilization. A high amount of context switching is acceptable if CPU utilization stays within the previously mentioned balance.

## 4.1 Using the `vmstat` Utility

* The following example demonstrates `vmstat` running at 1 second intervals.

![vmstat](https://raw.githubusercontent.com/larrylv/hammer/master/linux/linux-system-and-performance-monitoring/images/vmstat.png)

![vmstat_table](https://raw.githubusercontent.com/larrylv/hammer/master/linux/linux-system-and-performance-monitoring/images/vmstat_table.png)

## 4.2 Case Study: Sustained CPU Utilization

![vmstat_sustained](https://raw.githubusercontent.com/larrylv/hammer/master/linux/linux-system-and-performance-monitoring/images/vmstat_sustained.png)

* The following observations are made from the output:
  - There are a high amount of interrupts (in) and a low amount of context switches. It appears that a single process is making requests to hardware devices.
  - To further prove the presence of a single application, the user (us) time is constantly at 85% and above. Along with the low amount of context switches, the process comes on the processor and stays on the processor.
  - The run queue is just about at the limits of acceptable performance. On a couple occasions, it goes beyond acceptable limits.

## 4.3 Case Study: Overloaded Scheduler

![vmstat_overloaded](https://raw.githubusercontent.com/larrylv/hammer/master/linux/linux-system-and-performance-monitoring/images/vmstat_overloaded.png)

* The following conclusions can be drawn from the output:
  - The amount of context switches is higher than interrupts, suggesting that the kernel has to spend a considerable amount of time context switching threads.
  - The high volume of context switches is causing an unhealthy balance of CPU utilization. This is evident by the fact that the wait on IO percentage is extremely high and the user percentage is extremely low.
  - Because the CPU is block waiting for I/O, the run queue starts to fill and the amount of threads blocked waiting on I/O also fills.

## 4.4 Using the `mpstat` Utility

![mpstat](https://raw.githubusercontent.com/larrylv/hammer/master/linux/linux-system-and-performance-monitoring/images/mpstat.png)

* If your system has multiple processor cores, you can use the `mpstat` command to monitor each individual core. The Linux kernel treats a dual core processor as 2 CPU’s. So, a dual processor system with dual cores will report 4 CPUs available.

* The `mpstat` command provides the same CPU utilization statistics as `vmstat`, but `mpstat` breaks the statistics out on a per processor basis.

## 4.5 Case Study: Underutilized Process Load

## 4.6 Conclusion

Monitoring CPU performance consists of the following actions:

* Check the system run queue and make sure there are no more than 3 runnable threads per processor.

* Make sure the CPU utilization is split between 70/30 between user and system.

* When the CPU spends more time in system mode, it is more than likely overloaded and trying to reschedule priorities.

* Running CPU bound process always get penalized while I/O process are rewarded.
