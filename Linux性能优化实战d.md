#### 一  CPU性能篇

#### 1  CPU负载高问题排查

 每次发现系统变慢时，我们通常做的第一件事，就是执行 `top` 或者 `uptime` 命令，来了解系统的负载情况。

执行top命令结果：

<img src="https://raw.githubusercontent.com/yuejuntao/typoraImg/master/img/image-20201029182352038.png?token=ACGZFEGXN6JO2UN6REYDPPK7TKMHU" alt="image-20201029182352038" style="zoom:50%;" />

执行`uptime`执行结果：

<img src="https://raw.githubusercontent.com/yuejuntao/typoraImg/master/img/image-20201029182523674.png?token=ACGZFEFJCRT5T3P75ZND6ZK7TKMNG" alt="image-20201029182523674" style="zoom:50%;" />

    uptime的执行结果和top命令执行结果是第一行一样，其中前几列分别表示当前时间、系统运行时间、和正在登陆的用户数：

  ````java
  18:25:06 // 当前时间
  up 584 days,  3:25  // 系统运行时间
  1 user // 正在登陆的用户数
  ````
而最后三个数字呢，依次则是过去 1 分钟、5 分钟、15 分钟的平均负载（Load Average）。

平均负载最理想的情况是等于 CPU 个数。所以在评判平均负载时，首先你要知道系统有几个 CPU，这可以通过 top 命令或者从文件 /proc/cpuinfo 中读取，比如：

  ````bash
  grep 'model name' /proc/cpuinfo | wc -l
  16
  ````

有了 CPU 个数，我们就可以判断出，当平均负载比 CPU 个数还大的时候，系统已经出现了过载。
前面说到的 CPU 的三个负载时间段也是这个道理。

- 如果 1 分钟、5 分钟、15 分钟的三个值基本相同，或者相差不大，那就说明系统负载很平稳。
- 但如果 1 分钟的值远小于 15 分钟的值，就说明系统最近 1 分钟的负载在减少，而过去 15 分钟内却有很大的负载。
- 反过来，如果 1 分钟的值远大于 15 分钟的值，就说明最近 1 分钟的负载在增加，这种增加有可能只是临时性的，也有可能还会持续增加下去，所以就需要持续观察。一旦 1 分钟的平均负载接近或超过了 CPU 的个数，就意味着系统正在发生过载的问题，这时就得分析调查是哪里导致的问题，并要想办法优化了。

CPU 使用率，是单位时间内 CPU 繁忙情况的统计，跟平均负载并不一定完全对应。比如：

- CPU 密集型进程，使用大量 CPU 会导致平均负载升高，此时这两者是一致的；
- I/O 密集型进程，等待 I/O 也会导致平均负载升高，但 CPU 使用率不一定很高；
- 大量等待 CPU 的进程调度也会导致平均负载升高，此时的 CPU 使用率也会比较高。

##### 平均负载案例分析

预先安装 stress 和 sysstat 包，如 

````bash
apt install stress sysstat
````

查看cpu核数: 

````bash
lscpu
// 或者
grep 'model name' /proc/cpuinfo | wc -l
````

查看显示平均负载变化：

````bash
// -d会高亮显示变化的区域
watch -d uptime
````

压测命令：

````bash
// strees: 压测命令，--cpu cpu压测选项，-i io压测选项，-c 进程数压测选项，--timeout 执行时间
// 模拟  I/O  压力，即不停地执行  sync：
stress -i 1 --timeout 600
// 模拟一个  CPU  使用率 100% 的场景：
stress --cpu 1 --timeout 600
// 模拟的是 8 个进程
stress -c 8 --timeout 600
````

mpstat 是一个常用的多核 CPU 性能分析工具，用来实时查看每个 CPU 的性能指标，以及所有 CPU 的平均指标。

使用mpstat可以查看CPU 的使用率和iowait，从而可以判断出平均负载升高是因为CPU使用率过高还是因为IO密集或者因为大量进程

````bash

# -P ALL 表示监控所有CPU，后面数字5表示间隔5秒后输出一组数据
$ mpstat -P ALL 5
Linux 4.15.0 (ubuntu) 09/22/18 _x86_64_ (2 CPU)
13:30:06     CPU    %usr   %nice    %sys %iowait    %irq   %soft  %steal  %guest  %gnice   %idle
13:30:11     all   50.05    0.00    0.00    0.00    0.00    0.00    0.00    0.00    0.00   49.95
13:30:11       0    0.00    0.00    0.00    0.00    0.00    0.00    0.00    0.00    0.00  100.00
13:30:11       1  100.00    0.00    0.00    0.00    0.00    0.00    0.00    0.00    0.00    0.00
````

pidstat 是一个常用的进程性能分析工具，用来实时查看进程的 CPU、内存、I/O 以及上下文切换等性能指标。

通过pidstat可以看出是哪个进程导致的平均负载升高。

````bash

# 间隔5秒后输出一组数据，-u表示CPU指标
$ pidstat -u 5 1
Linux 4.15.0 (ubuntu)     09/22/18     _x86_64_    (2 CPU)
13:42:08      UID       PID    %usr %system  %guest   %wait    %CPU   CPU  Command
13:42:13        0       104    0.00    3.39    0.00    0.00    3.39     1  kworker/1:1H
13:42:13        0       109    0.00    0.40    0.00    0.00    0.40     0  kworker/0:1H
13:42:13        0      2997    2.00   35.53    0.00    3.99   37.52     1  stress
13:42:13        0      3057    0.00    0.40    0.00    0.00    0.40     0  pidstat
````

1. CPU密集型进程case：
   `mpstat -P ALL 5`: 观察是否有某个cpu的%usr会很高，但iowait应很低
   `pidstat -u 5 1`：每5秒输出一组数据，观察哪个进程%cpu很高，但是%wait很低，极有可能就是这个进程导致cpu飚高
2. IO密集型进程case：
   `mpstat -P ALL 5`: 观察是否有某个cpu的%iowait很高，同时%usr也较高
   `pidstat -u 5 1`：观察哪个进程%wait较高，同时%CPU也较高
3. 大量进程case：
   `pidstat -u 5 1`：观察那些%wait较高的进程是否有很多

##### 2  上下文切换问题排查

在每个任务运行前，CPU 都需要知道任务从哪里加载、又从哪里开始运行，也就是说，需要系统事先帮它设置好 CPU 寄存器和程序计数器（Program Counter，PC）。CPU 寄存器，是 CPU 内置的容量小、但速度极快的内存。而程序计数器，则是用来存储 CPU 正在执行的指令位置、或者即将执行的下一条指令位置。它们都是 CPU 在运行任何任务前，必须的依赖环境，因此也被叫做 CPU 上下文。

过多的上下文切换，会把 CPU 时间消耗在寄存器、内核栈以及虚拟内存等数据的保存和恢复上，从而缩短进程真正运行的时间，导致系统的整体性能大幅下降。

##### 上下文切换案例分析

1.启动压测，模拟系统多线程调度的瓶颈：

````bash
以10个线程运行5分钟的基准测试，模拟多线程切换的问题
$ sysbench --threads=10 --max-time=300 threads run
````

2.查看上下文切换情况：

`vmstat` 是一个常用的系统性能分析工具，主要用来分析系统的内存使用情况，也常用来分析 CPU 上下文切换和中断的次数。

````bash
# 每隔1秒输出1组数据（需要Ctrl+C才结束）
$ vmstat 1
procs -----------memory---------- ---swap-- -----io---- -system-- ------cpu-----
 r  b   swpd   free   buff  cache   si   so    bi    bo   in   cs us sy id wa st
 6  0      0 6487428 118240 1292772    0    0     0     0 9019 1398830 16 84  0  0  0
 8  0      0 6487428 118240 1292772    0    0     0     0 10191 1392312 16 84  0  0  0
````

- r（Running or Runnable）是就绪队列的长度，也就是正在运行和等待 CPU 的进程数。就绪队列的长度已经到了 8，远远超过了系统 CPU 的个数 2，所以肯定会有大量的 CPU 竞争。
- b（Blocked）则是处于不可中断睡眠状态的进程数。
- cs（context switch）是每秒上下文切换的次数。上下文切换次数上升到了 139 万。
- in（interrupt）则是每秒中断的次数。中断次数也上升到了 1 万左右，说明中断处理也是个潜在的问题。
- us（user）和 sy（system）列：这两列的 CPU 使用率加起来上升到了 100%，其中系统 CPU 使用率，也就是 sy 列高达 84%，说明 CPU 主要是被内核占用了。

`pidstat`  ,给它加上 `-w` 选项，你就可以查看每个进程上下文切换的情况了。

````bash
# 每隔1秒输出1组数据（需要 Ctrl+C 才结束）
# -w参数表示输出进程切换指标，而-u参数则表示输出CPU使用指标
$ pidstat -w -u 1
08:06:33      UID       PID    %usr %system  %guest   %wait    %CPU   CPU  Command
08:06:34        0     10488   30.00  100.00    0.00    0.00  100.00     0  sysbench
08:06:34        0     26326    0.00    1.00    0.00    0.00    1.00     0  kworker/u4:2

08:06:33      UID       PID   cswch/s nvcswch/s  Command
08:06:34        0         8     11.00      0.00  rcu_sched
08:06:34        0        16      1.00      0.00  ksoftirqd/1
08:06:34        0       471      1.00      0.00  hv_balloon
08:06:34        0      1230      1.00      0.00  iscsid
08:06:34        0      4089      1.00      0.00  kworker/1:5
08:06:34        0      4333      1.00      0.00  kworker/0:3
08:06:34        0     10499      1.00    224.00  pidstat
08:06:34        0     26326    236.00      0.00  kworker/u4:2
08:06:34     1000     26784    223.00      0.00  sshd
````

- `cswch`  ，表示每秒自愿上下文切换（voluntary context switches）的次数，是指进程无法获取所需资源，导致的上下文切换。比如说， I/O、内存等系统资源不足时，就会发生自愿上下文切换。
- `nvcswch`  ，表示每秒非自愿上下文切换（non voluntary context switches）的次数。则是指进程由于时间片已到等原因，被系统强制调度，进而发生的上下文切换。比如说，大量进程都在争抢 CPU 时，就容易发生非自愿上下文切换。

从 pidstat 的输出你可以发现，CPU 使用率的升高果然是 sysbench 导致的，它的 CPU 使用率已经达到了 100%。但上下文切换则是来自其他进程，包括非自愿上下文切换频率最高的 pidstat  ，以及自愿上下文切换频率最高的内核线程 kworker 和 sshd。

````bash
# 每隔1秒输出一组数据（需要 Ctrl+C 才结束）
# -wt 参数表示输出线程的上下文切换指标
$ pidstat -wt 1
08:14:05      UID      TGID       TID   cswch/s nvcswch/s  Command
...
08:14:05        0     10551         -      6.00      0.00  sysbench
08:14:05        0         -     10551      6.00      0.00  |__sysbench
08:14:05        0         -     10552  18911.00 103740.00  |__sysbench
08:14:05        0         -     10553  18915.00 100955.00  |__sysbench
08:14:05        0         -     10554  18827.00 103954.00  |__sysbench
...
````

虽然 sysbench 进程（也就是主线程）的上下文切换次数看起来并不多，但它的子线程的上下文切换次数却有很多。看来，上下文切换罪魁祸首，还是过多的 sysbench 线程。

从 `/proc/interrupts` 这个只读文件中读取。/proc 实际上是 Linux 的一个虚拟文件系统，用于内核空间与用户空间之间的通信。/proc/interrupts 就是这种通信机制的一部分，提供了一个只读的中断使用情况。

````bash
# -d 参数表示高亮显示变化的区域
$ watch -d cat /proc/interrupts
           CPU0       CPU1
...
RES:    2450431    5279697   Rescheduling interrupts
...
````

变化速度最快的是重调度中断（RES），这个中断类型表示，唤醒空闲状态的 CPU 来调度新的任务运行。这是多处理器系统（SMP）中，调度器用来分散任务到不同 CPU 的机制，通常也被称为处理器间中断（Inter-Processor Interrupts，IPI）。

所以，这里的中断升高还是因为过多任务的调度问题，跟前面上下文切换次数的分析结果是一致的。

根据上下文切换的类型，具体分析问题：

- 自愿上下文切换变多了，说明进程都在等待资源，有可能发生了 I/O 等其他问题；
- 非自愿上下文切换变多了，说明进程都在被强制调度，也就是都在争抢 CPU，说明 CPU 的确成了瓶颈；
- 中断次数变多了，说明 CPU 被中断处理程序占用，还需要通过查看 /proc/interrupts 文件来分析具体的中断类型。