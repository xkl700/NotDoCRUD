
# JVM内存模型（JDK8 HotSpot）

**JVM内存模型 - 需要掌握：JVM运行时数据区、类加载子系统、垃圾回收器**

## 类加载子系统

### 类加载子系统工作流程
在Java虚拟机中，负责查找并封装类的部分称为类加载子系统，类加载子系统用于定位和加载编译后的class文件；
**类加载会将类的信息加入到方法区（元数据空间 以下简称元空间）**。

**字节码 -> 类加载 -> 链接(验证 -> 准备 -> 解析) -> 初始化 -> 使用 -> 卸载**

<div align=center>
	<img src="https://i.loli.net/2021/03/25/aOMlNTVLCY4QfFE.png" alt="类加载的生命周期" style="zoom:100%;" />
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
  <td><img src="https://i.loli.net/2021/03/25/3DPoNKVr768Rmwh.png" alt="方法的调用" style="zoom:150%;" /></td>
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
- 方法区在JDK1.8称为元空间（Metaspace）,元空间与堆不相连，但与堆共享物理内存

**特点：**

- 线程共享
- 存储 类信息、常量、运行时常量池、静态变量、即时编译器编译后的代码等数据（字符串常量池在堆中）
- HotSpot虚拟机上将方法区叫永久代（1.7之前）
- 垃圾收集器很少光顾该区域（无GC回收）
- 方法区1.7及之前通过`-XX:MaxPermSize`设置最大值 ，元空间1.8是`-XX:MaxMetaspaceSize=48m`最大可分配大小 ,`-XX:MetaspaceSize`设置元空间初始大小
- 空间不够分配时`OutOfMemoryError`

#### 方法区内存溢出案例演示

- `-XX:PrintGCDetails -xx:PrintGCDateStamps -XX:UseCompressedClassPointers -XX:MetaspaceSize=10M -XX:MaxMetaspaceSize=10M`
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

#### 对象的访问过程

- 使用对象时，我们是通过栈上的reference引用来操作堆上的具体对象;

- Sun Hotspot虚拟机使用直接指针访问具体对象；

  ![image-20210325154428785](https://i.loli.net/2021/03/25/kxBr27EehQRoPH4.png)
  
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

### 垃圾回收算法

- 标记-清除算法
- 标记-整理算法
- 复制算法标记
- 分代收集算法

### 七种垃圾收集器

![image-20210325154630004](https://i.loli.net/2021/03/25/iua97X1qhK8wTBW.png)

