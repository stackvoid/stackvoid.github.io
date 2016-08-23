---
layout: post
title: Comparable vs Comparator
categories: [Java]
tags: [Collection]
---


Comparable 和 Comparator 是Java Core API提供的两个比较接口。实现这两个接口，可以方便我们
对相应数组和集合中的对象排序。但二者的优缺点差别是什么呢？本文的例子或许能解决这个问题。

### Comparable

一个类通过实现Comparable接口获得与其他此类对象比较的能力。compareTo()方法是必须要实现的。
下面来演示如何使用Comparable接口。Comparable 接口只有一个方法 compareTo(Object obj)
其中
- this < obj 返回负
- this = obj 返回 0
- this > obj 返回正
即将当前这个对象与指定的对象进行顺序比较，当该对象小于、等于或大于指定对象时，
分别返回一个负整数、 0 或正整数，如果无法进行比较，则抛出ClassCastException 异常。

{% highlight java %} 
class HDTV implements Comparable<HDTV> {
	private int size;
	private String brand;
	
	public HDTV(int size, String brand){
		this.size = size;
		this.brand = brand;
	}
	public int getSize(){
		return size;
	}
	public String getBrand(){
		return brand;
	}
	
	@Override
	public int compareTo(HDTV tv){
	/*为了程序健壮，记住不要使用 
	**return this.getSize() - tv.getSize()
	** 减法可能可能会溢出，尽管发生概率很小
	**或使用 return (i2 < i1 ? -1 : (i2 == i1 ? 0 :1))这种表示方法
	*/
		if (this.getSize() > tv.getSize())
				return 1;
		else if (this.getSize() < tv.getSize())
				return -1;
		else 
				return 0;
	}
	public class Main {
	public static void main(String[] args) {
		HDTV tv1 = new HDTV(55, "Samsung");
		HDTV tv2 = new HDTV(60, "Sony");
 
		if (tv1.compareTo(tv2) > 0) {
			System.out.println(tv1.getBrand() + " is better.");
		} else {
			System.out.println(tv2.getBrand() + " is better.");
		}
	}
}

{%endhighlight%} 

> Sony is better.

### Comparator

Comparator比较器是用一个新类实现Comparator接口，实现接口里的方法compare(Object o1, Object o2).
如果已有的类没有实现Comparable接口，而我们需要按类的某一属性排序，自定义的Comparator完成了“类外”排序任务。
下面一个例子演示了Comparator的用法。

{% highlight java %} 
import java.util.ArrayList;
import java.util.Collections;
import java.util.Comparator;
 
class HDTV {
	private int size;
	private String brand;
 
	public HDTV(int size, String brand) {
		this.size = size;
		this.brand = brand;
	}
 
	public int getSize() {
		return size;
	}
 
	public void setSize(int size) {
		this.size = size;
	}
 
	public String getBrand() {
		return brand;
	}
 
	public void setBrand(String brand) {
		this.brand = brand;
	}
}
 
class SizeComparator implements Comparator<HDTV> {
	@Override
	public int compare(HDTV tv1, HDTV tv2) {
		int tv1Size = tv1.getSize();
		int tv2Size = tv2.getSize();
 
		if (tv1Size > tv2Size) {
			return 1;
		} else if (tv1Size < tv2Size) {
			return -1;
		} else {
			return 0;
		}
	}
}
 
public class Main {
	public static void main(String[] args) {
		HDTV tv1 = new HDTV(55, "Samsung");
		HDTV tv2 = new HDTV(60, "Sony");
		HDTV tv3 = new HDTV(42, "Panasonic");
 
		ArrayList<HDTV> al = new ArrayList<HDTV>();
		al.add(tv1);
		al.add(tv2);
		al.add(tv3);
 
		Collections.sort(al, new SizeComparator());
		for (HDTV a : al) {
			System.out.println(a.getBrand());
		}
	}
}
{%endhighlight%} 

### 选择Comparable还是Comparator

如果一个类设计的很清晰有一种排序关系，可以实现Comparable接口；若不清晰，则等到需要的时候实现
Comparator，也避免使用Comparable引入了类的复杂性。

 

### 二者区别

下面来说说Comparable接口和Comparator接口的区别 

- Comparator位于包java.util下，而Comparable位于包 java.lang下 

- Comparable 是一个对象本身就已经支持自比较所需要实现的接口（如 String、Integer 自己就可以完成比较大小操作，已经实现了Comparable接口）  此接口强行对实现它的每个类的对象进行整体排序。这种排序被称为类的自然排序，类的 compareTo 方法被称为它的自然比较方法。 

- Comparator接口使用更加灵活。也是策略模式的一个体现。
