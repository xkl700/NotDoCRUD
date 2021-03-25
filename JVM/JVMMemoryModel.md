
## JVM内存模型（JDK8 HotSpot）

**JVM内存模型 - 需要掌握：JVM运行时数据区、类加载子系统、垃圾回收器**

### 类加载子系统

#### 类加载子系统工作流程
在Java虚拟机中，负责查找并封装类的部分称为类加载子系统，类加载子系统用于定位和加载编译后的class文件；
$\color{#00FF00}{类加载会将类的信息加入到方法区（元数据空间 以下简称元空间）}$

**字节码 -> 类加载 -> 链接(验证 -> 准备 -> 解析) -> 初始化 -> 使用 -> 卸载**

<div align=center>
	<img src="https://i.loli.net/2021/03/25/aOMlNTVLCY4QfFE.png" alt="类加载的生命周期" style="zoom:100%;" />
    <p>注意：步骤是这么几个步骤，顺序不是绝对的，例如：多态，程序在运行时才知道对象和类是什么类型的，此时解析和初始化顺序可能会发生颠倒</p>
    <p>类加载的时机：按需加载</p>
</div>

#### 类加载器

通过一个类的全限定名来获取描述此类的二进制字节流；

类加载器主要实现类的加载；

```tex
com.xxx.Test --> com.xxx.Test.class --> Class<Test>
java.long.String --> java.long.String.class --> Class<String>
```


#### 双亲委派模型

- Bootstrap ClassLoader 启动类加载器（C++ 实现，是虚拟机的一部分）；（以下简称BC）
- Extension ClassLoader 扩展类加载器（Java实现，独立于虚拟机外部且全继承自java.lang.ClassLoader）；（以下简称EC）
- Application ClassLoader 应用程序加载器（Java实现）；（以下简称AC）
- Custom ClassLoader 自定义类加载器（Java实现，用户自定义的）；（以下简称CC）

除了顶层的启动器加载器之外，其他类加载器都有自己的父类加载器；

![双亲委派模型](https://i.loli.net/2021/03/25/frpEPOsVlbqHJ98.png)

eg: 自定义一个String类，它是如何加载的？

​	不能加载，由AC开始检查是否可以加载，一层一层向上委托（AC -> EC ->BC）到BC时，BC中由于已经加载`java.long.String`类所以一层层向下（BC -> EC ->AC）告知加载器不能在加载自定义String类.

- 类加载器工作过程：如果一个类加载器收到一个类加载请求，他首先不会自己加载，而是把这个请求委派给父类加载器，只有弗雷加载器无法完成加载时子类加载器才会尝试加载。

### JVM运行时数据区

JVM在运行时会把它所管理的内存划分为若干不同的数据区域，总共是五个区域，宏观上划分为两大块：

- 线程私有数据区
- 线程公有数据区

![JVM运行时数据区域](https://i.loli.net/2021/03/25/3ZiP8voyxAQS7rK.png)

#### 线程私有数据区 -- 程序计数器

- 记录程序执行位置、行号
- 一块很小的区域
- 线程私有
- 不存在OutOfMemoryError
- 无GC回收

#### 线程私有数据区 -- 虚拟机栈

虚拟机栈是采用了一种栈的数据结构，入口和出口只有一个，分为入栈和出栈，先进后出；虚拟机栈主要是执行方法。

- 局部变量表 ： 是一组变量值存储空间，用于存放方法参数和方法内定义的局部变量。

- 操作数栈： 也叫操作栈，他是一个先进后出的栈。

- 动态链接：一个方法要调用其他方法，需要将这些方法的符号引用转化为其内存地址中的直接引用。

- 返回地址：方法不管是正常执行结束还是异常退出，需要返回方法被调用的位置。

  <table><tr>
  <td><img src="https://i.loli.net/2021/03/25/81i9MUrTYsqQGvE.png" alt="虚拟机栈" style="zoom:80%;" /></td>
  <td><img src="https://i.loli.net/2021/03/25/3DPoNKVr768Rmwh.png" alt="方法的调用" style="zoom:150%;" /></td>
  </tr></table>


![image-20210325143815086](https://i.loli.net/2021/03/25/uVEsFNCZX2ogqI8.png)

#### 线程私有数据区 -- 本地方法栈

- 与虚拟机栈基本类似
- 区别在于本地方法栈为Native方法服务
- Sun HotSpot将虚拟机栈和本地方法栈合并
- 有`StackOverfFowError`和`OutOfMemoryError`(比较少见)
- GC不会回收该区域

$\color{#00CD66}{栈的OutOfMenmoryError溢出一般是在多线程条件下可能会产生，建立过多的线程，每个线程的运行时间又比较长，可能产生栈的OutOfMemoryError溢出；}$

单个线程下，无论是由于栈帧太大还是虚拟机容量太小，当内存无法分配时，虚拟机抛出`StackOverFlowFrror`的错误异常；

#### 线程私有部分的整体特征（小结）

是线程私有的，随着线程执行结束而结束（JVM就销毁了虚拟机栈里面的栈帧），是比较有规律的，问题会少一些，出问题比较多的是线程共享的部分，也是堆和方法区（元空间）

#### 线程共享部分 -- 方法区（元空间）

- 方法区（jdk1.7后的某个版本合并到了堆）
- 方法区在JDK1.8称为元空间（Metaspace）,元空间与堆不相连，但与堆共享物理内存

**特点：**

- 线程共享
- 存储 类信息、常量、运行时常量池、静态变量、即时编译器编译后的代码等数据
- HotSpot虚拟机上将方法区叫永久代（1.7之前）
- 垃圾收集器很少光顾该区域（无GC回收）
- 方法区1.7及之前通过`-XX:MaxPermSize`设置最大值 ，元空间1.8是`-XX:MaxMetaspaceSize=48m`
- 空间不够分配时`OutOfMemoryError`

#### 方法区内存溢出案例演示

- `-XX:PrintGCDetails -xx:PrintGCDateStamps -XX:UseCompressedClassPointers -XX:MetaspaceSize=10M -XX:MaxMetaspaceSize=10M`
- 第一个参数用于打印GC日志
- 第二个参数用于打印对应的时间戳
- 第三个参数`-XX:UseCompressedClassPointers`表示在Metaspace中不要开辟出一块新的空间（Compressed Class Space），如果开辟这块空间的话空间默认大小是1G,所以我们关闭该功能，此时在设置Metaspace的大小

![image-20210325152306704](https://i.loli.net/2021/03/25/8o37bBZuqXxsGaz.png)

#### 堆

- 线程共享
- 内存中最大的区域
- 虚拟机启动时创建
- 存放所有实例对象或数组
- GC垃圾收集器的主要管理区域
- 可分为新生代、老年代
- 新生代更细化可分为Eden、From Survivor、To Survivor,Eden:Survivor = 8：1：1
- 可通过`-Xmx`、`-Xms`调节堆大小
- 无法再扩展`OutOfMemoryError`，`java.lang.OutOfMenmoryError:Java heap space`

![image-20210325153549190](https://i.loli.net/2021/03/25/Un48XArRoqsNzuV.png)

##### Java堆溢出

不断创建对象又不是放，当对象到达一定数量，无堆空间将产生堆内存溢出

- 内存泄漏：GC Roots到对象之间有可达路径而无法收集；

- 内存溢出：GC Roots到对象之间无可达路径，可以被收集，但对象还需要存活着，此时可根据物理机内存适当调大虚拟机参数`-Xms`、`-Xmx`,分析代码中对象生命周期是否过长、对象是否持有状态时间过长；

##### 对象的访问过程

- 使用对象时，我们是通过栈上的reference引用来操作堆上的具体对象;

- Sun Hotspot虚拟机使用直接指针访问具体对象；

  ![image-20210325154428785](https://i.loli.net/2021/03/25/kxBr27EehQRoPH4.png)

## GC

### 垃圾回收算法

- 标记-清除算法
- 标记-整理算法
- 复制算法标记
- 分代收集算法

### 七种垃圾收集器

![image-20210325154630004](https://i.loli.net/2021/03/25/iua97X1qhK8wTBW.png)

