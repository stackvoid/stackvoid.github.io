---
layout: post
title: Java类的生命周期
categories: [Java]
tags: [JVM]
---

编写一个java的源文件经编译后会生成一个后缀名为class的字节码文件，只有字节码文件才能够在Java虚拟机中运行，本文讨论的Java类的生命周期就是指一个字节码文件（.class文件）从加载到卸载的全过程。
一个java类的完整的生命周期会经历加载、连接、初始化、使用、和卸载五个阶段，当然也有在加载或者连接之后没有被初始化就直接
被使用的情况，如图所示：

![Java类完整生命周期](http://stackvoid.qiniudn.com/2014080101.jpg)

###加载阶段

加载阶段，虚拟机需要完成以下3件事情：

1. 通过一个类的全限定名来获取定义此类的二进制字节流。
1. 将这个字节流所代表的静态存储结构转化为方法区的运行时数据结构。
1. 在内存中生成一个代表这个类的java.lang.Class对象（HotSpot放在方法区里），作为这个类的各种数据访问入口。

**常用类加载方式有多种：**

- 根据类的全路径名找到相应的class文件，然后从class字节码文件中读取文件内容。

- 另一种是从jar文件中读取，然后在转化成各种运行时数据结构。

- 从网络中读取，运行时计算生成（动态代理技术），从数据库读取等等...

类的加载时机各个虚拟机实现不太一样，但是必须在一个类使用之前将此类加载。HopSpot虚拟机采用的是当一个类真正被用到才对它进行加载。

加载结束后就是连接阶段；加载阶段总是在连接阶段开始之前，连接阶段总是在加载阶段完成之后完成（二者可能交叉并行进行）。

###连接阶段

连接阶段主要做一些加载后的验证以及一些初始化前的准备工作（准备和解析）。

- **验证：**字节码文件被加载后，必须验证这个文件时候合法（合法才能被JVM运行），不会危害虚拟机的自身安全，有文件格式验证、元数据验证、字节码验证、符号引用验证。如果验证失败，会抛出java.lang.VerifyError异常。验证阶段对于虚拟机来说非常重要，但是不是一个必需的阶段，如果所运行的代码已经反复被使用和验证过了，可以通过-Xverify:none参数关闭大部分的验证措施，以提高虚拟机时间时间。

- **准备：** 给类的静态变量分配内存并设为JVM默认的值。对于非静态的变量，则不会为它们分配内存。注意，**静态变量的初值为jvm默认的初值，而不是我们在程序中设定的初值**。jvm默认的初值是这样的：

> 基本类型（int、long、short、char、byte、boolean、float、double）的默认值为0。
> 引用类型的默认值为null。
> 常量的默认值为我们程序中设定的值，对于普通非final的类变量，如public static int value = 123;在准备阶段过后的初始值是0(数据类型的零值)，而不是123，而把123赋值给value是在初始化阶段才进行的动作。
对于final的类变量，即常量，如public static final int value =123;在准备阶段过程的初始值直接就是123了，不需要准备为零值。

- **解析：** 把常量池中的符号引用转换为直接引用。

> 符号引用和直接引用关系。
比如在内存中找一个类的show方法，显然是很难找到。但是在解析阶段，JVM就会把show这个方法转换为指向方法区的的一块内存地址(此方法执行的起始地址)，比如0x894FAB12，通过0x894FAB12就可以找到show这个方法具体分配在内存的哪一个区域了。其中show就是符号引用，而0x894FAB12就是直接引用。在解析阶段，jvm会将所有的类或接口名、字段名、方法名转换为具体的内存地址。

###初始化

若一个类被直接引用，就会触发类的初始化，在Java中，直接引用的情况有：

- 通过关键字new实例化对象、读取或设置类的静态变量、调用类的静态方法。

- 通过反射执行实例化对象、读取或设置累的静态变量、调用类的静态方法。
 
- 初始化子类的时候触发父类的初始化。

- 作为程序入口直接运行（main方法）。

除了以上四种情况，其他使用类的方式叫做被动引用，而被动引用不会触发类的初始化。


{%highlight java%}

import java.lang.reflect.Field;
import java.lang.reflect.Method;

class InitClass{
	static {
		System.out.println("初始化InitClass");
	}
	public static String a = null;
	public static void method(){}
}

class SubInitClass extends InitClass{}

public class Test1 {

	/**
	 * 主动引用引起类的初始化的第四种情况就是运行Test1的main方法时
	 * 导致Test1初始化，这一点很好理解，就不特别演示了。
	 * 本代码演示了前三种情况，以下代码都会引起InitClass的初始化，
	 * 但由于初始化只会进行一次，运行时请将注解去掉，依次运行查看结果。
	 * @param args
	 * @throws Exception
	 */
	public static void main(String[] args) throws Exception{
	//  主动引用引起类的初始化一:new对象、读取或设置类的静态变量、调用类的静态方法。
		new InitClass();
		InitClass.a = "";
		String a = InitClass.a;
		InitClass.method();
		
	//  主动引用引起类的初始化二：通过反射实例化对象、读取或设置类的静态变量、调用类的静态方法。
		Class cls = InitClass.class;
		cls.newInstance();
		
		Field f = cls.getDeclaredField("a");
		f.get(null);
		f.set(null, "s");
	
		Method md = cls.getDeclaredMethod("method");
		md.invoke(null, null);
			
	//  主动引用引起类的初始化三：实例化子类，引起父类初始化。
		new SubInitClass();

	}
}

{%endhighlight%}

类的初始化过程是这样的：**按照顺序自上而下运行类中的变量赋值语句和静态语句，如果有父类，则首先按照顺序运行父类中的变量
赋值语句和静态语句。**

> 在类的初始化阶段，只会初始化与类相关的静态赋值语句和静态语句，也就是有static关键字修饰的信息，而没有static修饰的赋值
语句和执行语句在实例化对象的时候才会运行。

以上三个过程构成了类加载器，下面介绍**类加载的双亲委派模型**

双亲委派模型要求除了顶层的启动类加载器外，其他的类加载器都应当有自己的父类加载器。这里类加载器之间的父子关系一般不会以继承关系来实现，而是都使用组合关系来复用父加载器的代码

**工作过程：**

如果一个类加载器收到了类加载的请求，它首先不会自己去尝试加载这个类，而是把这个请求委派给父类加载器去完成，每一个层次的类加载器都是如此，因此所有的加载请求最终都应该传递到顶层的启动类加载器中，
只有当父类加载器反馈自己无法完成这个请求（它的搜索范围中没有找到所需的类）时，子加载器才会尝试自己去加载。

**好处：**

   Java类随着它的类加载器一起具备了一种带有优先级的层次关系。例如类Object，它放在rt.jar中，无论哪一个类加载器要加载这个类，最终都是委派给启动类加载器进行加载，因此Object类在程序的各种类加载器环境中都是同一个类
   判断两个类是否相同是通过classloader.class这种方式进行的，所以哪怕是同一个class文件如果被两个classloader加载，那么他们也是不同的类。

###使用

当初始化完成后，Java虚拟机就可以执行Class的业务逻辑指令，通过堆中java.lang.Class对象的入口地址，调用方法区的方法逻辑，
最后将方法的运算结果通过方法返回地址存放到方法区或堆中。

类的使用包括主动引用和被动引用，主动引用在初始化的章节中已经说过了，下面我们主要来说一下被动引用：

- 引用父类的静态字段，只会引起父类的初始化，而不会引起子类的初始化。

- 通过数组定义来引用类，不会触发此类的初始化。例如SupperClass[] sca = new SupperClass[10];

- 引用类的常量不会触发此类的初始化。常量在编译阶段会存入调用类的常量池（NotInitialization类），本质上没有直接引用到定义常量的类，因此不会触发定义常量的类的初始化。


示例代码：
{%highlight java%}
class InitClass{
	static {
		System.out.println("初始化InitClass");
	}
	public static String a = null;
	public final static String b = "b";
	public static void method(){}
}

class SubInitClass extends InitClass{
	static {
		System.out.println("初始化SubInitClass");
	}
}

public class Test4 {

	public static void main(String[] args) throws Exception{
		String a = SubInitClass.a;// 引用父类的静态字段，只会引起父类初始化，而不会引起子类的初始化
		String b = InitClass.b;// 使用类的常量不会引起类的初始化
		SubInitClass[] sc = new SubInitClass[10];// 定义类数组不会引起类的初始化
	}
}

{%endhighlight%}

总结一下使用阶段：**使用阶段包括主动引用和被动引用，主动饮用会引起类的初始化，而被动引用不会引起类的初始化。**
当使用阶段完成之后，java类就进入了卸载阶段。

###卸载

类使用完之后，如果满足下面的情况，类就会被卸载：

- 该类所有的实例都已经被回收，也就是java堆中不存在该类的任何实例。

- 加载该类的ClassLoader已经被回收。

- 该类对应的java.lang.Class对象没有任何地方被引用，无法在任何地方通过反射访问该类的方法。

如果以上三个条件全部满足，jvm就会在方法区垃圾回收的时候对类进行卸载，类的卸载过程其实就是在方法区中清空类信息，java类的整个生命周期就结束了。

对象基本上都是在jvm的堆区中创建，在创建对象之前，会触发类加载（加载、连接、初始化），当类初始化完成后，根据类信息在堆区中实例化类对象，初始化非静态变量、非静态代码以及默认构造方法，当对象使用完之后会在合适的时候被jvm垃圾收集器回收。读完本文后我们知道，对象的生命周期只是类的生命周期中使用阶段的主动引用的一种情况（即实例化类对象）。而类的整个生命周期则要比对象的生命周期长的多。

####参考

1. [深入理解java虚拟机Java虚拟机类生命周期](http://www.cnblogs.com/hnlshzx/p/3533264.html)

2. [深入理解Java虚拟机:JVM高级特性与最佳实践](book.douban.com/subject/6522893/)
