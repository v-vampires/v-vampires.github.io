title: G1垃圾收集器学习
author: v-vampires
tags: []
categories:
  - JVM
date: 2020-01-03 13:28:00
---
## 背景
之前的项目因为使用G1垃圾收集器，运行上几天后，发现有时候接口的响应时间出现抖动，导致调用方发生了超时，通过监控发现在超时的时候，系统的YongGC平均时间由原来的几十毫秒居然上涨到了几秒钟。于是趁着这个机会，学习了一下GC，并针对这个问题进行了一下简单的优化
## 介绍
G1是一种并发、并行、部分Stop The World、使用Copying算法收集的分代的增量式收集器。并发指的是在标记阶段，可以和应用程序并发执行，减少对应用的暂停时间。并行则是在清理阶段使用多线程处理。增量收集是指G1将堆划分为很多大小相等的Region，每次收集会判断各个Region的活性（垃圾对象的占比），然后G1按照设置的停顿时间、和前几次回收所用时间来估算要回收哪些Region，所以增量的意思就是回收收益较高的一部分Region，并不是全部Region
## 优势
1. 可预测的停顿时间
2. 不会存在内存碎片

## 实现方式

## 收集分类
### Young GC
当Eden区满了，就会进行Young GC，Young GC会将Eden和Survivor区对应的Regin（称为Collection Set, CSet）中的活对象copy到新的Region（即Survivor）中，当对象的年龄达到阈值后，会copy到old Region中，由于是采用的copy算法，就避免了内存碎片问题，不在需要单独压缩
### Mixed GC
当整个堆的使用率超过InitiatingHeapOccupancyPercent置顶值之后，就会开始ConcurentMarking, 完成了Concurrent Marking后，G1会从Young GC切换到Mixed GC, 在Mixed GC中，G1可以增加若干个Old区域的Region到CSet中
### Full GC
由于G1的一些收集过程是和应用程序并发执行的，所以可能还没有回收完，新申请的内存空间占满了所有空间，就会触发Full GC。（正常情况，不应该有Full GC）
## 常见的调节参数
```
-Xms12g -Xmx12g
```
设置堆大小
```
-XX:MaxGCPauseMillis=200
```
设置GC最大停顿时间
```
-XX:G1HeapRegionSize=8m
```
设置Region大小
```
-XX:InitiatingHeapOccupancyPercent
=30
```
开始一个标记周期的堆占用比例阈值，默认45%，注意这里是整个堆,当达到比例之后开始Mixed GC

而这次线上问题的调优正式根据gc日志发现YongGC的时间主要花费在Scan Rset上（如下图），考虑到可能是因为Region块太小导致Region的数量过多，而region的数量过多则会增加过多的跨区域引用，导致扫描的时间长，因此将-XX:G1HeapRegionSize由原来的2m调整到8m；-XX:InitiatingHeapOccupancyPercent则由原来的45%调整到30%，即相当于增加了MixedGC的频率。运行了一段时间之后发现整体系统的GC时间比较稳定，接口也没有再继续出现抖动

![upload successful](/images/pasted-1.png)
## 日志分析
```
2019-12-20T16:41:01.932+0800: 3.614: [GC concurrent-string-deduplication, 35.2K->1184.0B(34.1K), avg 96.7%, 0.0001139 secs]
//GC前打印heap使用信息
{Heap before GC invocations=2 (full 0):
 garbage-first heap   total 11534336K, used 580940K [0x0000000500000000, 0x0000000500802c00, 0x00000007c0000000)
  region size 8192K, 70 young (573440K), 9 survivors (73728K)
 Metaspace       used 14708K, capacity 14988K, committed 15360K, reserved 1062912K
  class space    used 1625K, capacity 1691K, committed 1792K, reserved 1048576K
2019-12-20T16:41:03.046+0800: 4.729: [GC pause (G1 Evacuation Pause) (young)
Desired survivor size 37748736 bytes, new threshold 15 (max 15)
- age   1:    6261936 bytes,    6261936 total
- age   2:   16888272 bytes,   23150208 total
//GC选择要收集的Region
 4.729: [G1Ergonomics (CSet Construction) start choosing CSet, _pending_cards: 1024, predicted base time: 35.60 ms, remaining time: 164.40 ms, target pause time: 200.00 ms]
 4.729: [G1Ergonomics (CSet Construction) add young regions to CSet, eden: 61 regions, survivors: 9 regions, predicted young region time: 223.97 ms]
 4.729: [G1Ergonomics (CSet Construction) finish choosing CSet, eden: 61 regions, survivors: 9 regions, old: 0 regions, predicted pause time: 259.58 ms, target pause time: 200.00 ms]
, 0.0498341 secs]
   [Parallel Time: 19.1 ms, GC Workers: 8]
      [GC Worker Start (ms): Min: 4729.1, Avg: 4739.0, Max: 4748.1, Diff: 19.0]
      [Ext Root Scanning (ms): Min: 0.0, Avg: 0.3, Max: 1.1, Diff: 1.1, Sum: 2.0]
      [Update RS (ms): Min: 0.0, Avg: 0.1, Max: 0.4, Diff: 0.4, Sum: 0.8]
         [Processed Buffers: Min: 0, Avg: 0.6, Max: 2, Diff: 2, Sum: 5]
      [Scan RS (ms): Min: 0.0, Avg: 0.0, Max: 0.0, Diff: 0.0, Sum: 0.1]
      [Code Root Scanning (ms): Min: 0.0, Avg: 0.3, Max: 1.2, Diff: 1.2, Sum: 2.4]
      [Object Copy (ms): Min: 0.0, Avg: 8.3, Max: 17.4, Diff: 17.4, Sum: 66.5]
      ...
   //回收情况
   [Eden: 488.0M(488.0M)->0.0B(488.0M) Survivors: 72.0M->72.0M Heap: 567.3M(11.0G)->97.2M(11.0G)]
//GC之后heap使用信息
Heap after GC invocations=3 (full 0):
 garbage-first heap   total 11534336K, used 99533K [0x0000000500000000, 0x0000000500802c00, 0x00000007c0000000)
  region size 8192K, 9 young (73728K), 9 survivors (73728K)
 Metaspace       used 14708K, capacity 14988K, committed 15360K, reserved 1062912K
  class space    used 1625K, capacity 1691K, committed 1792K, reserved 1048576K
}
 [Times: user=0.10 sys=0.01, real=0.05 secs]
 ...
 ...
 54243.724: [G1Ergonomics (CSet Construction) start choosing CSet, _pending_cards: 31754, predicted base time: 39.45 ms, remaining time: 160.55 ms, target pause time: 200.00 ms]
 54243.724: [G1Ergonomics (CSet Construction) add young regions to CSet, eden: 829 regions, survivors: 8 regions, predicted young region time: 23.89 ms]
 54243.725: [G1Ergonomics (CSet Construction) finish choosing CSet, eden: 829 regions, survivors: 8 regions, old: 0 regions, predicted pause time: 63.33 ms, target pause time: 200.00 ms]
//请求并发标记周期初始化，原因是堆使用占比超过了阈值
 54243.788: [G1Ergonomics (Concurrent Cycles) request concurrent cycle initiation, reason: occupancy higher than threshold, occupancy: 3548381184 bytes, allocation request: 0 bytes, thres
hold: 3543348000 bytes (30.00 %), source: end of GC]
, 0.0644085 secs]
   [Parallel Time: 52.2 ms, GC Workers: 8]
...
...
//开始并发周期
 54444.747: [G1Ergonomics (Concurrent Cycles) initiate concurrent cycle, reason: concurrent cycle initiation requested]
//Initial Marking Phase
2019-12-21T07:48:23.065+0800: 54444.748: [GC pause (G1 Evacuation Pause) (young) (initial-mark)
...
...
//Root Region Scan Phase
 2019-12-21T07:48:23.127+0800: 54444.810: [GC concurrent-root-region-scan-start]
2019-12-21T07:48:23.127+0800: 54444.810: [GC concurrent-string-deduplication, 149.5K->146.6K(3000.0B), avg 19.7%, 0.0003013 secs]
2019-12-21T07:48:23.168+0800: 54444.851: [GC concurrent-root-region-scan-end, 0.0406415 secs]
//Concurrent Marking Phase
2019-12-21T07:48:23.168+0800: 54444.851: [GC concurrent-mark-start]
2019-12-21T07:48:24.707+0800: 54446.389: [GC concurrent-mark-end, 1.5387352 secs]
//Remark Phase
2019-12-21T07:48:24.709+0800: 54446.392: [GC remark 2019-12-21T07:48:24.709+0800: 54446.392: [Finalize Marking, 0.0007364 secs] 2019-12-21T07:48:24.710+0800: 54446.393: [GC ref-proc, 0.02
//Cleanup Phase
61782 secs] 2019-12-21T07:48:24.736+0800: 54446.419: [Unloading, 0.0355887 secs], 0.0668872 secs]
 [Times: user=0.29 sys=0.01, real=0.07 secs]
2019-12-21T07:48:24.779+0800: 54446.462: [GC cleanup 3546M->3546M(11G), 0.0294983 secs]
 [Times: user=0.22 sys=0.00, real=0.03 secs]
...
...
 54663.058: [G1Ergonomics (CSet Construction) start choosing CSet, _pending_cards: 25667, predicted base time: 37.62 ms, remaining time: 162.38 ms, target pause time: 200.00 ms]
 54663.058: [G1Ergonomics (CSet Construction) add young regions to CSet, eden: 828 regions, survivors: 8 regions, predicted young region time: 22.57 ms]
 54663.058: [G1Ergonomics (CSet Construction) finish choosing CSet, eden: 828 regions, survivors: 8 regions, old: 0 regions, predicted pause time: 60.19 ms, target pause time: 200.00 ms]
 54663.107: [G1Ergonomics (Concurrent Cycles) do not request concurrent cycle initiation, reason: still doing mixed collections, occupancy: 3556769792 bytes, allocation request: 0 bytes,
threshold: 3543348000 bytes (30.00 %), source: end of GC]
 54663.107: [G1Ergonomics (Mixed GCs) start mixed GCs, reason: candidate old regions available, candidate old regions: 357 regions, reclaimable: 1319713712 bytes (11.17 %), threshold: 5.0
0 %]
, 0.0493032 secs]
   [Parallel Time: 38.2 ms, GC Workers: 8]
...
...
 54722.968: [G1Ergonomics (CSet Construction) start choosing CSet, _pending_cards: 32090, predicted base time: 34.60 ms, remaining time: 165.40 ms, target pause time: 200.00 ms]
 54722.968: [G1Ergonomics (CSet Construction) add young regions to CSet, eden: 62 regions, survivors: 8 regions, predicted young region time: 13.33 ms]
 54722.970: [G1Ergonomics (CSet Construction) finish adding old regions to CSet, reason: reclaimable percentage not over threshold, old: 39 regions, max: 141 regions, reclaimable: 5884430
56 bytes (4.98 %), threshold: 5.00 %]
 54722.970: [G1Ergonomics (CSet Construction) added expensive regions to CSet, reason: old CSet region num not reached min, old: 39 regions, expensive: 8 regions, min: 45 regions, remaini
ng time: 0.00 ms]
 54722.970: [G1Ergonomics (CSet Construction) finish choosing CSet, eden: 62 regions, survivors: 8 regions, old: 39 regions, predicted pause time: 242.63 ms, target pause time: 200.00 ms]
//不再进行MixedGC收集，切换到YongGC收集
 54723.088: [G1Ergonomics (Mixed GCs) do not continue mixed GCs, reason: reclaimable percentage not over threshold, candidate old regions: 181 regions, reclaimable: 588443056 bytes (4.98
%), threshold: 5.00 %]
, 0.1202978 secs]
   [Parallel Time: 108.3 ms, GC Workers: 8]
...
...
 54937.113: [G1Ergonomics (CSet Construction) start choosing CSet, _pending_cards: 39863, predicted base time: 36.57 ms, remaining time: 163.43 ms, target pause time: 200.00 ms]
 54937.113: [G1Ergonomics (CSet Construction) add young regions to CSet, eden: 836 regions, survivors: 8 regions, predicted young region time: 18.81 ms]
 54937.113: [G1Ergonomics (CSet Construction) finish choosing CSet, eden: 836 regions, survivors: 8 regions, old: 0 regions, predicted pause time: 55.39 ms, target pause time: 200.00 ms]
, 0.0532435 secs]
   [Parallel Time: 40.9 ms, GC Workers: 8]
```