---
layout: post
title: logcat 中关于虚拟机 GC 的日志
categories: [Android]
tags: [GC]
---

在调试 Android 程序的时候，经常看到下面的日志，本文分析这些日志的意思。

 D/dalvikvm( 3604): GC_CONCURRENT freed 1755K, 30% free 4589K/6476K, paused 2ms+14ms, total 69ms

 D/dalvikvm( 1173): GC_EXPLICIT freed 43K, 5% free 2773K/2904K, paused 5ms+3ms, total 47ms

 D/dalvikvm( 4370): GC_FOR_ALLOC freed 155K, 18% free 3982K/4812K, paused 60ms, total 61ms
 
 其含义是 
 
{%highlight java%}
 
 D/dalvikvm: <GC_Reason> <Amount_freed>, <Heap_stats>, <Pause_time>, <Total_time>

{%endhighlight%}
 
 其中 **GC_Reason** 又包括：
 
- GC_FOR_MALLOC：应用尝试申请内存时，虚拟机发现堆中内存不足以这一次分配，引发 GC。
- GC_CONCURRENT：当堆中对象大于一定阈值，GC 会自动触发回收对象。
- GC_EXPLICIT：程序主动调用 GC，如使用 Runtime.gc(),VMRuntime.gc(),或 SIGUSR1。
- GC_EXTERNAL_ALLOC：外部内存分配失败时触发 GC，回收内存以供外部内存使用(例如 Bitmap、Cursor)。

 
**Amount_freed**：本次 GC 所释放的内存。

**Heap_stats**：当前未被使用内存占总内存的百分比。

**Pause_time**：应用由于 GC 被暂停的时间(Stop The World)。

**Total_time**：本次发生 GC 花费的总时间。

以上的 GC 日志都是针对 Dalvikvm 虚拟机而言，而到了 **Android 5.0**，使用的是 **ART** 虚拟机，下面我们来看看 ART 虚拟机打印出来的 log。

I/art     (  444): Background partial concurrent mark sweep GC freed 32873(2MB) AllocSpace objects, 9(4MB) LOS objects, 33% free, 16MB/24MB, paused 3.160ms total 236.699ms

I/art     (  444): Explicit concurrent mark sweep GC freed 34931(2MB) AllocSpace objects, 3(721KB) LOS objects, 33% free, 14MB/22MB, paused 1.927ms total 118.311ms

其含义是：

{%highlight java%}

I/art: <GC_Reason> <Amount_freed>, <LOS_Space_Status>, <Heap_stats>, <Pause_time>, <Total_time>

{%endhighlight%}

**GC_Reason**

{%highlight java linenos%}
enum GcCause {
  // GC triggered by a failed allocation. Thread doing allocation is blocked waiting for GC before
  // retrying allocation.
  kGcCauseForAlloc,
  // A background GC trying to ensure there is free memory ahead of allocations.
  kGcCauseBackground,
  // An explicit System.gc() call.
  kGcCauseExplicit,
  // GC triggered for a native allocation.
  kGcCauseForNativeAlloc,
  // GC triggered for a collector transition.
  kGcCauseCollectorTransition,
  // Not a real GC cause, used when we disable moving GC (currently for GetPrimitiveArrayCritical).
  kGcCauseDisableMovingGc,
  // Not a real GC cause, used when we trim the heap.
  kGcCauseTrim,
  // GC triggered for background transition when both foreground and background collector are CMS.
  kGcCauseHomogeneousSpaceCompact,
};
{%endhighlight%}

**LOS_Space_Status**：Large Object Space，大对象占用的空间，这部分内存并不是分配在堆上的，但仍属于应用程序内存空间，主要用来管理 bitmap 等占内存大的对象，避免因分配大内存导致堆频繁 GC。

**Heap_stats**：堆的当前状态信息，与 Dalvikvm 相同。

**Pause_time、Total_time**：参考 Dalvikvm。