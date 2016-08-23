---
layout: post
title: Java基本概念原理总结(二)
categories: [Java]
tags: [Summary]
---

本文是一些Java基本字符串、数组和I/O的总结。主要来自《Java编程思想》和互联网。

### 字符串

String对象时不可变的，String类中每一个看起来修改String值的方法，实际上都是创建了了一个全新的String对象，以包含修改后的字符串内容，而最初的String对象则丝毫未动。

用于String的 + 和 += 是Java中仅有的两个重载过的操作符，而Java不允许程序员重载任何操作符。

预先指定StringBuilder大大小可以皮面多次分配缓冲；若在toString()方法里使用循环，最好自己创建一个StringBuilder对象，用它来构造最终结果。

**字符串查找**

String提供了两种查找字符串的方法，即indexOf与lastIndexOf方法。

indexOf（String s)

该方法用于返回参数字符串s在指定字符串中首次出现的索引位置，当调用字符串的indexOf()方法时，会从当前字符串的开始位置搜索s的位置；如果没有检索到字符串s，该方法返回-1

```java
 String str ="We are students";
 int size = str.indexOf("a"); // 变量size的值是3
```
lastIndexOf(String str)

该方法用于返回字符串最后一次出现的索引位置。当调用字符串的lastIndexOf()方法时，会从当前字符串的开始位置检索参数字符串str，并将最后一次出现str的索引位置返回。如果没有检索到字符串str，该方法返回-1.

如果lastIndexOf方法中的参数是空字符串""，则返回的结果与length方法的返回结果相同。

**获取指定索引位置的字符**

使用charAt()方法可将指定索引处的字符返回。

```java
 String str = "hello word";
 char mychar =  str.charAt(5);  // mychar的结果是w
```

**获取子字符串**

通过String类的substring()方法可对字符串进行截取。这些方法的共同点就是都利用字符串的下标进行截取，且应明确字符串下标是从0开始的。在字符串中空格占用一个索引位置。

```java
substring(int beginIndex)
```

该方法返回的是从指定的索引位置开始截取知道该字符串结尾的子串。

```java
String str = "Hello word";
String substr = str.substring(3); //获取字符串，此时substr值为lo word
```

substring(int beginIndex,  int endIndex)

beginIndex : 开始截取子字符串的索引位置

endIndex：子字符串在整个字符串中的结束位置

```java
 String str = "Hello word";
String substr = str.substring(0,3); //substr的值为hel
```

**去除空格**

trim()方法返回字符串的副本，忽略前导空格和尾部空格。

**字符串替换**

replace()方法可实现将指定的字符或字符串替换成新的字符或字符串

oldChar：要替换的字符或字符串

newChar：用于替换原来字符串的内容

如果要替换的字符oldChar在字符串中重复出现多次，replace()方法会将所有oldChar全部替换成newChar。需要注意的是，要替换的字符oldChar的大小写要与原字符串中字符的大小写保持一致。

```java
String str= "address";
String newstr = str.replace("a", "A");// newstr的值为Address
```

**判断字符串的开始与结尾**

startsWith()方法与endsWith()方法分别用于判断字符串是否以指定的内容开始或结束。这两个方法的返回值都为boolean类型。

startsWith(String prefix) 该方法用于判断当前字符串对象的前缀是否是参数指定的字符串。

endsWith(String suffix) 该方法用于判断当前字符串是否以给定的子字符串结束

**判断字符串是否相等**

equals(String otherstr)

如果两个字符串具有相同的字符和长度，则使用equals()方法比较时，返回true。同时equals()方法比较时区分大小写。

equalsIgnoreCase(String otherstr)

equalsIgnoreCase()方法与equals()类型，不过在比较时忽略了大小写。

**按字典顺序比较两个字符串**

compareTo()方法为按字典顺序比较两个字符串，该比较基于字符串中各个字符的Unicode值，按字典顺序将此String对象表示的字符序列与参数字符串所表示的字符序列进行比较。如果按字典顺序此String对象位于参数字符串之前，则比较结果为一个负整数；如果按字典顺序此String对象位于参数字符串之后，则比较结果为一个正整数；如果这两个字符串相等，则结果为0.

str.compareTo(String otherstr);

**字母大小写转换**

字符串的toLowerCase()方法可将字符串中的所有字符从大写字母改写为小写字母，而tuUpperCase()方法可将字符串中的小写字母改写为大写字母。

```java
str.toLowerCase();
str.toUpperCase();
```

**字符串分割**

使用split()方法可以使字符串按指定的分隔字符或字符串对内容进行分割，并将分割后的结果存放在字符数组中。

str.split(String sign);

sign为分割字符串的分割符，也可以使用正则表达式。

没有统一的对字符串进行分割的符号，如果想定义多个分割符，可使用符号“|”。例如，“,|=”表示分割符分别为“，”和“=”。

 str.split(String sign, in limit)；

该方法可根据给定的分割符对字符串进行拆分，并限定拆分的次数。

**格式化输出**

Java中，所有新的格式化功能都是由java.util.Formatter类处理。

在插入数据时，如果想要控制空格与对齐，你需要更精细复杂的格式修饰符，以下是其抽象语法：

{% highlight java %}
%[argument_index$][flags][width][.precision]conversion
{% endhighlight %}

可选的 argument_index 是一个十进制整数，用于表明参数在参数列表中的位置。第一个参数由 "1$" 引用，第二个参数由 "2$" 引用，依此类推。

可选 flags是修改输出格式的字符集。有效标志集取决于转换类型。

可选 width是一个非负十进制整数，表明要向输出中写入的最少字符数。

可选 precision是一个非负十进制整数，通常用来限制字符数。特定行为取决于转换类型。

所需 conversion是一个表明应该如何格式化参数的字符。给定参数的有效转换集取决于参数的数据类型。

用来表示日期和时间类型的格式说明符的语法如下：

{% highlight java %}
%[argument_index$][flags][width]conversion
{% endhighlight %}

可选的 argument_index、flags 和 width的定义同上。

所需的 conversion 是一个由两字符组成的序列。第一个字符是 't' 或 'T'。第二个字符表明所使用的格式。这些字符类似于但不完全等同于那些由 GNU date 和 POSIX strftime(3c)定义的字符。

[更详细见这篇文章](http://www.cnblogs.com/Lowp/archive/2012/09/16/2687745.html)和[很多例子](http://lavasoft.blog.51cto.com/62575/179081).

*正则表达式*我会单独列出来学习总结。

### 数组

数组使用[]来访问元素，而List使用add()和get()方法。
无论使用何种类型，数组对象用来保存指向其他对象的引用。

新生成一个数组对象时，其中所有的引用被自动化初始化为null；基本类型数值型被自动初始化为0，字符型（char），被自动初始化为(char)0，布尔类型全被初始化为false。

**多维数组**

对于基本类型多维数组，可以通过使用花括号将每个向量分开。

数组浅复制和深复制。直接复制对象数组，仅仅是复制了对象的引用。

### Java I/O

Java 的 I/O 操作类在包 java.io 下，大概有将近 80 个类，但是这些类大概可以分成四组，分别是：

- 基于字节操作的 I/O 接口：InputStream 和 OutputStream

- 基于字符操作的 I/O 接口：Writer 和 Reader

- 基于磁盘操作的 I/O 接口：File

- 基于网络操作的 I/O 接口：Socket

前两组主要是根据传输数据的数据格式，后两组主要是根据传输数据的方式，虽然 Socket 类并不在 java.io 包下，但是我仍然把它们划分在一起，因为我个人认为 I/O 的核心问题要么是数据格式影响 I/O 操作，要么是传输方式影响 I/O 操作，也就是将什么样的数据写到什么地方的问题，I/O 只是人与机器或者机器与机器交互的手段，除了在它们能够完成这个交互功能外，我们关注的就是如何提高它的运行效率了，而数据格式和传输方式是影响效率最关键的因素了。我们后面的分析也是基于这两个因素来展开的。

**File类**

在整个IO包了，唯一表示与文件本身有关的类就是File类。使用File类可以进行创建或删除文件等常用操作。要想使用File类。则首先要观察File类的构造方法，此类的常用构造方法如下所示：

public File(String pathname)  

实例化File类的时候，必须设置好路径。

![File方法](http://stackvoid.qiniudn.com/2013-05-01-class-file.png)

[File类使用的例子](http://www.cnblogs.com/lich/archive/2011/12/10/2283445.html)

*File类是在java.io包中唯一与文件本身有关的,可以使用File类创建、删除等常见的文件操作,在使用File类指定路径的时候一定要注意操作系统间的差异，尽量使用separator进行分割。*

**RandomAccessFile类**

File类只是针对文件本身进行操作的，而如果相对文件内容进行操作，则可以使用RandomAccessFile类，此类属于随即读取类，可以随机的读取一个文件中指定位置的数据。

因为在文件中，所有得内容都是按照字节存放的，都有固定的保存位置。

构造函数：

public RandomAccessFile(*File file,String mode*)throws FileNotFoundException

实例化此类的时候需要传递File类。告诉程序应该操作的是哪个文件，之后有个模式，文件的打开模式，常用的两种模式：

r：读
w：写
rw：读写，如果使用此模式，如果文件不存在，则会自动创建。
[RandomAccessFile类例子](http://www.cnblogs.com/lich/archive/2011/12/10/2283590.html)



**基于字节的 I/O 操作接口**

字节流主要是操作byte类型数据，以byte数组为准，主要操作类就是OutputStream、InputStream.

[字节输出流：OutputStream]()

OutputStream是整个IO包中[字节输出流的最大父类]()，此类的定义如下：

[public abstract class OutputStream extends Object implements Closeable,Flushable]()

从以上的定义可以发现，此类是一个抽象类，如果想要使用此类的话，则首先必须通过子类实例化对象，那么如果现在要操作的是一个文件，则可以使用：FileOutputStream类。通过向上转型之后，可以为OutputStream实例化

[Closeable表示可以关闭的操作，因为程序运行到最后肯定要关闭]()

[Flushable：表示刷新，清空内存中的数据]()

FileOutputStream类的构造方法如下：

[public FileOutputStream(File file)throws FileNotFoundException]()

例子[click](http://www.cnblogs.com/lich/archive/2011/12/11/2283700.html)

[字节输入流：InputStream]()

程序既然可以向文件中写入内容，则就可以通过InputStream从文件中把内容读取进来，首先来看InputStream类的定义：

```java
public abstract class InputStream extends Object implements Closeable
```
与OutputStream类一样，InputStream本身也是一个抽象类，必须依靠其子类，如果现在是从文件中读取，就用FileInputStream来实现。

观察FileInputStream类的构造方法：

```java
public FileInputStream(File file)throws FileNotFoundException
```

例子[click](http://www.cnblogs.com/lich/archive/2011/12/11/2283700.html)

[字符输出流：Writer]()

Writer本身是一个字符流的输出类，此类的定义如下：

{% highlight java %}
public abstract class Writer extends Object implements Appendable，Closeable，Flushable
{% endhighlight %}

此类本身也是一个抽象类，如果要使用此类，则肯定要使用其子类，此时如果是向文件中写入内容，所以应该使用FileWriter的子类。

FileWriter类的构造方法定义如下：

{% highlight java %}
public FileWriter(File file)throws IOException
{% endhighlight %}

字符流的操作比字节流操作好在一点，就是可以直接输出字符串了，不用再像之前那样进行转换操作了。

[字符输入流：Reader]()

Reader是使用字符的方式从文件中取出数据，Reader类的定义如下：

public abstract class Reader extends Objects implements Readable，Closeable

Reader本身也是抽象类，如果现在要从文件中读取内容，则可以直接使用FileReader子类。

FileReader的构造方法定义如下：

{% highlight java %}
public FileReader(File file)throws FileNotFoundException
{% endhighlight %}

[例子](http://www.cnblogs.com/lich/archive/2011/12/11/2283700.html)


**总结**

基于字节的 I/O 操作接口输入和输出分别是：InputStream 和 OutputStream，InputStream 输入流的类继承层次如下图所示：

![InputStream 相关类层次结构](http://stackvoid.qiniudn.com/2013-0501-Core-Java201.png)

输入流根据数据类型和操作方式又被划分成若干个子类，每个子类分别处理不同操作类型，OutputStream 输出流的类层次结构也是类似，如下图所示：

![OutputStream](http://stackvoid.qiniudn.com/2013-0501-Core-Java202.png)

说明两点，一个是操作数据的方式是可以组合使用的，如这样组合使用

```java
OutputStream out = new BufferedOutputStream(new ObjectOutputStream(new FileOutputStream("fileName"))；
```

还有一点是流最终写到什么地方必须要指定，要么是写到磁盘要么是写到网络中，其实从上面的类图中我们发现，写网络实际上也是写文件，只不过写网络还有一步需要处理就是底层操作系统再将数据传送到其它地方而不是本地磁盘。关于网络 I/O 和磁盘 I/O 我们将在后面详细介绍。

**基于字符的 I/O 操作接口**

不管是磁盘还是网络传输，最小的存储单元都是字节，而不是字符，所以 I/O 操作的都是字节而不是字符，但是为啥有操作字符的 I/O 接口呢？这是因为我们的程序中通常操作的数据都是以字符形式，为了操作方便当然要提供一个直接写字符的 I/O 接口，如此而已。我们知道字符到字节必须要经过编码转换，而这个编码又非常耗时，而且还会经常出现乱码问题，所以 I/O 的编码问题经常是让人头疼的问题。关于 I/O 编码问题请参考另一篇文章 [《深入分析Java中的中文编码问题》](http://www.ibm.com/developerworks/cn/java/j-lo-chinesecoding/)。

下图是写字符的 I/O 操作接口涉及到的类，Writer 类提供了一个抽象方法 write(char cbuf[], int off, int len) 由子类去实现。

![Writer 相关类层次结构](http://stackvoid.qiniudn.com/2013-0501-Core-Java203.jpg)

读字符的操作接口也有类似的类结构，如下图所示：

![Reader类层次结构](http://stackvoid.qiniudn.com/2013-0501-Core-Java203.jpg)

读字符的操作接口中也是 int read(char cbuf[], int off, int len)，返回读到的 n 个字节数，不管是 Writer 还是 Reader 类它们都只定义了读取或写入的数据字符的方式，也就是怎么写或读，但是并没有规定数据要写到哪去，写到哪去就是我们后面要讨论的基于磁盘和网络的工作机制。

**字节与字符的转化接口**

另外数据持久化或网络传输都是以字节进行的，所以必须要有字符到字节或字节到字符的转化。字符到字节需要转化，其中读的转化过程如下图所示：

![字符解码相关类结构](http://stackvoid.qiniudn.com/2013-0501-Core-Java205.jpg)

InputStreamReader 类是字节到字符的转化桥梁，InputStream 到 Reader 的过程要指定编码字符集，否则将采用操作系统默认字符集，很可能会出现乱码问题。StreamDecoder 正是完成字节到字符的解码的实现类。也就是当你用如下方式读取一个文件时：

{% highlight java %}
 try { 
            StringBuffer str = new StringBuffer(); 
            char[] buf = new char[1024]; 
            FileReader f = new FileReader("file"); 
            while(f.read(buf)>0){ 
                str.append(buf); 
            } 
            str.toString(); 
 } catch (IOException e) {}
{% endhighlight %}

FileReader 类就是按照上面的工作方式读取文件的，FileReader 是继承了 InputStreamReader 类，实际上是读取文件流，然后通过 StreamDecoder 解码成 char，只不过这里的解码字符集是默认字符集。
写入也是类似的过程如下图所示：

![字符编码相关类结构2](http://stackvoid.qiniudn.com/2013-0501-Core-Java206.jpg)

通过 OutputStreamWriter 类完成，字符到字节的编码过程，由 StreamEncoder 完成编码过程。

上面Java I/O来自 [深入分析 Java I/O 的工作机制](http://www.ibm.com/developerworks/cn/java/j-lo-javaio/)

[字节流与字符流的区别]()

字节流和字符流使用是非常相似的，那么除了操作代码的不同之外，还有哪些不同呢？

字节流在操作的时候本身是不会用到缓冲区（内存）的，是与文件本身直接操作的，而字符流在操作的时候是使用到缓冲区的

字节流在操作文件时，即使不关闭资源（close方法），文件也能输出，但是如果字符流不使用close方法的话，则不会输出任何内容，说明字符流用的是缓冲区，并且可以使用flush方法强制进行刷新缓冲区，这时才能在不close的情况下输出内容


那开发中究竟用字节流好还是用字符流好呢？

在所有的硬盘上保存文件或进行传输的时候都是以字节的方法进行的，包括图片也是按字节完成，而字符是只有在内存中才会形成的，所以使用字节的操作是最多的。


如果要java程序实现一个拷贝功能，应该选用字节流进行操作（可能拷贝的是图片），并且采用边读边写的方式（节省内存）。

**内存操作流**

之前所讲解的程序中，输出和输入都是从文件中来得，当然，也可以将输出的位置设置在内存之上，此时就要使用ByteArrayInputStream、ByteArrayOutputStream来完成输入输出功能了。

ByteArrayInputStream的主要功能将内容输入到内存之中，

ByteArrayOutputStream的主要功能是将内存中的数据输出，

此时应该把内存作为操作点。

ByteArrayInputStream类的定义：

{% highlight java %}
public class ByteArrayInputStream extends InputStream
{% endhighlight %}

构造方法：

{% highlight java %}
public ByteArrayInputStream(byte[] buf)
{% endhighlight %}

接受一个byte数组，实际上内存的输入就是在构造方法上将数据传入到内存中。

ByteArrayOutputStream：输出就是从内存中[写出]()数据：

{% highlight java %}
public void write(int b)
{% endhighlight %}

**管道流**

管道流是JAVA中线程通讯的常用方式之一，可以进行两个线程间的通讯，分为管道输出流(PipedOutputStream)、管道输入流（PipedInputStream），如果想要进行管道输出，则必须要把输出流连在输入流之上，在PipedOutputStream类上有如下的一个方法用于连接管道：

{% highlight java %}
public void connect(PipedInputStream snk)throws IOException
{% endhighlight %}

基本流程如下（[Example](http://www.cnblogs.com/king1302217/p/3158842.html)）：

1）创建管道输出流PipedOutputStream pos和管道输入流PipedInputStream pis

2）将pos和pis匹配，pos.connect(pis);

3）将pos赋给信息输入线程，pis赋给信息获取线程，就可以实现线程间的通讯了。

[管道流只能实现单向发送，如果要两个线程之间互通讯，则需要两个管道流]()

 可以看到上面的[例子](http://www.cnblogs.com/king1302217/p/3158842.html)中，线程producer通过管道流向线程consumer发送数据，如果线程consumer想给线程producer发送数据，则需要新建另一个管道流pos1和pis1，将pos1赋给consumer1，将pis1赋给producer，具体例子本文不再多说。

 **BufferedReader和Scanner**

如果想要接收任意长度的数据，而且避免乱码产生，就可以使用BufferedReader类

{% highlight java %}
public class BufferedReader extends Reader
{% endhighlight %}

因为输入的数据有可能出现中文，所以，此处使用字符流完成。BufferedReader是从缓冲区之中读取内容，所有的输入的字节数据都将放在缓冲区之中。

System.in本身表示的是InputStream（字节流），现在要求接收的是一个字符流，需要将字节流变成字符流才可以，所以要用InputStreamReader。

使用Scanner接收键盘的输入数据。