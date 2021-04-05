# String

## String中的享元模式

享元模式（Flyweight）可以粗略的理解为缓存（cache），是设计中的一种优化策列。

### 常量与常量池

​		在这里需要引入常量池这个简单的概念。**常量池(constant pool)指的是在编译期被确定，并被保存在已编译的.class文件中的一些数据。它包括了关于类、方法、接口等中的常量，也包括字符串常量。**

用例1：

```java
String a0 = "String";
String a1 = "String";
String a2 = "Str" + "ing";
String a3 = new String("String");
String a4 = new String("String");
System.out.println(a0 == a1);
System.out.println(a0 == a2);
System.out.println(a0 == a3);
System.out.println(a3 == a4);

a3.intern();
a4 = a4.intern();
System.out.println(a0 == a3);
System.out.println(a3 == a3.intern());
System.out.println(a0 == a4);

输出结果：
true
true
false
false
false
false
true
```

​		首先，我们要知结果Java会确保一个字符串常量只有一个拷贝。因为例子中的**a0和a1都是字符串常量，它们在编译期就被确定了**，所以a0==a1为true；
**而”Str”和”ing”也都是字符串常量，当一个字符串由多个字符串常量连接而成时，它自己肯定也是字符串常量，所以a2也同样在编译期就被解析为一个字符串常量，所以as2也是常量池中”String”的一个引用**。所以我们得出a0==a1==a2;到这里我们就可以理解string通过这种常量池中相同常量共享对象的方式实现了类似于缓存的享元模式设计。**用new String（）方式创建的String对象，每次都开辟了新的内存空间。**

### String的构造方法

用new String() 创建的字符串不是常量，不能在编译期就确定，所以new String() 创建的字符串不放入常量池中，它们有自己的地址空间。

a0还是常量池中”String”的应用，a3、a4因为无法在编译期确定，所以是运行时创建的新对象”String”的引用。

### String.intern()扩充常量池：

存在于.class文件中的常量池，在运行期被JVM装载，并且可以扩充。String的intern()方法就是扩充常量池的一个方法；

当一个String实例a3调用intern()方法时，Java查找常量池中是否有相同Unicode的字符串常量，如果有则返回其的引用，如果没有，则在常量池中增加一个Unicode等于a3的字符串并返回它的引用；

这里需要说明另外一点：

如果常量池中一开始是没有”String”的，当我们调用a3.intern()后就在常量池中新添加了一个”String”常量，原来的不在常量池的”String”地址仍然存在，a3 == a3.intern()为false，说明原本的”String”地址仍然存在,也就不是“将自己的地址注册到常量池中”了。

### 享元模式总结

> 享元模式能够解决重复对象的内存浪费的问题，当系统中有大量相似对象，需要缓冲池时。不需一直创建新对象，可以从缓冲池里拿。这样可以降低系统内存，同时提高效率。经典的应用场景就是池技术，String常量池、数据库连接池、缓冲池等等都是享元模式的应用，享元模式是池技术的重要实现方式。享元模式使得系统更加复杂。为了使对象可以共享，需要时刻管理对象的状态变化，这使得程序的逻辑变得复杂。

## 源码阅读

- String类被**`final`**修饰无法被继承

### 类图

- 实现了`java.io.Serializable, Comparable<T>, CharSequence`接口

  - java.io.Serializable ：序列化接口没有任何方法和域，仅用于标识序列化的语意。

  - Comparable<Ts> ：这个接口只有一个compareTo(T o)方法 对两个实例化对象比较大小。

  - CharSequence ：这个接口是一个只读的字符序列。包括length(), charAt(int index), subSequence(int start, int end)这几个API接口，值得一提的是，StringBuffer和StringBuild也是实现了该接口。

<div align=center>
<img src="https://i.loli.net/2021/02/27/B6pfaRw9D7o4Ji2.png" alt="String 继承接口" style="zoom:100%;" />
</div>

### 主要属性

- `private final char value[];` 
  - 可以看到String类里声明了一个私有的被final修饰的char类型的数组（即它不能指向另一个对象,但对象本身的修改不受限制）
  - 是string的值，所以string字符串声明后值属性是不可变的

- `private int hash; `
  - 缓存字符串的HashCode，默认值为0

- `private static final long serialVersionUID = -6849794470754667710L;`
  - 使用 serialVersionUID 来完成互操作性

- `private static final ObjectStreamField[] serialPersistentFields = new ObjectStreamField[0];`
  - Class String 在序列化流协议中是区分特殊大小写的

### 构造函数

```java
//无参构造函数，这里就不提了，一般没什么用，因为value是不可变量
//参数为String类型
public String(String original) {
    this.value = original.value;
    this.hash = original.hash;
}
//参数为char数组，使用java.utils包中的Arrays类复制
public String(char value[]) {
    this.value = Arrays.copyOf(value, value.length);
}
//从bytes数组中的offset位置开始，将长度为length的字节，以charset格式编码，拷贝到value
public String(byte bytes[], int offset, int length, Charset charset) {
    if (charset == null)
        throw new NullPointerException("charset");
    checkBounds(bytes, offset, length);
    this.value =  StringCoding.decode(charset, bytes, offset, length);
}
```

### 方法

```java
/*
* 自身对象字符串长度 len1
* 被比较字符串长度 len2
* 取两个字符串的最小值lim
* 从value的第一个字符开始到最小长度lim处为止，如果字符不相等，返回自身（对象不相等处字符，被比较对象不相等字符）
* 如果前面都相等，则返回（自身长度 - 被比较长度）
*/
public int compareTo(String anotherString) {
    int len1 = value.length;
    int len2 = anotherString.value.length;
    int lim = Math.min(len1, len2);
    char v1[] = value;
    char v2[] = anotherString.value;

    int k = 0;
    while (k < lim) {
        char c1 = v1[k];
        char c2 = v2[k];
        if (c1 != c2) {
            return c1 - c2;
        }
        k++;
    }
    return len1 - len2;
}
/*
* 如果引用的是同一个对象，返回真
* 如果不是String类型的数据,或者长度不一致，返回假
* 从后往前单个字符判断，如果有不相等，返回假
* 否则返回真
*/
public boolean equals(Object anObject) {
    if (this == anObject) {
        return true;
    }
    if (anObject instanceof String) {
        String anotherString = (String)anObject;
        int n = value.length;
        if (n == anotherString.value.length) {
            char v1[] = value;
            char v2[] = anotherString.value;
            int i = 0;
            while (n-- != 0) {
                if (v1[i] != v2[i])
                    return false;
                i++;
            }
            return true;
        }
    }
    return false;
}
/*
* 找到字符串前段没有空格的位置
* 找到字符串末尾没有空格的位置
* 如果前后都没有出现空格，返回字符串本身
*/
public String trim() {
    int len = value.length;
    int st = 0;
    char[] val = value;    /* avoid getfield opcode */

    while ((st < len) && (val[st] <= ' ')) {
        st++;
    }
    while ((st < len) && (val[len - 1] <= ' ')) {
        len--;
    }
    return ((st > 0) || (len < value.length)) ? substring(st, len) : this;
}
/*
* @param  delimiter  
* @param  elements  两个join方法形参（1）Iterable 是泛型接口，Iterable<T>是针对 T 类型的 Iterable， 
* T 可以任何引用类型。Iterable<? extends CharSequence>  是要求 T 必须实现 CharSequence，比如说Iterable<String>。
* （2）如果是形参里面出现，表示的是可变参数，比如：CharSequence... elements 表示的传入的参数可以随意，传多少个参数都被放到一个数组里面。
* 判空，创建StringJoiner对象，遍历数组或集合，拼接，返回字符串
*/
public static String join(CharSequence delimiter,
                          Iterable<? extends CharSequence> elements) {
    Objects.requireNonNull(delimiter);
    Objects.requireNonNull(elements);
    StringJoiner joiner = new StringJoiner(delimiter);
    for (CharSequence cs: elements) {
        joiner.add(cs);
    }
    return joiner.toString();
}
public static String join(CharSequence delimiter, CharSequence... elements) {
    Objects.requireNonNull(delimiter);
    Objects.requireNonNull(elements);
    // Number of elements not likely worth Arrays.stream overhead.
    StringJoiner joiner = new StringJoiner(delimiter);
    for (CharSequence cs: elements) {
        joiner.add(cs);
    }
    return joiner.toString();
}
```
### String类的其他方法
<div align=center>
<img src="https://i.loli.net/2021/02/27/Llzfyw2PTdmN7A8.png" alt="String方法" style="zoom:100%;" />
</div>

## 补充

- 关于equals() & ==

  equals()对于String简单来说就是比较两字符串的Unicode序列是否相当，如果相等返回true;而==是比较两字符串的地址是否相同

，也就是是否是同一个字符串的引用。

- 关于String是不可变的
  - String的实例一旦生成就不会再改变了，比如说：String str=”kv”+”ill”+” “+”ans”;
  - 就是有4个字符串常量，首先”kv”和”ill”生成了”kvill”存在内存中，然后”kvill”又和” “ 生成 ”kvill “存在内存中，
  - 最后又和生成了”kvill ans”;并把这个字符串的地址赋给了str,就是因为String的“不可变”产生了很多临时变量，
  - 这也就是为什么建议用StringBuffer的原因了，因为StringBuffer是可改变的。 

- 允许String对象缓存HashCode
  
- Java中String对象的哈希码被频繁地使用, 比如在hashMap 等容器中。字符串不变性保证了hash码的唯一性,因此可以放心地进行缓存.这也是一种性能优化手段,意味着不必每次都去计算新的哈希码.
  
- 安全性
  
- String被许多的Java类(库)用来当做参数,例如 网络连接地址URL,文件路径path,还有反射机制所需要的String参数等,假若String不是固定不变的,将会引起各种安全隐患。
  
- 线程安全
  
- 因为字符串是不可变的，所以是多线程安全的，同一个字符串实例可以被多个线程共享。这样便不用因为线程安全问题而使用同步。字符串自己便是线程安全的。
  
- 总体来说, String不可变的原因包括 设计考虑,效率优化问题,以及安全性这三大方面.

- 如何实现一个不可变类

  既然不可变类有这么多优势，那么我们借鉴String类的设计，自己实现一个不可变类。不可变类的设计通常要遵循以下几个原则：

  - 将类声明为final，所以它不能被继承。
  - 将所有的成员声明为私有的，这样就不允许直接访问这些成员。
  - 对变量不要提供setter方法。
  - 将所有可变的成员声明为final，这样只能对它们赋值一次。
  - 通过构造器初始化所有成员，进行深拷贝(deep copy)。
  - 在getter方法中，不要直接返回对象本身，而是克隆对象，并返回对象的拷贝。
  
- 后来发现的Hollis大佬的文章  [我终于搞清楚了和String有关的那点事儿](https://www.hollischuang.com/archives/2517)