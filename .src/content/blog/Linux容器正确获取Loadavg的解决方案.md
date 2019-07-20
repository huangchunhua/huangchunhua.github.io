+++
author = "黄春华"
categories = ["Linux"]
date = "2018-07-01"
featured = "pic04.jpg"
featuredalt = ""
featuredpath = "date"
linktitle = ""
title = "Linux容器正确获取Loadavg的解决方案"
type = "post"
+++

背景
----
本文主要解决Linux容器中正确获取Loadavg信息。我们cat /proc/loadavg时会发现如下值：

	$ > cat /proc/loadavg
	0.64 0.81 0.86 3/364 6930

这些值的含义依次为：

* 0.64：1-分钟平均负载。
* 0.81：5-分钟平均负载。
* 0.86：15-分钟平均负载。
* 3: 在采样时刻，运行队列的任务的数目。
* 364: 在采样时刻，系统中活跃的任务的个数（不包括运行已经结束的任务）。
* 6930: 最大的pid值，包括轻量级进程，即线程。

平均负载定义：在特定时间间隔内运行队列中的平均进程数。

进程状态定义
---

* R (TASK_RUNNING)，可执行状态。
* S (TASK_INTERRUPTIBLE)，可中断的睡眠状态。
* D (TASK_UNINTERRUPTIBLE)，不可中断的睡眠状态。
* T (TASK_STOPPED or TASK_TRACED)，暂停状态或跟踪状态。
* Z (TASK_DEAD – EXIT_ZOMBIE)，退出状态，进程成为僵尸进程。
* X (TASK_DEAD – EXIT_DEAD)，退出状态，进程即将被销毁。

如何计算Loadavg?
---

计算公式：**load(t) = load(t-1) * exp(-5/60R) + n(t) * (1 – exp(5/60R))**

	n(t)是系统活动的进程数， R对应1、5、15分钟（如当计算15分钟的平均负载时，R的值就为15）。

Linux内核认为进程的生存时间服从参数为1的指数分布，指数分布的概率密度为：内核计算负载load1为例，设相邻两个计算时刻之间系统活动的进程集合为S0。从1分钟前到当前计算时刻这段时间里面活动的load1个进程，设他们的集合是S1，内核认为的概率密度是:λe-λx，而在当前时刻活动的n个进程，设他们的集合是Sn内核认为的概率密度是1-λe-λx。其中 x = 5 / 60，因为相邻两个计算时刻之间进程所耗的CPU时间为5秒，而考虑的时间段是1分钟(60秒)。那么可以求出最近1分钟系统运行队列的长度：

	load1 
	 		= |S1|  * λe-λx + |Sn| * (1-λe-λx) 
		    = load1 * λe-λx + n * (1-λe-λx)

	其中λ = 1， x = 5 / 60， |S1|和|Sn|是集合元素的个数。

Linux内核定义一个unsigned long类型数组avenrun[3]，因为内核不能使用浮点数，就将低11位用于存放负载的小数部分，高21位用于存放整数部分。

    avenrun[0]对应前1分钟系统负载；

    avenrun[1]对应前5分钟系统负载；

    avenrun[2]对应前15分钟系统负载；

内核每隔**5秒钟**更新一次load average的值。


内核如何实现Loadavg?
-----
在内核中Loadavg计算部分与读取部分是分开的。

**注：使用3.10.107版本内核源码**

- 读取部分。由下列源码我们可以看出，根据输出格式，LOAD_INT对应计算的是load的整数部分，LOAD__FRAC计算的是load的小数部分。loadavg_proc_show读取get_avenrun来获取1、5、15分钟的系统负载值。

	
```	
文件：fs/proc/loadavg.c
#define LOAD_INT(x) ((x) >> FSHIFT)
#define LOAD_FRAC(x) LOAD_INT(((x) & 	(FIXED_1-1)) * 100)

static int loadavg_proc_show(struct seq_file *m, void *v)
{
	unsigned long avnrun[3];

	get_avenrun(avnrun, FIXED_1/200, 0);

	seq_printf(m, "%lu.%02lu %lu.%02lu %lu.%02lu %ld/%d %d\n",
			LOAD_INT(avnrun[0]), LOAD_FRAC(avnrun[0]),
			LOAD_INT(avnrun[1]), LOAD_FRAC(avnrun[1]),
			LOAD_INT(avnrun[2]), LOAD_FRAC(avnrun[2]),
			nr_running(), nr_threads,
			task_active_pid_ns(current)->last_pid);
	return 0;
}

文件：kernel/sched/core.c
unsigned long avenrun[3];
void get_avenrun(unsigned long *loads, unsigned long offset, int shift)
{
	loads[0] = (avenrun[0] + offset) << shift;
	loads[1] = (avenrun[1] + offset) << shift;
	loads[2] = (avenrun[2] + offset) << shift;
}

unsigned long nr_running(void)
{
	unsigned long i, sum = 0;

	for_each_online_cpu(i)
		sum += cpu_rq(i)->nr_running;

	return sum;
}
	
```
	
	
- 计算部分。内核设计一个定时器，时钟一到就会去调用xtime_update（）-> do_timer()-> calc_global_load()函数，如果超时5s那么就会更新一次load数据，即：avenrun数组。具体如下内核源码：

```
//文件：include/linux/sched.h
#define FSHIFT		11		/* nr of bits of precision */
#define FIXED_1		(1<<FSHIFT)	/* 1.0 as fixed-point */
#define LOAD_FREQ	(5*HZ+1)	/* 5 sec intervals */
#define EXP_1		1884		/* 1/exp(5sec/1min) as fixed-point */
#define EXP_5		2014		/* 1/exp(5sec/5min) */
#define EXP_15		2037		/* 1/exp(5sec/15min) */


//文件：kernel/time/timekeeping.c
/**
 * xtime_update() - advances the timekeeping infrastructure
 * @ticks:	number of ticks, that have elapsed since the last call.
 *
 * Must be called with interrupts disabled.
 */
void xtime_update(unsigned long ticks)
{
	write_seqlock(&jiffies_lock);
	do_timer(ticks);
	write_sequnlock(&jiffies_lock);
}

/*
 * Must hold jiffies_lock
 */
void do_timer(unsigned long ticks)
{
	jiffies_64 += ticks;
	update_wall_time();
	calc_global_load(ticks);
}

// 文件：kernel/sched/core.c
/*
 * a1 = a0 * e + a * (1 - e)
 */
static unsigned long
calc_load(unsigned long load, unsigned long exp, unsigned long active)
{
	load *= exp;
	load += active * (FIXED_1 - exp);
	load += 1UL << (FSHIFT - 1);
	return load >> FSHIFT;
}

static unsigned long calc_load_update;
/*
 * calc_load - update the avenrun load estimates 10 ticks after the
 * CPUs have updated calc_load_tasks.
 */
void calc_global_load(unsigned long ticks)
{
	long active, delta;

	if (time_before(jiffies, calc_load_update + 10))
		return;

	/*
	 * Fold the 'old' idle-delta to include all NO_HZ cpus.
	 */
	delta = calc_load_fold_idle();
	if (delta)
		atomic_long_add(delta, &calc_load_tasks);

	active = atomic_long_read(&calc_load_tasks);
	active = active > 0 ? active * FIXED_1 : 0;

	avenrun[0] = calc_load(avenrun[0], EXP_1, active);
	avenrun[1] = calc_load(avenrun[1], EXP_5, active);
	avenrun[2] = calc_load(avenrun[2], EXP_15, active);

	calc_load_update += LOAD_FREQ;

	/*
	 * In case we idled for multiple LOAD_FREQ intervals, catch up in bulk.
	 */
	calc_global_nohz();
}

```

容器中获取Loadavg方案
-----
如果想在容器中获取正确Loadavg信息的那么就要具备以下几点：

* 获取运行在容器中的所有进程（包括：线程）。
* 获取运行在容器中的进程总数。
* 获取运行在容器中的所有进程运行状态。
* Loadavg计算公式。

目前cgroup已实现了获取容器中所有的进程ID，具体如下：

```
../cgroup/pids/<docker|lxc>/<id>/tasks
922
2038
2545
2546
2547
2548
```
cgroup中也可以获取运行在容器中的运行进程总数，具体如下：

```
../cgroup/pids/<docker|lxc>/<id>/pids.current
128
```
获取运行在容器中的所有进程后可以通过系统proc来获取进程状态，具体如下: 

```
cat /proc/2546/status
Name:	connmaster
State:	S (sleeping)
Tgid:	2546
Ngid:	0
Pid:	2546
PPid:	2545
TracerPid:	0
Uid:	669	669	669	669
Gid:	669	669	669	669
FDSize:	1024
Groups:	669
VmPeak:	 4621592 kB
VmSize:	 4621592 kB
VmLck:	       0 kB
VmPin:	       0 kB
VmHWM:	  157948 kB
VmRSS:	  152624 kB
RssAnon:	  149084 kB
RssFile:	    3540 kB
RssShmem:	       0 kB
VmData:	 4606308 kB
VmStk:	     136 kB
VmExe:	    8860 kB
VmLib:	    1972 kB
VmPTE:	     600 kB
VmSwap:	       0 kB
Threads:	34
SigQ:	0/600000
SigPnd:	0000000000000000
ShdPnd:	0000000000000000
SigBlk:	fffffffe7bfa7a25
SigIgn:	0000000000000001
SigCgt:	ffffffffffc1fefe
CapInh:	0000001fffffffff
CapPrm:	0000000000000000
CapEff:	0000000000000000
CapBnd:	0000001fffffffff
CapAmb:	0000000000000000
Seccomp:	0
Cpus_allowed:	000c,000003fc
Cpus_allowed_list:	2-9,34-35
Mems_allowed:	00000000,00000000,00000000,00000000,00000000,00000000,00000000,00000000,00000000,00000000,00000000,00000000,00000000,00000000,00000000,00000000,00000000,00000000,00000000,00000000,00000000,00000000,00000000,00000000,00000000,00000000,00000000,00000000,00000000,00000000,00000000,00000003
Mems_allowed_list:	0-1
voluntary_ctxt_switches:	88
nonvoluntary_ctxt_switches:	1
或
cat /proc/2546/stat
2546 (connmaster) S 2545 2545 2545 0 -1 1077944320 1279939 0 109 0 568906 239100 0 0 20 0 34 0 448464477 4732510208 38156 18446744073709551615 4194304 13266708 140727468767200 140727468766640 4591875 0 2080012837 1 2143420158 18446744073709551615 0 0 17 6 0 0 12 0 0 15363864 15540872 21028864 140727468774844 140727468774891 140727468774891 140727468777449 0

```

Loadavg计算公式上面已得知。


总结
-----
综合上面分析，只要在了解Loadavg的实现原理，那么就可以根据自己需求完成容器中正确获取Loadavg信息。

