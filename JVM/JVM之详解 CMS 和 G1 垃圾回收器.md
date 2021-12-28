## CMS 垃圾回收器

### 概述

CMS 全称为 Concurrent Mark Sweep，意为并发标记清除，使用的是标记清除法。主要关注系统停顿时间（Stop The World）。

### JVM 参数

*-XX:+UseConcMarkSweepGC*：设置老年代使用该回收器。

*-XX:ConcGCThreads*：设置并发线程数量。

CMS 是并发的垃圾回收器，也就说CMS 回收的过程中，应用程序仍然在不停的工作，又会有新的垃圾不断的产生。CMS 不会等到应用程序饱和的时候才去回收垃圾，而是在某一阀值的时候开始回收，即 *-XX:CMSInitiatingoccupancyFraction* ，默认为 68，也就是说当老年代的空间使用率达到 68% 的时候，就会执行 CMS 垃圾回收。

如果内存使用率增长的很快，在 CMS 执行的过程中，已经出现了内存不足的情况，此时 CMS 回收就会失败，虚拟机会降级采用 SerialOld 串行回收器进行垃圾回收，这会导致应用程序中断，直到垃圾回收完成后才会正常工作。这个过程 GC 的停顿时间可能较长，所以 *-XX:CMSInitiatingoccupancyFraction* 的设置要根据实际的情况。

CMS 采用标记清除算法进行垃圾回收，而标记清除算法会导致内存碎片的问题。在 CMS 中可以设置 *-XX:+UseCMSCompactAtFullCollecion* ，使 CMS 回收完成之后进行一次碎片整理。

### GC 过程



## G1 垃圾回收器

### 概述



### JVM 参数



### GC 过程