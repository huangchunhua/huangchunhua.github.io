+++
author = "黄春华"
categories = ["Linux"]
date = "2018-07-19"
featured = "pic03.jpg"
featuredalt = ""
featuredpath = "date"
linktitle = ""
title = "Linux Cgroups资源限制配置参数用法"
type = "post"

+++
背景
====
Cgroups是control groups的缩写，是Linux内核提供的一种可以限制、记录、隔离进程组（process groups）所使用的物理资源（如：cpu,memory,IO等等）的机制。最初由google的工程师提出，后来被整合进Linux内核。本文重点cgroups配置参数用法，目的是能为后续容器开发提供帮助。

配置参数用法
====

1.blkio - BLOCK IO资源管理
----
**限额类**
限额类是主要有两种策略，一种是基于完全公平队列调度的按权重分配各个cgroup所能占用总体资源的百分比。好处是当资源空闲时可以充分利用，但只能用于最底层节点cgroup的配置；另一种则是设定资源使用上限。这种限额在各个层次的cgroup都可以配置，但这种限制较为生硬并且容器之间依然会出现资源的竞争。

 - 按比例分配块设备IO资源
     - **blkio.weight**: 指定 cgroup 默认可用 IO 的比例（加权），值的范围为"100~1000"。该值可由具体设备的blkio.weight_device参数覆盖。例如：设置cgroup lv0的默认加权为 500： **cgset -r blkio.weight=100 lv0**。
     - **blkio.weight\_device**：指定 cgroup 中指定设备的可用 IO 比例（加权），范围为"100~1000"，格式为"major:minor weight"。 例如：设置 cgroup lv0 对 /dev/sda 的加权为500。 **cgset -r blkio.weight_device="8:0 500" lv0**。
- 控制IO读写速度上限
	- **blkio.throttle.read_bps_device / blkio.throttle.write_bps_device**：指定cgroup中某设备每秒钟读/写数据的字节上限。格式：major:minor bytes_per_second。例如：设置 cgroup lv0在/dev/sdc上每秒最多读 10KiB 数据： **cgset -r blkio.throttle.read_bps_device="8:32 10240" lv0**。
	- **blkio.throttle.read_iops_device / blkio.throttle.write\_iops_device**：指定cgroup中某设备每秒钟可以执行的读/写请求数上限。格式: major:minor operations_per_second。例如：设置cgroup lv0在/dev/sdc上每秒最多执行1000次读请求： cgset -r blkio.throttle.read_iops_device="8:32 1000" lv0
- 针对特定操作(read, write, sync, 或async)设定读写速度上限
	- **blkio.throttle.io_serviced**：针对特定操作按每秒操作次数设定上限，格式：major:minor operation operations_per_second
	- **blkio.throttle.io_service_bytes**：针对特定操作按每秒数据量设定上限，格式major:minor operation bytes_per_second

- 统计与监控
	- **blkio.reset_stats**：向该文件中写入一个整数，可以重置该 cgroup 中的报告计数。
	- **blkio.time**：统计cgroup对设备的访问时间，格式:major:minor milliseconds读取信息即可.
	- **blkio.io_serviced**：统计cgroup对特定设备的IO操作（包括read、write、sync及async）次数，格式：major:minor operation number
	- **blkio.sectors**：统计cgroup对设备扇区访问次数，格式：major:minor sector_count
	- **blkio.io_service_bytes**：统计cgroup对特定设备IO操作（包括read、write、sync及async）的数据量，格式：major:minor operation bytes
	- **blkio.io_queued**：统计cgroup的队列中对IO操作（包括read、write、sync及async）的请求次数，格式number operation
	- **blkio.io_service_time**：统计cgroup对特定设备的IO操作（包括read、write、sync及async）时间(单位为ns)，格式：major:minor operation time
	- **blkio.io_merged**：统计cgroup 将 BIOS 请求合并到IO操作（包括read、write、sync及async）请求的次数，格式number operation
	- **blkio.io_wait_time**：统计cgroup在各设​​​备​​​中各类型​​​IO操作（包括read、write、sync及async）在队列中的等待时间​(单位ns)，格式：major:minor operation time
	- **__blkio.__recursive_***：各类型的统计都有一个递归版本，Docker中使用的都是这个版本。获取的数据与非递归版本是一样的，但是包括cgroup所有层级的监控数据。

2.CPU资源管理
----
- CPU资源控制
	- **cpu.rt_period_us**：设定周期时间。
	- **cpu.rt_runtime_us**：设定周期中的运行时间。
	- **cpu.cfs_period_us**：设定周期时间，必须与cfs_quota_us配合使用。
	- **cpu.cfs_quota_us**：设定周期内最多可使用的时间。这里的配置指task对单个cpu的使用上限，若cfs_quota_us是cfs_period_us的两倍，就表示在两个核上完全使用。数值范围为1000 - 1000,000（微秒）。
	- **cpu.stat**：统计信息，包含nr_periods（表示经历了几个cfs_period_us周期）、nr_throttled（表示task被限制的次数）及throttled_time（表示task被限制的总时长）。
- CPU绑定
	- **cpuset.cpus**：可使用的CPU编号，如0-2,16代表 0、1、2和16这4个CPU。
	- **cpuset.mems**：与CPU类似，表示cgroup可使用的memory node，格式同上.
- CPU资源报告
	- **cpuacct.usage**：统计cgroup中所有task的cpu使用时长
	- **cpuacct.stat**：统计cgroup中所有task的用户态和内核态分别使用cpu的时长
	- **cpuacct.usage_percpu**：统计cgroup中所有task使用每个cpu的时长
	- **cpu.shares**：设定一个整数（必须大于等于2）表示相对权重，最后除以权重总和算出相对比例如：按比例分配CPU时间。（cgroup A设置100，cgroup B设置300，那么cgroup A中的task运行25%的CPU时间。对于一个4核CPU的系统来说，cgroup A 中的task可以100%占有某一个CPU，这个比例是相对整体的一个值。）

3.device - 限制task对device的使用
----
- 设置黑白名单
	- **devices.allow**：允许名单，语法type device_types:node_numbers access type ；type有三种类型：b（块设备）、c（字符设备）、a（全部设备）；access也有三种方式：r（读）、w（写）、m（创建）。
	- **devices.deny**：禁止名单，语法格式同上。

- 统计报告
	- **devices.list**：报​​​告​​​为​​​这​​​个​​​ cgroup 中​​​的​task设​​​定​​​访​​​问​​​控​​​制​​​的​​​设​​​备

4.内存资源管理
----
- 设置限额
	- **memory.limit_bytes**：强制限制最大内存使用量。单位有k、m、g三种，填-1则代表无限制。
	- **memory.soft_limit_bytes**：软限制，必须比强制限制设置的值小时才有意义。填写格式同上。当整体内存紧张的情况下，task获取的内存就被限制在软限制额度之内，以保证不会有太多进程因内存不足。加入了内存的资源限制并不代表没有资源竞争。
	- **memory.memsw.limit_bytes**：设定最大内存与swap区内存之和的用量限制。填写格式同上。
	
- 报警与自动控制
	- **memory.oom_control**：值：0或1。0表示开启，当cgroup中的进程使用资源超过界限时立即杀死进程；1表示不启用。默认情况下，包含memory子系统的cgroup都**启用**。当oom_control不启用时，实际使用内存超过界限时进程会被暂停直到有空闲的内存资源。
- 统计与监控类
	- **memory.usage_bytes**：报​​​告​​​该​​​cgroup中​​​进​​​程​​​使​​​用​​​的​​​当​​​前​​​总​​​内​​​存​​​用​​​量（以字节为单位）。
	- **memory.max_usage_bytes**：报​​​告​​​该​​​cgroup中​​​进​​​程​​​使​​​用​​​的​​​最​​​大​​​内​​​存​​​用​​​量。
	- **memory.failcnt**：报​​​告​​​内​​​存​​​达​​​到​​​在​​​ memory.limit_in_bytes设​​​定​​​的​​​限​​​制​​​值​​​的​​​次​​​数​​​。
	- **memory.stat**：包含大量的内存统计数据。
	- **cache**：页​​​缓​​​存​​​，包​​​括​​​ tmpfs（shmem），单位为字节。
	- **rss**：匿​​​名​​​和​​​ swap 缓​​​存​​​，不​​​包​​​括​​​ tmpfs（shmem），单位为字节。
	- **mapped_file**：memory-mapped 映​​​射​​​的​​​文​​​件​​​大​​​小​​​，包​​​括​​​ tmpfs（shmem），单​​​位​​​为​​​字​​​节​​​
	- **pgpgin**：存​​​入​​​内​​​存​​​中​​​的​​​页​​​数​​​。
	- **pgpgout**：从​​​内​​​存​​​中​​​读​​​出​​​的​​​页​​​数。
	- **swap**：swap 用​​​量​​​，单​​​位​​​为​​​字​​​节​​​。
	- **active_anon**：在​​​活​​​跃​​​的​​​最​​​近​​​最​​​少​​​使​​​用​​​（least-recently-used，LRU）列​​​表​​​中​​​的​​​匿​​​名​​​和​​​ swap 缓​​​存​​​，包​​​括​​​ tmpfs（shmem），单​​​位​​​为​​​字​​​节​​​。
	- **inactive_anon**：不​​​活​​​跃​​​的​​​ LRU 列​​​表​​​中​​​的​​​匿​​​名​​​和​​​ swap 缓​​​存​​​，包​​​括​​​ tmpfs（shmem），单​​​位​​​为​​​字​​​节。
	- **active_file**：活​​​跃​​​ LRU 列​​​表​​​中​​​的​​​ file-backed 内​​​存​​​，以​​​字​​​节​​​为​​​单​​​位。
	- **inactive_file**：不​​​活​​​跃​​​ LRU 列​​​表​​​中​​​的​​​ file-backed 内​​​存​​​，以​​​字​​​节​​​为​​​单​​​位.
	- **unevictable**：无​​​法​​​再​​​生​​​的​​​内​​​存​​​，以​​​字​​​节​​​为​​​单​​​位​​​
	- **hierarchical_memory_limit**：包​​​含​​​ memory cgroup 的​​​层​​​级​​​的​​​内​​​存​​​限​​​制​​​，单​​​位​​​为​​​字​​​节​​​
	- **hierarchical_memsw_limit**：包​​​含​​​ memory cgroup 的​​​层​​​级​​​的​​​内​​​存​​​加​​​ swap 限​​​制​​​，单​​​位​​​为​​​字​​​节​​​.

更多参考
----
- https://www.kernel.org/doc/Documentation/cgroup-v1/
- https://www.kernel.org/doc/Documentation/cgroup-v2.txt


作者：黄春华