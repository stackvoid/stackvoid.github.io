---
layout: post
title: 随机数浅析-伪随机数
categories: [Utils]
tags: [Principle]
---

> Any one who considers orithmetical methods of producing random digits is,of course,in a state of sin.

> JOHN VON NEUMANN(1951)

<Br />
冯诺依曼告诫我们，不要相信任何伪随机数。


##基础知识

通俗的讲，随机数就是一个确定分布的独立的随机数序列，即每个序列里的数出现都只是偶然的，并且与这序列中的其它数无关，每个数落入此序列任何位置的概率都是相等的。
在这个随机数序列中，0-9 这10个数字，每个数字出现的概率都为 1/10。

伪随机数列，如果一个序列，一方面它是可以预先确定的，并且是可以重复地生产和复制的；一方面它又具有某种随机序列的随机特性（即统计特性），我们便称这种序列为伪随机序列。

随机数的历史可以参考 TAOCPP 第二卷第三章。

##线性同余随机数生成器

LCG(linear congruential generator)几乎是最好最朴素的伪随机数产生器算法。比较简单，容易实现，产数效率较高。 

LCG 算法数学上基于公式：

X(n+1) = (a * X(n) + c) % m

**其中，各系数为：**[^11]

模m, m > 0
系数a, 0 < a < m
增量c, 0 <= c < m
原始值(种子) 0 <= X(0) < m


其中参数c, m, a比较敏感，或者说直接影响了伪随机数产生的质量。
**一般而言，高LCG的m是2的指数次幂(一般2^32或者2^64)，因为这样取模操作截断最右的32或64位就可以了。多数编译器的库中使用了该理论实现其伪随机数发生器rand()。**[^22]

##伪随机数在Java中实现

Java 中的伪随机数原理就是线性同余。

不管 Math.random() 还是 Random.nextInt() 等，最终调用的都是类 Random 中的 next 方法。

代码比较简单，不在展开论述。

{%highlight java linenos%}

    protected int next(int bits) {
        long oldseed, nextseed;
        AtomicLong seed = this.seed;
        do {
            oldseed = seed.get();
			//multiplier、addend、mask是魔数，不同语言值都不太一样
			
            nextseed = (oldseed * multiplier + addend) & mask;
        } while (!seed.compareAndSet(oldseed, nextseed));//保证顺序操作
        return (int)(nextseed >>> (48 - bits));//返回的位数，按照bits进行斩断
    }
    
{%endhighlight%}

[1]:详情请见 the art of computer programming volume 2  3.2.1节
[2]:http://www.cnblogs.com/xkfz007/archive/2012/03/27/2420154.html