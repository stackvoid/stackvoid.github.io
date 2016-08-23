---
layout: post
title: Object类默认hashCode方法解析
categories: [Java]
tags: [JVM]
---

在Java中，计算对象默认的hash值的方法在[synchronizer.cpp](http://hg.openjdk.java.net/jdk7u/jdk7u/hotspot/file/74d14a44c398/src/share/vm/runtime/synchronizer.cpp)文件中。对象的hash是在第一次使用时，即首次调用hashCode方法时进行计算，并将hash值存储在对象的Mark Word中。

类java.lang.Object#hashCode() 方法是native，最终调用到ObjectSynchronizer::FastHashCode函数获取hash值，下图为获取对象hash值的基本流程。

![FastHashCode函数](http://stackvoid.qiniudn.com/2014-06-16-HashCode%20in%20Hotspot%20.png)

结合此流程图和代码注释读FastHashCode函数，应该很容易看懂，本文暂不详细讲解。

FastHashCode在计算hash值时，有两个核心计算hash值的函数，一个是get_next_hash()函数，一个是hash()函数.

synchronizer.cpp中的get_next_hash方法用于计算新的hash值。

{%highlight c%}
static inline intptr_t get_next_hash(Thread * Self, oop obj) {
  intptr_t value = 0 ;
  if (hashCode == 0) {
     value = os::random() ;
  } else
  if (hashCode == 1) {
     intptr_t addrBits = intptr_t(obj) >> 3 ;
     value = addrBits ^ (addrBits >> 5) ^ GVars.stwRandom ;
  } else
  if (hashCode == 2) {
     value = 1 ;            // for sensitivity testing
  } else
  if (hashCode == 3) {
     value = ++GVars.hcSequence ;
  } else
  if (hashCode == 4) {
     value = intptr_t(obj) ;
  } else {
     unsigned t = Self->_hashStateX ;
     t ^= (t << 11) ;
     Self->_hashStateX = Self->_hashStateY ;
     Self->_hashStateY = Self->_hashStateZ ;
     Self->_hashStateZ = Self->_hashStateW ;
     unsigned v = Self->_hashStateW ;
     v = (v ^ (v >> 19)) ^ (t ^ (t >> 8)) ;
     Self->_hashStateW = v ;
     value = v ;
  }

  value &= markOopDesc::hash_mask;
  if (value == 0) value = 0xBAD ;
  assert (value != markOopDesc::no_hash, "invariant") ;
  TEVENT (hashCode: GENERATE) ;
  return value;
}
{%endhighlight%}

get_next_hash函数会根据传给JVM的参数-XX:hashCode=n来选择使用哪种方法生成对象的hashcode：

1. hashCode=0，hash值为系统生成的随机数
1. hashCode=1，hash值为对对象地址做移位和异或操作
1. hashCode=2，所有的hash值都等于1
1. hashCode=3，hash的值为一个自增序列的值
1. hashCode=4，hash值为此对象地址
1. hashCode=others, 使用[Xorshift随机数生成器](http://en.wikipedia.org/wiki/Xorshift)，Xorshift随机数生成器总体性能非常好。[Xorshift原理](http://www.jstatsoft.org/v08/i14/paper)

另外一种是hash()函数。其先获取该对象的Mark Word，然后对Mark Word对象的的地址做位移和逻辑与操作，以结果作为hash值。

{%highlight c%}
intptr_t hash() const {
    return mask_bits(value() >> hash_shift, hash_mask);
}

uintptr_t value() const { 
    return (uintptr_t) this; 
}

inline intptr_t mask_bits (intptr_t  x, intptr_t m) { 
    return x & m; 
}
{%endhighlight%}
