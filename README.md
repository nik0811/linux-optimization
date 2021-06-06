<p align="center">
  <img href="http://www.brendangregg.com/Perf/linux_perf_tools_full.svg" src="./logo/log.svg" width="40%">
  <h1 align="center">
    IMPORTANT LINUX TROUBLESHOOTING COMMAND
  </h1>
<p align="center">Linux Optimization Command</p>
</p>

```
sudo apt-get install sysstat
uptime
dmesg | tail
vmstat 1
mpstat -P ALL 1
pidstat 1
iostat -xz 1
free -m
sar -n DEV 1
sar -n TCP,ETCP 1
top
```

# uptime

The three numbers are exponentially damped moving sum averages with a 1 minute, 5 minute, and 15 minute constant. The three numbers give us some idea of how load is changing over time. For example, if you’ve been asked to check a problem server, and the 1 minute value is much lower than the 15 minute value, then you might have logged in too late and missed the issue.

In the example above, the load averages show a recent increase, hitting 30 for the 1 minute value, compared to 19 for the 15 minute value. That the numbers are this large means a lot of something: probably CPU demand; vmstat or mpstat will confirm, which are commands 3 and 4 in this sequence.


# dmesg | tail

This views the last 10 system messages, if there are any. Look for errors that can cause performance issues. The example above includes the oom-killer, and TCP dropping a request.

Don’t miss this step! dmesg is always worth checking.


# vmstat 1

Short for virtual memory stat, vmstat(8) is a commonly available tool (first created for BSD decades ago). It prints a summary of key server statistics on each line.

vmstat was run with an argument of 1, to print one second summaries. The first line of output (in this version of vmstat) has some columns that show the average since boot, instead of the previous second. For now, skip the first line, unless you want to learn and remember which column is which.

Columns to check:

   1. r: Number of processes running on CPU and waiting for a turn. This provides a better signal than load averages for determining CPU saturation, as it does not include I/O. To interpret: an “r” value greater than the CPU count is saturation.

  2. free: Free memory in kilobytes. If there are too many digits to count, you have enough free memory. The “free -m” command, included as command 7, better explains the state of free memory.

  3. si, so: Swap-ins and swap-outs. If these are non-zero, you’re out of memory.

  4. us, sy, id, wa, st: These are breakdowns of CPU time, on average across all CPUs. They are user time, system time (kernel), idle, wait I/O, and stolen time (by other guests, or with Xen, the guest’s own isolated driver domain).

The CPU time breakdowns will confirm if the CPUs are busy, by adding user + system time. A constant degree of wait I/O points to a disk bottleneck; this is where the CPUs are idle, because tasks are blocked waiting for pending disk I/O. You can treat wait I/O as another form of CPU idle, one that gives a clue as to why they are idle.

System time is necessary for I/O processing. A high system time average, over 20%, can be interesting to explore further: perhaps the kernel is processing the I/O inefficiently.

In the above example, CPU time is almost entirely in user-level, pointing to application level usage instead. The CPUs are also well over 90% utilized on average. This isn’t necessarily a problem; check for the degree of saturation using the “r” column.

# mpstat -P ALL 1

This command prints CPU time breakdowns per CPU, which can be used to check for an imbalance. A single hot CPU can be evidence of a single-threaded application.

# pidstat 1

Pidstat is a little like top’s per-process summary, but prints a rolling summary instead of clearing the screen. This can be useful for watching patterns over time, and also recording what you saw (copy-n-paste) into a record of your investigation

# iostat -xz 1

This is a great tool for understanding block devices (disks), both the workload applied and the resulting performance. Look for:

    1. r/s, w/s, rkB/s, wkB/s: These are the delivered reads, writes, read Kbytes, and write Kbytes per second to the device. Use these for workload characterization. A performance problem may simply be due to an excessive load applied.

    2. await: The average time for the I/O in milliseconds. This is the time that the application suffers, as it includes both time queued and time being serviced. Larger than expected average times can be an indicator of device saturation, or device problems.

    3. avgqu-sz: The average number of requests issued to the device. Values greater than 1 can be evidence of saturation (although devices can typically operate on requests in parallel, especially virtual devices which front multiple back-end disks.)

    4. %util: Device utilization. This is really a busy percent, showing the time each second that the device was doing work. Values greater than 60% typically lead to poor performance (which should be seen in await), although it depends on the device. Values close to 100% usually indicate saturation.

If the storage device is a logical disk device fronting many back-end disks, then 100% utilization may just mean that some I/O is being processed 100% of the time, however, the back-end disks may be far from saturated, and may be able to handle much more work.

Bear in mind that poor performing disk I/O isn’t necessarily an application issue. Many techniques are typically used to perform I/O asynchronously, so that the application doesn’t block and suffer the latency directly (e.g., read-ahead for reads, and buffering for writes).

# free -m

    1. buffers: For the buffer cache, used for block device I/O.

    2. cached: For the page cache, used by file systems.

We just want to check that these aren’t near-zero in size, which can lead to higher disk I/O (confirm using iostat), and worse performance. The above example looks fine, with many Mbytes in each.

The “-/+ buffers/cache” provides less confusing values for used and free memory. Linux uses free memory for the caches, but can reclaim it quickly if applications need it. So in a way the cached memory should be included in the free memory column, which this line does. There’s even a website, linuxatemyram, about this confusion.

It can be additionally confusing if ZFS on Linux is used, as we do for some services, as ZFS has its own file system cache that isn’t reflected properly by the free -m columns. It can appear that the system is low on free memory, when that memory is in fact available for use from the ZFS cache as needed.

# sar -n DEV 1

Use this tool to check network interface throughput: rxkB/s and txkB/s, as a measure of workload, and also to check if any limit has been reached. In the 

This version also has %ifutil for device utilization (max of both directions for full duplex), which is something we also use Brendan’s nicstat tool to measure. And like with nicstat, this is hard to get right, and seems to not be working in this example (0.00).


# sar -n TCP,ETCP 1

This is a summarized view of some key TCP metrics. These include:

    1. active/s: Number of locally-initiated TCP connections per second (e.g., via connect()).

    2. passive/s: Number of remotely-initiated TCP connections per second (e.g., via accept()).

    3. retrans/s: Number of TCP retransmits per second.

The active and passive counts are often useful as a rough measure of server load: number of new accepted connections (passive), and number of downstream connections (active). It might help to think of active as outbound, and passive as inbound, but this isn’t strictly true (e.g., consider a localhost to localhost connection).

Retransmits are a sign of a network or server issue; it may be an unreliable network (e.g., the public Internet), or it may be due a server being overloaded and dropping packets. The example above shows just one new TCP connection per-second.

# top

The top command includes many of the metrics we checked earlier. It can be handy to run it to see if anything looks wildly different from the earlier commands, which would indicate that load is variable.

A downside to top is that it is harder to see patterns over time, which may be more clear in tools like vmstat and pidstat, which provide rolling output. Evidence of intermittent issues can also be lost if you don’t pause the output quick enough (Ctrl-S to pause, Ctrl-Q to continue), and the screen clears.

