+++
date = '2026-04-13T20:57:14+08:00'
draft = false
title = '当CPU飙高时，SRE的排查清单——USE Method实战'
+++



\## 背景



作为 SRE，接到 CPU 告警是家常便饭。但很多工程师的排查方式是靠直觉——"感觉是 CPU 问题就看 CPU，感觉是内存问题就看内存"，容易遗漏，也没有系统性。



本文介绍 Brendan Gregg 提出的 \*\*USE Method\*\*，并通过一个真实的 CPU 压力场景，演示如何用这套方法论系统化地定位问题根因。



\---



\## 什么是 USE Method



USE Method 是一套性能排查方法论，核心思想是：\*\*对系统里每一个资源，都检查三个维度\*\*。



\- \*\*U\*\*tilization（利用率）：资源被使用了多少，比如 CPU 使用率 30%

\- \*\*S\*\*aturation（饱和度）：资源是否过载，有没有请求在排队，比如 CPU 运行队列长度

\- \*\*E\*\*rrors（错误）：资源有没有报错，比如内核错误日志、网卡丢包



资源包括：CPU、内存、磁盘、网络等。



这套方法的价值在于——\*\*不靠直觉，靠系统化的检查清单，确保不遗漏任何资源\*\*。



\---



\## 故障现场



本文使用一个 Go 编写的 demo-app，部署在 k3s 单节点集群上。通过调用 `/debug/burst?ms=500` 接口注入 CPU 压力——每次请求会触发 500ms 的 busy loop，模拟 CPU 密集型负载。



```bash

curl "http://10.0.0.187:30080/debug/burst?ms=500"

\# 输出：Burst enabled: 500 ms busy loop per request

```



然后用多个终端并发持续发请求：



```bash

while true; do curl -s "http://10.0.0.187:30080/api" > /dev/null; done

```



故障现场准备好了，开始排查。



\---



\## 第一步：发现问题（uptime）



收到告警第一件事，先用 `uptime` 快速看系统负载趋势：



```bash

uptime

\# 13:59:01 up 3:48, 4 users, load average: 1.34, 0.64, 0.35

\# 13:59:37 up 3:48, 4 users, load average: 1.39, 0.73, 0.39

```



三个数字分别是 \*\*1分钟、5分钟、15分钟\*\*的平均负载。从 0.35 涨到 1.39，趋势明显向上。



用 `nproc` 确认 CPU 核数：



```bash

nproc

\# 2

```



这台机器是 2 核，load average 超过 2 就意味着过载。现在是 1.39，已经接近满负荷，而且还在上升。



\*\*结论\*\*：系统负载异常，需要进一步定位是哪个资源出了问题。



\---



\## 第二步：定位 CPU 利用率（mpstat）



`uptime` 只告诉你负载高，但不知道 CPU 具体怎么用的。用 `mpstat` 进一步分解：



```bash

mpstat 1 3

```



```

%usr   %sys  %iowait  %irq  %soft  %idle

25.77   1.03     0.00  3.09   0.00  70.10

24.87   1.04     0.00  3.11   0.51  70.98

Average: 30.86  0.86   0.00  3.10   0.34  64.83

```



关键列的含义：



\- \*\*%usr\*\*：用户态 CPU 消耗，应用层代码在这里跑

\- \*\*%sys\*\*：内核态 CPU 消耗，系统调用、网络处理等

\- \*\*%iowait\*\*：CPU 等待磁盘 I/O 的时间

\- \*\*%idle\*\*：CPU 空闲时间



现在 `%usr` 达到 25-30%，`%iowait` 为 0，说明\*\*瓶颈在应用层，不是磁盘 I/O\*\*。



\*\*结论（Utilization）\*\*：CPU 利用率在上升，压力来自用户态，是应用层代码在消耗 CPU。



\---



\## 第三步：检查饱和度（vmstat）



利用率高不一定是问题，在去看火焰图之前，有一个更重要的问题需要先回答：\*\*这个 CPU 利用率有没有真正影响到服务质量？\*\*



用 `vmstat` 来看：



```bash

vmstat 1 3

```



```

&#x20;r  b   us  sy  id  wa

&#x20;3  0   26   4  70   0

&#x20;4  0   25   6  69   0

```



最关键的是 \*\*r 列\*\*——运行队列长度，表示有多少个进程在等待 CPU。判断标准很简单：\*\*r 持续超过 CPU 核数就说明 CPU 已经饱和\*\*。



\*\*饱和度的真正作用是判断紧急程度\*\*，而不是帮你找根因：



举个反例：如果 `%usr` 60%，但 r=1，说明请求没有积压，每个请求都能及时处理，用户感受不到慢。这时候去优化代码，可能是在做无效的工作。



但如果 r 超过核数，说明请求已经在排队，用户响应时间一定变长了，必须立刻处理。



|Utilization|Saturation|结论|

|---|---|---|

|低|低|CPU 没压力，瓶颈不在这里|

|高|低|CPU 忙但还能处理，用户无感知，不紧急|

|低|高|异常，可能是锁竞争或调度问题，需要深挖|

|高|高|请求积压，用户已感受到慢，立刻处理|



现在 r=3-4，超过了核数 2，属于第四种情况——请求在排队，服务已经受影响，需要立刻找根因。



\*\*结论（Saturation）\*\*：CPU 已饱和，用户响应时间受影响，必须立刻处理，下一步用火焰图定位根因。



\---



\## 第四步：检查内核错误（dmesg）



已经知道了利用率和饱和度，为什么还要检查这一步？



因为有些问题利用率和饱和度完全看不出来，只有 dmesg 能发现。比如 CPU 硬件故障（MCE 错误）、OOM Killer 杀进程——这些情况下利用率和饱和度可能完全正常，但服务就是不对劲。dmesg 是一个\*\*兜底检查\*\*，成本只有10秒，但关键时刻能避免你在错误方向上浪费大量时间。



```bash

dmesg | tail -20

```



```

perf: interrupt took too long (9101 > 8927), lowering kernel.perf\_event\_max\_sample\_rate to 21000

perf: interrupt took too long (14323 > 14253), lowering kernel.perf\_event\_max\_sample\_rate to 13000

perf: interrupt took too long (35973 > 35746), lowering kernel.perf\_event\_max\_sample\_rate to 5000

```



这条日志出现了5次，perf 采样率从初始值一路降到了 5000。说明 \*\*CPU 太忙了，连内核的性能采样中断都处理不过来\*\*，被迫降低采样频率。这是 CPU 压力过高的直接副作用，侧面印证了前两步的判断。



这次没有发现硬件故障或 OOM，可以放心继续往下走。



\*\*结论（Errors）\*\*：无硬件故障，perf 采样率被压低是 CPU 高负载的副作用，结论与前两步一致，下一步直接用火焰图定位根因。



\---



\## 第五步：火焰图定位根因



USE Method 告诉我们 CPU 是瓶颈，但还不知道是哪段代码在烧 CPU。用火焰图来找根因。



用 perf 采集30秒 CPU 样本：



```bash

perf record -F 99 -a -g -- sleep 30

```



参数说明：



\- `-F 99`：每秒采样99次

\- `-a`：采集所有 CPU

\- `-g`：记录完整调用栈



然后生成火焰图：



```bash

perf script > /tmp/perf.out

cat /tmp/perf.out | /opt/FlameGraph/stackcollapse-perf.pl | /opt/FlameGraph/flamegraph.pl > /tmp/flamegraph.svg

```



用浏览器打开 SVG 文件：



!\[\[Pasted image 20260413201508.png]]



\*\*怎么看火焰图\*\*：



\- 从下往上是调用链，下面的函数调用了上面的函数

\- 横向宽度代表 CPU 时间占比，\*\*越宽越耗 CPU\*\*

\- 顶部最宽的函数就是真正的性能热点



看左边最宽的那一块，调用链从下往上：



```

runtime.goexit.abi0         ← Go 程序入口

net/http.(\*Server).Serve    ← HTTP 服务器监听

net/http.(\*conn).serve      ← 处理 HTTP 连接

net/http.serverHandler      ← 分发请求

main.WithMetrics.func1      ← 监控中间件

main.apiHandler             ← /api handler

main.busyLoop               ← ← 热点在这里

math/rand.Intn              ← busyLoop 在反复调用随机数

```



`main.busyLoop` 这一行最宽，它上面是 `math/rand.Intn`，说明 busyLoop 把大部分 CPU 时间都花在反复计算随机数上了。



\*\*结论\*\*：CPU 热点是 `main.busyLoop`，根因找到。



\---



\## 总结



|步骤|命令|对应 USE|发现|

|---|---|---|---|

|发现问题|`uptime`|—|load average 从 0.35 涨到 1.39，趋势向上|

|利用率|`mpstat 1 3`|Utilization|%usr 25-30%，压力来自用户态|

|饱和度|`vmstat 1 3`|Saturation|运行队列 r=3-4，超过核数，已饱和|

|错误|`dmesg`|Errors|perf 采样率被压低，CPU 响应变慢|

|根因|火焰图|—|`main.busyLoop` 是 CPU 热点|



USE Method 的价值在于\*\*系统化\*\*——每次排查都按照 Utilization → Saturation → Errors 的顺序检查，不靠直觉，不遗漏资源，最终用火焰图精确定位根因。

