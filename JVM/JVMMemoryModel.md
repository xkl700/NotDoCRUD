
# JVM内存模型（JDK8 HotSpot）

**JVM内存模型 - 需要掌握：JVM运行时数据区、类加载子系统、垃圾回收器**

下面是我改的一张JDK7的图，有地方画的不对大家给我指正以下：
![JVM全局概览](https://i.loli.net/2021/03/29/7ZwyPFCdH8aDgi6.png)

## 类加载子系统

### 类加载子系统工作流程
在Java虚拟机中，负责查找并封装类的部分称为类加载子系统，类加载子系统用于定位和加载编译后的class文件；
**类加载会将类的信息加入到方法区（元数据空间 以下简称元空间）**。

**字节码 -> 类加载 -> 链接(验证 -> 准备 -> 解析) -> 初始化 -> 使用 -> 卸载**

<div >
	<img src="https://i.loli.net/2021/03/29/pgh2PmxaqDiRSY5.png" alt="类加载的生命周期" style="zoom:100%;" />
    <p>注意：步骤是这么几个步骤，顺序不是绝对的，例如：多态，程序在运行时才知道对象和类是什么类型的，此时解析和初始化顺序可能会发生颠倒</p>
    <p>类加载的时机：按需加载</p>
</div>

### 类加载器

通过一个类的全限定名来获取描述此类的二进制字节流；

类加载器主要实现类的加载；

```tex
com.xxx.Test --> com.xxx.Test.class --> Class<Test>
java.long.String --> java.long.String.class --> Class<String>
```

### 双亲委派模型

- Bootstrap ClassLoader 启动类加载器（C++ 实现，是虚拟机的一部分）；（以下简称BC）
- Extension ClassLoader 扩展类加载器（Java实现，独立于虚拟机外部且全继承自java.lang.ClassLoader）；（以下简称EC）
- Application ClassLoader 应用程序加载器（Java实现）；（以下简称AC）
- Custom ClassLoader 自定义类加载器（Java实现，用户自定义的）；（以下简称CC）

除了顶层的启动器加载器之外，其他类加载器都有自己的父类加载器；

![双亲委派模型](https://i.loli.net/2021/03/25/frpEPOsVlbqHJ98.png)

eg: 自定义一个String类，它是如何加载的？

​	不能加载，由AC开始检查是否可以加载，一层一层向上委托（AC -> EC ->BC）到BC时，BC中由于已经加载`java.long.String`类所以一层层向下（BC -> EC ->AC）告知加载器不能在加载自定义String类.

- 类加载器工作过程：如果一个类加载器收到一个类加载请求，他首先不会自己加载，而是把这个请求委派给父类加载器，只有弗雷加载器无法完成加载时子类加载器才会尝试加载。

> 另外从网上找到一篇文章[你知道 Java 类是如何被加载的吗？](https://www.sohu.com/a/331624683_612370)从源码讲述了类加载的机制

## JVM运行时数据区

JVM在运行时会把它所管理的内存划分为若干不同的数据区域，总共是五个区域，宏观上划分为两大块：

- 线程私有数据区
- 线程公有数据区

![JVM运行时数据区域](https://i.loli.net/2021/03/25/3ZiP8voyxAQS7rK.png)

### 线程私有数据区 -- 程序计数器

> 每个线程都有一个程序计数器，是线程私有的，就是一个指针，指向方法区中的方法字节码（用来存储指向像一条指令的地址，也即将要执行的指令代码），在执行引擎读取下一条指令，是一个非常小的内存空间，几乎可以忽略不计。
- 记录程序执行位置、行号
- 一块很小的区域
- 线程私有
- 不存在OutOfMemoryError
- 无GC回收

### 线程私有数据区 -- 虚拟机栈

虚拟机栈是采用了一种栈的数据结构，入口和出口只有一个，分为入栈和出栈，先进后出；虚拟机栈主要是执行方法。
一个方法一个栈帧，存储当前线程运行方法所需要的数据，指令，返回地址。线程私有，无GC.

- 局部变量表 ： 是一组变量值存储空间，用于存放方法参数和方法内定义的局部变量。

- 操作数栈： 也叫操作栈，他是一个先进后出的栈。

- 动态链接：一个方法要调用其他方法，需要将这些方法的符号引用转化为其内存地址中的直接引用。

- 返回地址：方法不管是正常执行结束还是异常退出，需要返回方法被调用的位置。

  <table><tr>
  <td><img src="https://i.loli.net/2021/03/25/81i9MUrTYsqQGvE.png" alt="虚拟机栈" style="zoom:70%;" /></td>
  <td><img src="https://i.loli.net/2021/03/29/wnTApWV7zZNlhCs.png" alt="方法的调用" style="zoom:150%;" /></td>
  </tr></table>

![image-20210325143815086](https://i.loli.net/2021/03/25/uVEsFNCZX2ogqI8.png)

### 线程私有数据区 -- 本地方法栈

- 与虚拟机栈基本类似
- 区别在于本地方法栈为Native方法服务
- Sun HotSpot将虚拟机栈和本地方法栈合并
- 有`StackOverfFowError`和`OutOfMemoryError`(比较少见)
- GC不会回收该区域

**栈的OutOfMenmoryError溢出一般是在多线程条件下可能会产生，建立过多的线程，每个线程的运行时间又比较长，可能产生栈的OutOfMemoryError溢出；**

单个线程下，无论是由于栈帧太大还是虚拟机容量太小，当内存无法分配时，虚拟机抛出`StackOverFlowFrror`的错误异常；

### 线程私有部分的整体特征（小结）

是线程私有的，随着线程执行结束而结束（JVM就销毁了虚拟机栈里面的栈帧），是比较有规律的，问题会少一些，出问题比较多的是线程共享的部分，也是堆和方法区（元空间）

### 线程共享部分 -- 方法区（元空间）

- 方法区（jdk1.7后的某个版本合并到了堆）
- 方法区在JDK1.8称为元空间（Metaspace）,元空间与堆不相连，使用本机内存
![元空间都放了些什么](https://img-blog.csdnimg.cn/20210308155224155.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L0ZIX1pR,size_16,color_FFFFFF,t_70)
> [元空间](https://blog.csdn.net/FH_ZQ/article/details/114534054) 这里有一段很详细的讲了Metaspase中放了什么

**特点：**

- 线程共享
- 存储 类信息、常量、运行时常量池、即时编译器编译后的代码等数据（字符串常量池、静态变量在堆中）
- HotSpot虚拟机上将方法区叫永久代（1.7之前）
- 垃圾收集器很少光顾该区域（无GC回收）
- 方法区1.7及之前通过`-XX:MaxPermSize`设置最大值 ，元空间1.8是`-XX:MaxMetaspaceSize=48m`最大可分配大小 ,`-XX:MetaspaceSize`设置元空间初始大小
- 空间不够分配时`OutOfMemoryError`

#### 方法区内存溢出案例演示

- `-XX:+PrintGCDetails -XX:+PrintGCDateStamps -XX:+UseCompressedClassPointers -XX:MetaspaceSize=10M -XX:MaxMetaspaceSize=10M`
- 第一个参数用于打印GC日志
- 第二个参数用于打印对应的时间戳
- 第三个参数`-XX:UseCompressedClassPointers`表示在Metaspace中不要开辟出一块新的空间（Compressed Class Space），如果开辟这块空间的话空间默认大小是1G,所以我们关闭该功能，此时在设置Metaspace的大小

![image-20210325152306704](https://i.loli.net/2021/03/25/8o37bBZuqXxsGaz.png)

### 堆

- 线程共享
- 内存中最大的区域
- 虚拟机启动时创建
- 存放所有实例对象或数组
- GC垃圾收集器的主要管理区域
- 可分为新生代、老年代
- 新生代更细化可分为Eden、From Survivor、To Survivor,Eden:Survivor = 8：1：1
- 可通过`-Xmx -Xms`调节堆大小
- 无法再扩展`OutOfMemoryError`，`java.lang.OutOfMenmoryError:Java heap space`

![image-20210327223448634](https://i.loli.net/2021/03/27/MHqPGleLkrTisNF.png)

#### JVM堆溢出

不断创建对象又不是放，当对象到达一定数量，无堆空间将产生堆内存溢出

- 内存泄漏 ：`Memory Leak`，简称ML,GC Roots到对象之间有可达路径而无法收集；
  - 分配的内存没有得到释放
  - 内存一直在增长，有OOM的风险
  - GC时该回收的回收不掉
  - 能够回收掉但很快又占满，产生压力
- 内存溢出：`OutOfMemoryError`，简称OOM ,GC Roots到对象之间无可达路径，可以被收集，但对象还需要存活着，此时可根据物理机内存适当调大虚拟机参数`-Xms`、`-Xmx`,分析代码中对象生命周期是否过长、对象是否持有状态时间过长；
  - 堆是最常见的情况
  - 堆外内存排查困难
  - OOM发生的几个原因：
    - 内存的容量太小，需要扩容，或者需要调整堆的空间
    - 错误的引用方式，发生了ML。没有及时的切断与GC Roots的关系。比如线程池里的线程，在复用的情况下忘记清理ThreadLocal的内容。
    - 接口没有进行范围校验，外部传参超出范围。比如数据库查询时的每页条数等。
    - 对堆外内存无限制的使用。这种情况一旦发生更加严重，会造成操作系统内存耗尽。
> [十种JVM内存溢出的情况，你碰到过几种？](https://segmentfault.com/a/1190000017226359?utm_medium=referral&utm_source=tuicool)

#### 对象的访问过程

- 使用对象时，我们是通过栈上的reference引用来操作堆上的具体对象;

- Sun Hotspot虚拟机使用直接指针访问具体对象：
    - B 类的 .class 信息存放在元空间中
    - b 变量存放在 Java 栈的局部变量表中
    - 真正的 b 对象存放在 Java 堆中
    - 在 b 对象中，有个指针指向元空间中的 B 类型数据，表明这个 b 对象是用方法区中的 B 类 new 出来的

![image-20210329123430983](https://i.loli.net/2021/03/29/FWRfMABbEzOpqLZ.png)  
    
### 对象分配过程

1. new的对象先放在Eden区。这个区域有大小限制。
2. 当Eden区填满时程序又要创建对象，GC将将对Eden预期进行垃圾回收（Minor GC）
3. 将销毁Eden区中不再被其他对象引用的对象，并将新对象加载到Eden区
   - 然后将Eden中的剩余对象移动到From Survivor
4. 如果再次触发GC，那么上次幸存的放在From Survivor区的，如果没有回收，它将被放在To Survivor。
5. 如果再次进行GC，此时会重新返回到From Survivor，接着再去To Survivor。
6. 如果累计次数达到15次，则进入old区。
   - 阈值可以通过以下参数进行调整:-XX:MaxTenuringThreshold=N
7. 如果old区内存不足，它将再次启动GC:Major GC进行old区的内存清理
8. 如果在Major GC之后没有办法保存对象，则将报告一个OOM异常。

### 类如何加载？

学完这一步部分让我们在回过头来看一下类是如何加载的,举例来说`private B b = new B();`时，就会触发类的加载。

<table><tr>
<td><img src="https://i.loli.net/2021/03/27/MESPDwesx5OnJq7.png" alt="代码" style="zoom:150%;" /></td>
<td><img src="https://i.loli.net/2021/03/27/BKyM4Ger2novUcw.png" alt="类如何加载" style="zoom:100%;" /></td>
</tr></table>

首先通过类加载器执行的一系列过程（**字节码 -> 类加载 -> 链接(验证 -> 准备 -> 解析) -> 初始化 -> 使用 -> 卸载**）将class文件加载到元空间；执行静态代码，加载该类的静态信息（类、 static代码、方法）在元空间分配空间；执行非静态代码，在堆中为对象分配空间 （对象信息：类引用、类变量引用、方法引用、成员变量等）

static代码只在类初始化时加载一次，加载后存在元空间，而每一个对象在实例化时，只是在堆中保存指向元空间的引用，所以**全局唯一，一改都改，节省资源**

> [Java HotSpot VM Command-Line Options](https://docs.oracle.com/javase/8/docs/technotes/guides/troubleshoot/clopts001.html#CHDJEIHC)
>
> [JVM - 结合代码示例彻底搞懂Java内存区域_对象在堆-栈-方法区(元空间)之间的关系](https://blog.csdn.net/yangshangwei/article/details/106893464?utm_medium=distribute.pc_relevant.none-task-blog-baidujs_title-0&spm=1001.2101.3001.4242)
> 
> JVM常用参数
>
> -XX:MaxMetaspaceSize 元空间最大值，默认是没有限制的。
>
> -XX:MetaspaceSize 元空间初始空间大小，达到该值就会触发垃圾收集进行类型卸载，同时GC会对该值进行调整
>
> -Xms256m 设置最小堆内存
>
> -Xmx512m 设置最大堆内存
>
> -Xss128k 每个线程堆栈大小，默认256K，减少此数值产生更多线程
>
> -Xmn300m (-XX:NewSize) 设置年轻代内存
>
> -XX:NewRatio 设置年轻代和老年代的比值，如：3 表示年轻代与老年代的比值为1:3
>
> -XX:SurvivorRatio 年轻代中eden区与两个survivor区的比,默认8，即edan:s0:s1=8:1:1
>
> -XX:TargetSurvivorRatio=60 如果Survivor空间的占用超过该设定值，对象在未达到他们的最大年龄
> 之前就会被提升至老年代。默认值为50，即超过50%的进入老年代
>
> -XX:+PrintGCDetails 打印GC详情 -XX:+PrintGC 与 -verbose:gc 打印GC简略信息
>
> -XX:+PrintCommandLineFlags 打印JVM使用的参数
>
> -XX:MaxTenuringThreshold=3 设置对象在年轻代中经历多少次GC后进入老年代，默认15(0-15之间)
  CMS收集器默认为6
>
> -Xloggc:C:\Users\30748\Desktop\JVMTestGC.log 指定GClog存放位置
>
> -XX:+HeapDumpOnOutOfMemoryError 堆溢出时dump hprof格式文件
>
> -XX:HeapDumpPath=C:\Users\30748\java_error_in_idea.hprof 堆溢出文件位置

## GC

### 垃圾收集概念

GC 需要做 3 件事情：

- 分配内存，为每个新建的对象分配空间
- 确保还在使用的对象的内存一直还在，不能把有用的空间当垃圾回收了
- 释放不再使用的对象所占用的空间

我们把还被 **GC Roots** 引用的对象称为**活的**，把不再被引用的对象认为是**死的**，也就是我们说的垃圾，GC 的工作就是找到死的对象，回收它们占用的空间。

在这里，我们总结一下 GC Roots 有哪些：

- 当前各线程执行方法中的局部变量（包括形参）引用的对象
- 已被加载的类的 static 域引用的对象
- 方法区中常量引用的对象
- JNI 引用

> [java的gc为什么要分代？](https://www.zhihu.com/question/53613423/answer/135743258)

我们把 GC 管理的内存称为 **堆（heap）**，垃圾收集启动的时机取决于各个垃圾收集器，通常，垃圾收集发生于整个堆或堆的部分已经被使用光了，或者使用的空间达到了某个百分比阈值。这些后面都会具体说，这里的每一句话都是对应了某些场景的。

对于内存分配请求，实现的难点在于在堆中找到一块没有被使用的确定大小的内存空间。所以，对于大部分垃圾回收算法来说**避免内存碎片化**是非常重要的，它将使得空间分配更加高效。

### 回收策略

> ​	在JVM中内存分配的基本粒度主要是**对象、基本类型**。而基本类型的使用主要是包括在对象中的局部变量，所以回收对象所占用的内存是JAVA垃圾回收的主要目标。
>
> 那么如何判断对象是处于可回收状态的呢？在主流的JVM中是采用“可达性分析算法”来进行判断的。
>
> 这个算法的基本思路就是通过一系列的称为“GC Roots”的对象作为起始点，并从这些节点开始往下进行搜索，搜索走过的路径我们称之为引用链（Reference Chain），当一个对象到GC Roots没有任何引用链相连时，我们就称之为对象引用不可达，则证明这个对象是不可用的，就可以暂时判定这个对象为可回收对象。示意图如下：
>
> ![可达性分析](https://i.loli.net/2021/03/29/Y8b1SD7uAp4cOKj.jpg)
>
> 在图中虽然Obj F与Obj J之间互相有关联但是它们到GC Roots是不可达的，所以将会被判定为可回收对象。既然如此，什么样的对象可以作为GC Roots对象呢？
>
> 在JAVA中可以被作为GC Roots的对象主要是：虚拟机栈-栈帧中的本地变量表所引用的对象、方法区（<JDK1.8）中类静态属性所引用的对象／常量属性所引用的对象、本地方法栈中引用的对象。
>
> 这里还需要注意一个小的细节，就是被判定为对象不可达的对象也并非会被立刻回收，在学习JAVA语法是我们应该学习过finalize()方法，如果对象重写了finalize方法，并重新把this关键字赋值给了某个类变量或对象的成员变量的话，该对象就会被**“救活”，**具体过程可参考上图所示，只是这种方式并不鼓励大家使用，了解下就行。
>
> 在关于如何判定对象是否属于不再使用的内存时，还有个通常会被大家错误认为是JVM使用的方式-“引用计数法”，事实上引用计数法的实现比较简单，判定效率也比较高，在Python语言中就使用了这种算法进行内存管理，但是它有一个比较难解决的对象之间循环引用的问题，所以在JAVA虚拟机里并没有选用“引用计数法”来管理内存。这个问题很多人都会搞错，包括有很多年开发经验的程序员，需要大家注意下！



### 垃圾回收算法

在JVM中主要的垃圾收集算法有：标记-清除、标记-清除-压缩（简称**“标记-整理”）、标记-复制-清除（简称“复制”）、分代收集算法**。这几种收集算法互相配合，针对不同的内存区域采取对应的收集算法实现（这里具体是由相应的垃圾收集器实现）。

下面我们就分别来看下这几种收集算法的特点：

- 标记-清除算法

标记-清除算法是最为基础的一种收集算法，算法分为：“标记”和“清除”两个阶段。首先标记出所有需要回收的对象（标记的过程就是上面介绍过的根节点可达算法），在标记完后统一回收所有被标记对象占用的内存空间。

这种收集算法的优点是简单直接，不会影响JVM进程的正常运行。而其缺点也是非常明显，首先，这样的回收方式会产生大量不连续的内存碎片，不利于后续连续内存的分配；其次，这种方式的效率也不高。

- 标记-复制-清除算法

这种算法的思路是将可用的内存空间按容量划分为大小相等的两块，每次只使用其中一块。当这一块使用完了，就将还存活着的对象复制到另外一块上面（移动堆顶指针，按顺序分配内存），然后再把已使用过的内存空间一次清理掉。

示意图如下：

<table><tr>
<td><img src="https://imgconvert.csdnimg.cn/aHR0cHM6Ly91cGxvYWQtaW1hZ2VzLmppYW5zaHUuaW8vdXBsb2FkX2ltYWdlcy8xNTQ2MjA1Ny0yMDdhYmFjNDIxMTRjZDZhLmpwZw?x-oss-process=image/format.png" alt="标记-复制-清除算法" style="zoom:50%;" /></td>
<td><img src="https://imgconvert.csdnimg.cn/aHR0cHM6Ly91cGxvYWQtaW1hZ2VzLmppYW5zaHUuaW8vdXBsb2FkX2ltYWdlcy8xNTQ2MjA1Ny0zNDYwMzA5OGNiZTM5NzlhLmpwZw?x-oss-process=image/format.png" alt="标记-复制-清除算法" style="zoom:50%;" /></td>
</tr></table>


这种收集方式比较好的解决了效率和内存碎片的问题，但是会浪费掉一般的内存空间。目前此种算法主要用于新生代回收。

因为新生代的中98%的对象都是很快就需要被回收的对象，这一点大家在编程时可以体会到，所以并不需要1:1的比例来划分内存空间，在新生代中JVM是按照“8:1:1”的比例来将整个新生代内存划分为一块较大的Eden区和两块较小的Survivor区（S0、S1）。

每次使用Eden区和其中一个Survivor区，当发生回收时将Eden区和Survivor区中还存活的对象一次性复制到另一块Survivor区上，最后清理掉Eden区和刚才使用过的Survivor区。理想情况下，每次新生代中的可用空间是整个新生代容量的90%（80%+10%），只会有10%的内存会被浪费。实际情况中，如果另外一个10%的Survivor区无法装下所有还存活的对象时，就会将这些对象直接放入老年代空间中（这块在后面的分代回收算法会说到，这里先了解下）。

- 标记-清除-整理算法

如果在对象存活率较高的情况下，仍然采用复制算法的话，因为要进行较多的复制操作，效率就会变得很低，而且如果不想浪费50%的内存空间的话，就还需要额外的空间进行分配担保****，以应对存活对象超额的情况**。**显然老年代不能采用2）中的复制算法。

根据老年代的特点，标记-清除-压缩（简称标记-整理）算法应运而生，这种算法的标记过程仍然与“标记-清除”算法一样，只是后续的步骤不再是直接清除可以回收的对象，而是将所有存活的对象都向一端移动后，再直接清理掉端边界以外的内存。

示意图如下：

<table><tr>
<td><img src="https://imgconvert.csdnimg.cn/aHR0cHM6Ly91cGxvYWQtaW1hZ2VzLmppYW5zaHUuaW8vdXBsb2FkX2ltYWdlcy8xNTQ2MjA1Ny1mZjQ4NTg1OTQ5NTJmMGY3LmpwZw?x-oss-process=image/format.png" alt="标记-清除-整理算法" style="zoom:50%;" /></td>
<td><img src="https://imgconvert.csdnimg.cn/aHR0cHM6Ly91cGxvYWQtaW1hZ2VzLmppYW5zaHUuaW8vdXBsb2FkX2ltYWdlcy8xNTQ2MjA1Ny00OWJmZDZjNDY3ODg1NjUzLmpwZw?x-oss-process=image/format.png" alt="标记-清除-整理算法" style="zoom:50%;" /></td>
</tr></table>

4）分代回收算法

实际上在讲解复制算法时已经涉及到了分代回收的内容，这种算法根据对象存活周期的不同将内存划分为几块，Java中主要是新生代、年老代**。这样就可以根据各个年代的特点，采用合适的收集算法了，在文顶的图中已经标示，新生代采用了复制算法，而老年代采用了整理算法，这里就不再赘述。**

### HotSpot Garbage Collectors垃圾收集器

![JVM虚拟机HotSpot内存科普](https://javakk.com/wp-content/uploads/2020/09/hotspot_metaspace4.png)

> ```shell
> -JAVA_OPTS=-Xms4g -Xmx4g -Xmn2g -XX:MaxDirectMemorySize=1g -XX:-OmitStackTraceInFastThrow -XX:+UseG1GC -XX:G1ReservePercent=25 -XX:InitiatingHeapOccupancyPercent=30 -XX:SoftRefLRUPolicyMSPerMB=0 -XX:SurvivorRatio=8 -XX:+DisableExplicitGC -verbose:gc -Xloggc:/opt/gc_%p.log -XX:+PrintGCDetails -XX:+PrintGCDateStamps -XX:+PrintGCApplicationStoppedTime -XX:+PrintAdaptiveSizePolicy -XX:+UseGCLogFileRotation -XX:NumberOfGCLogFiles=5 -XX:GCLogFileSize=30m
> ```

#### 热点垃圾收集类型

**Young Generation年轻代系列**

- **Serial（新生代-串行-收集器）**是一个Stop-the-World，复制收集器(复制算法)，使用单个GC线程
  - Stop-the-World机制，简称STW，即在执行垃圾收集算法时,Java应用程序的其他所有除了垃圾收集收集器线程之外的线程都被挂起。
  - 高并发的场景下不能采用，所有不做深入了解。

其工作示意图如下：

![img](https:////upload-images.jianshu.io/upload_images/15462057-442a9338f2e3696b.jpg?imageMogr2/auto-orient/strip|imageView2/2/w/1200/format/webp)

- **ParNew（新生代-并行-收集器）**是一个Stop-the-World，复制使用多个GC线程的收集器

  - Parnew是Serial的多线程版本，除多线程收集外，其余部分与Serial相比并没有多大的差别，由于其可以与CMS收集器配合使用，所以在JDK1.7之前，对于Java服务应用来说是首选的新生代收集器。

  - 与Serial一样是一个Stop-the-World，只是它采取了多线程回收，所以回收速度会比Serial快，从而缩短卡顿时间。

    示意图如下：

    ![img](https:////upload-images.jianshu.io/upload_images/15462057-50f0d6917e663ed9.jpg?imageMogr2/auto-orient/strip|imageView2/2/w/1200/format/webp)

- **Parallel Scavenge （新生代-并行-收集器）**是一个停止世界，复制使用多个GC线程的收集器

  - Parallel Scavenge也是一个新生代收集器，采用的也是多线程，以及标记-复制-清除算法，与其他同类型收集器不同的是，它的关注点是达到一个可控制吞吐量的目标，**（吞吐量=运行用户代码时间/(运行用户代码时间+垃圾回收的时间)，**假设虚拟机总共运行了100分钟，其中垃圾回收花了一分钟，那么吞吐量就是99%。高吞吐量的目的是为了高效的利用CPU时间，从而尽快的完成程序运算任务，主要适合后台运算不需要有太多交互的应用场景。

    该收集器提供的控制参数有：

    - **MaxGCPauseMillis**：GC时间的最大值。

    - **GCTimeRatio**：GC时间占用总时间的比例。

    - **UseAdaptiveSizePolicy**：这个参数则是开启GC内存分配的自适应调整策略。可以自动调节，新生代的大小、Eden与Survivor的比例、晋升老年代对象的年龄。

**Old Generation老年代**

- **Serial Old**是一个使用单个GC线程的Stop-the-World、mark-sweep紧凑型收集器，采用标记-整理算法的收集器。
  - [Mark-Sweep](https://zhuanlan.zhihu.com/p/87790329)先扫描整个 Heap，标出可到达对象，然后执行 sweep 操作回收不可到达对象。(遍历图标记可达对象,清除没有被标记可达的对象)

- **CMS（Concurrent Mark-Sweep/年老代-并行-收集器）**

CMS是一款以**获取最短回收停顿时间**为目标的收集器(多并发、低暂停)。比较适合互联网响应式应用场景，采用的是**“标记-清除”**算法。

其收集过程如下图所示：

![img](https:////upload-images.jianshu.io/upload_images/15462057-b85e962922858c89.jpg?imageMogr2/auto-orient/strip|imageView2/2/w/1200/format/webp)

如图所示，在初始标记和重新标记两个步骤，也会和Serial 一样暂停所有用户线程。

- Parallel Old是一个使用多个GC线程的压缩收集器

**G1是大型堆的第一个垃圾收集器，提供可靠的GC暂停**，其采用的收集算法是标记-清除-整理算法

#### 如何启用收集器

- UseSerialGC：Serial + Serial Old

- UseParNewGC：ParNew + Serial Old

- [-XX:+UseParallelGC](https://docs.oracle.com/javase/8/docs/technotes/guides/vm/gctuning/collectors.html) : Parallel Collector并联采集器

- -XX:+UseConcMarkSweepGC 使用CMS垃圾回收器

- 使用concmarksweepgc:ParNew+CMS+Serial Old。CMS是大多数是时候收集老一代了。当一个并发的发生模式故障。

- UseParallelGC：Parallel Scavenge + Parallel Old

- 两代都使用G1 GC

<table><tr>
<td><img src="https://javakk.com/wp-content/uploads/2020/09/hotspot_metaspace5.png" alt="JVM虚拟机HotSpot内存科普" style="zoom:50%;" /></td>
<td><img src="https://javakk.com/wp-content/uploads/2020/09/hotspot_metaspace6.png" alt="JVM虚拟机HotSpot内存科普" style="zoom:50%;" /></td>
</tr></table>

> [《一张图看懂JVM之垃圾回收器详解》](https://www.jianshu.com/p/db158c758f67)原文连接没有找到，这里放一个转载链接
>
> [深入理解JVM虚拟机3：垃圾回收器详解](https://blog.csdn.net/a724888/article/details/78764006)
