---
layout: post
title: Android ListView 优化最佳实践
categories: [Android]
tags: [Optimization]
---

我[有篇博客](http://stackvoid.com/using-adapter-in-efficiency-way/)教大家如何利用 convertView 以及 viewHolder(static) 改善 ListView 卡顿情况；但是在 ListView 加载大量复杂布局和图片的时候，即使使用了 convertView 和 viewHolder，ListView还是卡顿，本文主要讨论了如何在加载复杂 list_item 同时保证 ListView 流畅性。

核心思想是

> 监听滑动据加载，异步加载数据。

> getView 函数一定不能耗时，有耗时任务要异步加载。


主要的方法：

1. 先判断当前 ListView 的状态，只有 ListView 停止滑动才开启新线程加载数据，其他状态均忽略。

1. 使用 getFirstVisiblePosition 和 getLastVisiblePosition 方法来显示 item。

1. 耗时任务一定不要在 getView 方法中进行，最好异步进行。

具体代码如下：

{%highlight java linenos%}
//1. 判断listView状态
AbsListView.OnScrollListener onScrollListener = new AbsListView.OnScrollListener() {// ListView
// 触摸事件

public void onScroll(AbsListView view, int firstVisibleItem, int visibleItemCount, int totalItemCount) {
}

public void onScrollStateChanged(AbsListView view, int scrollState) {
switch (scrollState) {
  case AbsListView.OnScrollListener.SCROLL_STATE_FLING:// 滑动状态
  threadFlag = false;
  break;
  case AbsListView.OnScrollListener.SCROLL_STATE_IDLE:// 停止
  threadFlag = true;
  startThread();//开启新线程，加载数据
  break;
  case AbsListView.OnScrollListener.SCROLL_STATE_TOUCH_SCROLL:// 触摸listView
  threadFlag = false;
  break;
  default:
  // Toast.makeText(contextt, "default",
  // Toast.LENGTH_SHORT).show();
  break;
  }
}
};
{%endhighlight%}

相信做到以上三点，就能运用自如的使用 ListView了，O(∩_∩)O哈哈~
