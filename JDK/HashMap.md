# HashMap

## 一、定义
```java
public class HashMap<K,V> extends AbstractMap<K,V> implements Map<K,V>, Cloneable, Serializable
```
![HashMap类图](https://i.loli.net/2021/04/02/IKhOHRde5P1bzZs.png)
### 继承了  AbstractMap<K,V>

AbstractMap提供了Map接口的实现，其中实现了Map中最重要最常用和方法，这样HashMap继承AbstractMap就不需要实现Map的所有方法，让HashMap减少了大量的工作。

### 实现了Map<K,V>, Cloneable, Serializable

HashMap 实现了Serializable接口，可以序列化。实现了Cloneable，可以被克隆。

实现Map<K,V>,看到这我有点蒙了，它既然继承了 AbstractMap<K,V> 为什么还要实现Map<K,V>？

在[Why does LinkedHashSet<E> extend HashSet<E> and implement Set<E>](https://stackoverflow.com/questions/2165204/why-does-linkedhashsete-extend-hashsete-and-implement-sete)中提到 :
>  I've asked Josh Bloch, and he informs me that it was a mistake. He used to think, long ago, that there was some value in it, but he since "saw the light". Clearly JDK maintainers haven't considered this to be worth backing out later.

这是类库设计者的拼写错误，其实HashMap不应实现Map的。其他容器如List、Set也有这个问题。

## 成员变量和常量

```java
    //默认初始容量
    static final int DEFAULT_INITIAL_CAPACITY = 1 << 4; // aka 16
    //最大值容量
    static final int MAXIMUM_CAPACITY = 1 << 30;
    //默认负载因子
    static final float DEFAULT_LOAD_FACTOR = 0.75f;
    // 链表节点转换红黑树节点的阈值, 9个节点转
    static final int TREEIFY_THRESHOLD = 8;
    // 红黑树节点转换链表节点的阈值, 6个节点转
    static final int UNTREEIFY_THRESHOLD = 6;
    // 转红黑树时, table的最小长度
    static final int MIN_TREEIFY_CAPACITY = 64; 

    //Entry类型的数组,HashMap用来维护内部的数据结构,长度由容量决定
    transient Entry<K,V>[] table;
    //保存缓存EntrySet()。注意，keyset()和values()使用了AbstractMap字段。
    transient Set<Map.Entry<K,V>> entrySet;
    //HashMap的大小
    transient int size;
    //HashMap的极限容量,容量乘以负载因子,是扩容的临界点
    int threshold;
    //手动设置的负载因子（当前 HashMap 所能容纳键值对数量的最大值，超过这个值，则需扩容）
    final float loadFactor;
    //修改次数
    transient int modCount;
```

## 构造函数

```java
    //构造一个带指定初始容量和负载因子的空 HashMap 
    public HashMap(int initialCapacity, float loadFactor) {
        if (initialCapacity < 0)
            throw new IllegalArgumentException("Illegal initial capacity: " +
                                               initialCapacity);
        if (initialCapacity > MAXIMUM_CAPACITY)
            initialCapacity = MAXIMUM_CAPACITY;
        if (loadFactor <= 0 || Float.isNaN(loadFactor))
            throw new IllegalArgumentException("Illegal load factor: " +
                                               loadFactor);
        this.loadFactor = loadFactor;
        this.threshold = tableSizeFor(initialCapacity);
    }

    // 构造一个带指定初始容量和默认负载因子 (0.75) 的空 HashMap
    public HashMap(int initialCapacity) {
        this(initialCapacity, DEFAULT_LOAD_FACTOR);
    }

    // 构造一个具有默认初始容量 (16) 和默认负载因子 (0.75) 的空 HashMap
    public HashMap() {
        this.loadFactor = DEFAULT_LOAD_FACTOR; // all other fields defaulted
    }

    /**
     * 构造一个映射关系与指定 Map 相同的新 HashMap
     *
     * @param   m the map whose mappings are to be placed in this map
     * @throws  NullPointerException if the specified map is null
     */
    public HashMap(Map<? extends K, ? extends V> m) {
        this.loadFactor = DEFAULT_LOAD_FACTOR;
        putMapEntries(m, false);
    }
```
在 HashMap 构造方法中，可供我们调整的参数有两个，一个是初始容量 initialCapacity，另一个负载因子 loadFactor。
通过这两个设定这两个参数，可以进一步影响阈值大小。但初始阈值 threshold 仅由 initialCapacity 经过移位操作计算得出。

默认情况下，HashMap 初始容量是16，负载因子为 0.75。这里并没有默认阈值，原因是阈值可由容量乘上负载因子计算而来（注释中有说明），
即threshold = capacity * loadFactor。但当你仔细看构造方法HashMap(int initialCapacity, float loadFactor)时，会发现阈值并不是由上面公式计算而来，而是通过tableSizeFor方法算出来的。
这是不是可以说明 threshold 变量的注释有误呢？还是仅这里进行了特殊处理，其他地方遵循计算公式呢？关于这个疑问，这里也先不说明，
后面在分析扩容方法时，再来解释这个问题。接下来，我们来看看tableSizeFor方法，源码如下：
```java
    /**
     * Returns a power of two size for the given target capacity.
     */
    static final int tableSizeFor(int cap) {
        int n = cap - 1;
        n |= n >>> 1;
        n |= n >>> 2;
        n |= n >>> 4;
        n |= n >>> 8;
        n |= n >>> 16;
        return (n < 0) ? 1 : (n >= MAXIMUM_CAPACITY) ? MAXIMUM_CAPACITY : n + 1;
    }
```
- 阀值 threshold，通过方法 tableSizeFor 进行计算，是根据初始化来计算的。
- 这个方法也就是要寻找比初始值大的，最小的那个 2 进制数值。比如传了 17，我应该找到的是 32。
- MAXIMUM_CAPACITY = 1 << 30，这个是临界范围，也就是最大的 Map 集合。
- 乍一看可能有点晕,怎么都在向右移位 1、2、4、8、16，这主要是为了把二进制的各个位置都填上 1，当二进制的各个位置都是 1 以后，就是
一个标准的 2 的倍数减 1 了，最后把结果加 1 再返回即可。

在这里顺便看一下负载因子是什么概念？

负载因子，可以理解成一辆车可承重重量超过某个阀值时，把货放到新的车上。

那么在 HashMap 中，负载因子决定了数据量多少了以后进行扩容。即使你数据比数组容量大时也是不一定能正
好的把数组占满的，而是在某些小标位置出现了大量的碰撞，只能在同一个位置用链表存放，那么这样就失去了 Map 数组的性能。
所以，要选择一个合理的大小下进行扩容，默认值 0.75 就是说当阀值容量占了3/4 时赶紧扩容，减少 Hash 碰撞。
同时 0.75 是一个默认构造值，在创建 HashMap 也可以调整，比如你希望用更多的空间换取时间，可以把负载因子调的更小一些，减少碰撞。


## 其他方法
> 我在写这一部分的时候总觉得差点意思，然后从网上翻资料的时候翻到了小傅哥的这三篇文章，主要涉及以下几点：
> 1、散列表实现、2、扰动函数、3、初始化容量、4、负载因子、5、扩容元素拆分、6、链表树化、7、红黑树、8、插入、9、查找、10、删除、11、遍历、12、分段锁
> 强烈推荐阅读

[数据结构，HashCode为什么使用31作为乘数？](https://bugstack.cn/interview/2020/08/04/%E9%9D%A2%E7%BB%8F%E6%89%8B%E5%86%8C-%E7%AC%AC2%E7%AF%87-%E6%95%B0%E6%8D%AE%E7%BB%93%E6%9E%84-HashCode%E4%B8%BA%E4%BB%80%E4%B9%88%E4%BD%BF%E7%94%A831%E4%BD%9C%E4%B8%BA%E4%B9%98%E6%95%B0.html)

[HashMap核心知识，扰动函数、负载因子、扩容链表拆分，深度学习](https://bugstack.cn/interview/2020/08/07/%E9%9D%A2%E7%BB%8F%E6%89%8B%E5%86%8C-%E7%AC%AC3%E7%AF%87-HashMap%E6%A0%B8%E5%BF%83%E7%9F%A5%E8%AF%86-%E6%89%B0%E5%8A%A8%E5%87%BD%E6%95%B0-%E8%B4%9F%E8%BD%BD%E5%9B%A0%E5%AD%90-%E6%89%A9%E5%AE%B9%E9%93%BE%E8%A1%A8%E6%8B%86%E5%88%86-%E6%B7%B1%E5%BA%A6%E5%AD%A6%E4%B9%A0.html)

[HashMap数据插入、查找、删除、遍历，源码分析](https://bugstack.cn/interview/2020/08/13/%E9%9D%A2%E7%BB%8F%E6%89%8B%E5%86%8C-%E7%AC%AC4%E7%AF%87-HashMap%E6%95%B0%E6%8D%AE%E6%8F%92%E5%85%A5-%E6%9F%A5%E6%89%BE-%E5%88%A0%E9%99%A4-%E9%81%8D%E5%8E%86-%E6%BA%90%E7%A0%81%E5%88%86%E6%9E%90.html)

[看图说话，讲解2-3平衡树「红黑树的前身」](https://bugstack.cn/interview/2020/08/16/%E9%9D%A2%E7%BB%8F%E6%89%8B%E5%86%8C-%E7%AC%AC5%E7%AF%87-%E7%9C%8B%E5%9B%BE%E8%AF%B4%E8%AF%9D-%E8%AE%B2%E8%A7%A32-3%E5%B9%B3%E8%A1%A1%E6%A0%91-%E7%BA%A2%E9%BB%91%E6%A0%91%E7%9A%84%E5%89%8D%E8%BA%AB.html)

[带着面试题学习红黑树操作原理，解析什么时候染色、怎么进行旋转、与2-3树有什么关联](https://bugstack.cn/interview/2020/08/20/%E9%9D%A2%E7%BB%8F%E6%89%8B%E5%86%8C-%E7%AC%AC6%E7%AF%87-%E5%B8%A6%E7%9D%80%E9%9D%A2%E8%AF%95%E9%A2%98%E5%AD%A6%E4%B9%A0%E7%BA%A2%E9%BB%91%E6%A0%91%E6%93%8D%E4%BD%9C%E5%8E%9F%E7%90%86-%E8%A7%A3%E6%9E%90%E4%BB%80%E4%B9%88%E6%97%B6%E5%80%99%E6%9F%93%E8%89%B2-%E6%80%8E%E4%B9%88%E8%BF%9B%E8%A1%8C%E6%97%8B%E8%BD%AC-%E4%B8%8E2-3%E6%A0%91%E6%9C%89%E4%BB%80%E4%B9%88%E5%85%B3%E8%81%94.html)

### 线程安全性

在多线程使用场景中，应该尽量避免使用线程不安全的HashMap，而使用线程安全的ConcurrentHashMap。

> 首先 HashMap 并发 resize 可能导致循环链表的问题，其实在 1.8 中确实是不存在了，因为 1.7(含)之前 HashMap 链表使用的是头插法，resize 过程中会有顺序倒置，所以才并发时才有这个风险，但是 1.8 改为使用尾插法，已经不会有循环链表的风险。但是 1.8 下 HashMap 下并发时，依然可能出现 length 计算错误，或者节点丢失的问题。所以 1.8 下 HashMap 依然不是线程安全的

# LinkedHashMap

LinkedHashMap 继承自 HashMap，所以它的底层仍然是基于拉链式散列结构。该结构由数组和链表或红黑树组成。
LinkedHashMap 在上面结构的基础上，增加了一条双向链表，使得上面的结构可以保持键值对的插入顺序。同时通过对链表进行相应的操作，实现了访问顺序相关逻辑。

![LinkedHashMap数据结构](![img](https://i.loli.net/2021/04/08/qfXlxrBJ8S9vd4A.png))

上图中，淡蓝色的箭头表示前驱引用，红色箭头表示后继引用。每当有新键值对节点插入，新节点最终会接在 tail 引用指向的节点后面。而 tail 引用则会移动到新的节点上，这样一个双向链表就建立起来了。

## 实现了 Map<K,V> 继承了 HashMap<K,V>

```java
public class LinkedHashMap<K,V> extends HashMap<K,V> implements Map<K,V>
```
可以发现LinkedHashMap extends HashMap已经被动实现了Map,为什么还需要主动 implements Map ？
> 这个问题在前文提到过这里就不再说了

## 变量与常量

```java

    // 双向链表的头（最老）
    transient LinkedHashMap.Entry<K,V> head;

    // 双链表的末尾（最年轻）。
    transient LinkedHashMap.Entry<K,V> tail;

    // 此链接的哈希映射的迭代排序方法：true表示访问顺序， false表示插入顺序。
    final boolean accessOrder;
```

## 构造函数

```java
    public LinkedHashMap(int initialCapacity, float loadFactor) {
        super(initialCapacity, loadFactor);
        accessOrder = false;
    }


    public LinkedHashMap(int initialCapacity) {
        super(initialCapacity);
        accessOrder = false;
    }

    public LinkedHashMap() {
        super();
        accessOrder = false;
    }

    public LinkedHashMap(Map<? extends K, ? extends V> m) {
        super();
        accessOrder = false;
        putMapEntries(m, false);
    }

    public LinkedHashMap(int initialCapacity,
                         float loadFactor,
                         boolean accessOrder) {
        super(initialCapacity, loadFactor);
        this.accessOrder = accessOrder;
    }
```
可以看到LinkedHashMap的构造函数基本上有以下两点：
1. 调用HashMap的构造方法，其实就是根据初始容量、负载因子去初始化Entry[] table
2. accessOrder = false 是否基于访问排序，默认为false（即按插入顺序存储）

```java
    static class Entry<K,V> extends HashMap.Node<K,V> {
        Entry<K,V> before, after;
        Entry(int hash, K key, V value, Node<K,V> next) {
            super(hash, key, value, next);
        }
    }
```

再看一下LinkedHashMap节点的实现，它继承了HashMap.Node<K,V>，并新增了两个引用，分别是before和after。这两个引用的用途不难理解，也就是用于维护双向链表。

### 其他方法
这篇文章我觉得已经讲得很全面了，我在学习的过程中也没看到有其他要注意的点，如果我再复述一次就实在是浪费时间，所以直接在此转载田小波的文章
[LinkedHashMap 源码详细分析（JDK1.8）](http://www.tianxiaobo.com/2018/01/24/LinkedHashMap-%E6%BA%90%E7%A0%81%E8%AF%A6%E7%BB%86%E5%88%86%E6%9E%90%EF%BC%88JDK1-8%EF%BC%89/)

# TreeMap

TreeMap的学习我主要参考了田小波的 [TreeMap源码分析](http://www.tianxiaobo.com/2018/01/11/TreeMap%E6%BA%90%E7%A0%81%E5%88%86%E6%9E%90/) 那下面主要写一下这篇文章里没有详细提到的内容。

TreeMap的底层数据结构就是一个红黑树。关于红黑树的知识可以查看算法--[带着面试题学习红黑树操作原理，解析什么时候染色、怎么进行旋转、与2-3树有什么关联](https://bugstack.cn/interview/2020/08/20/%E9%9D%A2%E7%BB%8F%E6%89%8B%E5%86%8C-%E7%AC%AC6%E7%AF%87-%E5%B8%A6%E7%9D%80%E9%9D%A2%E8%AF%95%E9%A2%98%E5%AD%A6%E4%B9%A0%E7%BA%A2%E9%BB%91%E6%A0%91%E6%93%8D%E4%BD%9C%E5%8E%9F%E7%90%86-%E8%A7%A3%E6%9E%90%E4%BB%80%E4%B9%88%E6%97%B6%E5%80%99%E6%9F%93%E8%89%B2-%E6%80%8E%E4%B9%88%E8%BF%9B%E8%A1%8C%E6%97%8B%E8%BD%AC-%E4%B8%8E2-3%E6%A0%91%E6%9C%89%E4%BB%80%E4%B9%88%E5%85%B3%E8%81%94.html)。

TreeMap的特点就是存储的时候是根据键Key来进行排序的。其顺序与添加顺序无关，该顺序根据key的自然排序进行排序或者根据构造方法中传入的Comparator比较器进行排序。自然排序要求key需要实现Comparable接口。

## 变量
```java
    //比较器，若无则按Key的自然排序
    private final Comparator<? super K> comparator;
    //树根结点
    private transient Entry<K,V> root;
    //树节点个数
    private transient int size = 0;
    //用于判断数据是否变化
    private transient int modCount = 0;
```

复制代码Entry<K,V>表示红黑树的一个结点，既然是红黑树，那么每个节点中除了Key-->Value映射之外，必然存储了红黑树节点特有的一些内容
```java
static final class Entry<K,V> implements Map.Entry<K,V> {
        K key;
        V value;
        Entry<K,V> left;
        Entry<K,V> right;
        Entry<K,V> parent;
        boolean color = BLACK;//黑色表示为true，红色为false
}
```

## 构造函数

```java
    public TreeMap() {
        comparator = null;
    }

    public TreeMap(Comparator<? super K> comparator) {
        this.comparator = comparator;
    }

    public TreeMap(Map<? extends K, ? extends V> m) {
        comparator = null;
        putAll(m);
    }

    public TreeMap(SortedMap<K, ? extends V> m) {
        comparator = m.comparator();
        try {
            buildFromSorted(m.size(), m.entrySet().iterator(), null, null);
        } catch (java.io.IOException cannotHappen) {
        } catch (ClassNotFoundException cannotHappen) {
        }
    }
```

代码比较简单，我们主要关注一下buildFromSorted方法和computeRedLevel方法。TreeMap主要通过这两个方法在初始化的时候构造一个简单的红黑树。


computeRedLevel方法是计算当前结点数的完全二叉树的层数。或者说，着色红色结点的层数。
TreeMap是如何构造红黑树的呢，简单来说，就是把当前的结点按照完全二叉树的结构来排列，此时，最下层的符合二叉树又未满足满二叉树 的那一排结点，
就全部设为红色，这样就满足红黑树的条件了。（TreeMap中第一层根结点层数为0）
![红黑树](https://user-gold-cdn.xitu.io/2018/12/29/167f908725408674?imageslim)

如上图，如果一个树有9个节点，那么我们构造红黑树的时候，只要把前面3层的结点都设置为黑色，第四层的节点设置为红色，则构造完的树，就是红黑树。而实现的关键就是找到要构造树的完全二叉树的层数

```java
        private void buildFromSorted(int size, Iterator<?> it,
                                     java.io.ObjectInputStream str,
                                     V defaultVal)
            throws  java.io.IOException, ClassNotFoundException {
            this.size = size;
            root = buildFromSorted(0, 0, size-1, computeRedLevel(size),
                                   it, str, defaultVal);
        }

    /**
    * level: 当前树的层数，注意：是从0层开始
    * lo: 子树第一个元素的索引
    * hi: 子树最后一个元素的索引
    * redLevel: 上述红节点所在层数
    * it: 传入的map的entries迭代器
    * str: 如果不为空，则从流里读取key-value
    * defaultVal：不为空，则value都用这个值
    */
    @SuppressWarnings("unchecked")
    private final Entry<K,V> buildFromSorted(int level, int lo, int hi,
                                             int redLevel,
                                             Iterator<?> it,
                                             java.io.ObjectInputStream str,
                                             V defaultVal)
        throws  java.io.IOException, ClassNotFoundException {
        /*
         * 策略：根是最中间的元素。为了达到这个目的，我们必须首先递归地构造整个左子树，以获取其所有元素。
         * 然后我们可以继续右子树。
         * lo和hi参数是要从当前子树的迭代器或流中拉出的最小和最大索引。
         *它们实际上没有被索引，我们只是按顺序进行，确保以相应的顺序提取项目。
         */

        // hi >= lo 说明子树已经构造完成
        if (hi < lo) return null;

        // 取中间位置，无符号右移，相当于除2
        int mid = (lo + hi) >>> 1;

        Entry<K,V> left  = null;
        //递归构造左结点
        if (lo < mid)
            left = buildFromSorted(level+1, lo, mid - 1, redLevel,
                                   it, str, defaultVal);

        // extract key and/or value from iterator or stream
        K key;
        V value;
        //递归完左子树后，迭代器的下一个结点就是每棵树或子树的根结点，所以此时获取的key，value就是树根结点的key，value
        if (it != null) {
            if (defaultVal==null) {
                Map.Entry<?,?> entry = (Map.Entry<?,?>)it.next();
                key = (K)entry.getKey();
                value = (V)entry.getValue();
            } else {
                key = (K)it.next();
                value = defaultVal;
            }
        } else { // use stream  通过流来读取key, value
            key = (K) str.readObject();
            value = (defaultVal != null ? defaultVal : (V) str.readObject());
        }
        //构建结点
        Entry<K,V> middle =  new Entry<>(key, value, null);

         // 这里是判断该节点是否是最下层的叶子结点。
        if (level == redLevel)
            middle.color = RED;
        //如果存在的话，设置左结点，
        if (left != null) {
            middle.left = left;
            left.parent = middle;
        }
         // 递归构造右结点
        if (mid < hi) {
            Entry<K,V> right = buildFromSorted(level+1, mid+1, hi, redLevel,
                                               it, str, defaultVal);
            middle.right = right;
            right.parent = middle;
        }

        return middle;
    }

    private static int computeRedLevel(int sz) {
        int level = 0;
        for (int m = sz - 1; m >= 0; m = m / 2 - 1)
            level++;
        return level;
    }

```

## 总结

TreeMap的底层数据结构就是一个红黑树，增删改查和统计相关的操作的时间复杂度都为 O(logn)。所以了解红黑树的数据结构和逻辑就非常重要。
TreeMap的特点就是存储的时候是根据键Key来进行排序的。
key的排序分为自然排序和比较器排序。自然排序要求key需要实现Comparable接口，比较器排序需要在构造方法中传入的Comparator比较器进行排序。

# CouncurrentHashMap
>我们都知道JDK1.7中ConcurrentHashMap使用Segment分段锁的方式保证了线程安全，JDK1.8中抛弃了原有的Segment分段锁，采用了 CAS + synchronized 来保证并发安全性。
>
>在阅读CpuncurrentHashMap前最好先补充一下CAS 和 JMM的知识
>
>[乐观锁的一种实现方式——CAS](https://www.hollischuang.com/archives/1537)
>
>[《成神之路-基础篇》JVM——Java内存模型(已完结)](https://www.hollischuang.com/archives/1003)

## 定义

### 继承了  AbstractMap<K,V>

AbstractMap提供了Map接口的实现，其中实现了Map中最重要最常用和方法，这样HashMap继承AbstractMap就不需要实现Map的所有方法，让HashMap减少了大量的工作。

### 实现了ConcurrentMap<K,V>, Serializable

HashMap 实现了Serializable接口，可以序列化。实现了ConcurrentMap，支持并发访问。

```java
public class ConcurrentHashMap<K,V> extends AbstractMap<K,V>
    implements ConcurrentMap<K,V>, Serializable 
```

## Node

```java
    static class Node<K,V> implements Map.Entry<K,V> {
        final int hash;
        final K key;
        volatile V val;
        volatile Node<K,V> next;

        Node(int hash, K key, V val, Node<K,V> next) {
            this.hash = hash;
            this.key = key;
            this.val = val;
            this.next = next;
        }

        public final K getKey()       { return key; }
        public final V getValue()     { return val; }
        public final int hashCode()   { return key.hashCode() ^ val.hashCode(); }
        public final String toString(){ return key + "=" + val; }
        public final V setValue(V value) {
            throw new UnsupportedOperationException();
        }

        public final boolean equals(Object o) {
            Object k, v, u; Map.Entry<?,?> e;
            return ((o instanceof Map.Entry) &&
                    (k = (e = (Map.Entry<?,?>)o).getKey()) != null &&
                    (v = e.getValue()) != null &&
                    (k == key || k.equals(key)) &&
                    (v == (u = val) || v.equals(u)));
        }

        /**
         * Virtualized support for map.get(); overridden in subclasses.
         */
        Node<K,V> find(int h, Object k) {
            Node<K,V> e = this;
            if (k != null) {
                do {
                    K ek;
                    if (e.hash == h &&
                        ((ek = e.key) == k || (ek != null && k.equals(ek))))
                        return e;
                } while ((e = e.next) != null);
            }
            return null;
        }
    }
```

我们可以看到ConcurrentHashMap与HashMap的存放数据的 Node 是相似的，其中的 val next 都用了 volatile 修饰，保证了可见性。

## 变量

```java
    /* ---------------- Constants -------------- */

    // 最大容量
    private static final int MAXIMUM_CAPACITY = 1 << 30;
    // 默认初始表容量。必须是2的幂（即至少为1），并且最大为MAXIMUM_CAPACITY。
    private static final int DEFAULT_CAPACITY = 16;
    // 最大的数组大小（非2的幂）。 toArray和相关方法需要。
    static final int MAX_ARRAY_SIZE = Integer.MAX_VALUE - 8;
    // 该表的默认并发级别。未使用，但定义为与此类的先前版本兼容。
    private static final int DEFAULT_CONCURRENCY_LEVEL = 16;
    // 默认负载因子
    private static final float LOAD_FACTOR = 0.75f;
    // 链表节点转换红黑树节点的阈值.
    static final int TREEIFY_THRESHOLD = 8;
    // 红黑树节点转换链表节点的阈值
    static final int UNTREEIFY_THRESHOLD = 6;
    // 转红黑树时, table的最小长度
    static final int MIN_TREEIFY_CAPACITY = 64;

    // 每个传输步骤的最小重新绑定数量。 范围被细分以允许多个调整程序线程。 此值用作下限，以避免调整器遇到过多的内存争用。该值至少应为* DEFAULT_CAPACITY。
    private static final int MIN_TRANSFER_STRIDE = 16;
    // sizeCtl中用于生成标记的位数。对于32位阵列，必须至少为6。
    private static int RESIZE_STAMP_BITS = 16;

    // 可以帮助调整大小的最大线程数。必须为32-RESIZE_STAMP_BITS位。
    private static final int MAX_RESIZERS = (1 << (32 - RESIZE_STAMP_BITS)) - 1;

    // 在sizeCtl中记录大小标记的位移位。
    private static final int RESIZE_STAMP_SHIFT = 32 - RESIZE_STAMP_BITS;

    /*
     * 节点哈希字段的编码。请参阅上面的说明。
     */
    static final int MOVED     = -1; // 用于转发节点的哈希
    static final int TREEBIN   = -2; // 散列为树的根
    static final int RESERVED  = -3; // 临时保留的哈希
    static final int HASH_BITS = 0x7fffffff; // 普通节点哈希的可用位

    /** CPUS的数量，以限制某些尺寸 */
    static final int NCPU = Runtime.getRuntime().availableProcessors();

    /* ---------------- Fields -------------- */

    // 垃圾箱数组。第一次插入时延迟进行初始化。大小始终是2的幂。由迭代器直接访问。
    transient volatile Node<K,V>[] table;

    // 要使用的下一张表；仅在调整大小时为非null。
    private transient volatile Node<K,V>[] nextTable;

    // 基本计数器值，主要在没有争用时使用，也用作表初始化期间的回退d争用。通过CAS更新。
    private transient volatile long baseCount;

    /**
     * 表初始化和大小调整控制。如果为负，则表正在初始化或调整大小：-1表示初始化，
     * 否则-（1 +活动的调整大小线程数）。否则，当table为null时，保留创建时要使用的初始表大小，或者默认为0。
     * 初始化后，保留下一个元素计数值，以在该值上调整表的大小。
     */
    private transient volatile int sizeCtl;

    // 在调整大小时要拆分的下一个表索引（加1）
    private transient volatile int transferIndex;

    // 调整大小和/或创建CounterCell时使用的Spinlock（通过CAS锁定）。
    private transient volatile int cellsBusy;

    /**
     * 计数器单元格表。如果为非null，则大小为2的幂。
     */
    private transient volatile CounterCell[] counterCells;

    // views
    private transient KeySetView<K,V> keySet;
    private transient ValuesView<K,V> values;
    private transient EntrySetView<K,V> entrySet;

```

## 构造函数

```java
    /* ---------------- Public operations -------------- */

    // 使用默认的初始表大小（16）创建一个新的空地图。
    public ConcurrentHashMap() {
    }

    // 创建一个新的空映射，其初始表大小可以容纳指定数量的元素，而无需动态调整大小。
    public ConcurrentHashMap(int initialCapacity) {
        if (initialCapacity < 0)
            throw new IllegalArgumentException();
        int cap = ((initialCapacity >= (MAXIMUM_CAPACITY >>> 1)) ?
                   MAXIMUM_CAPACITY :
                   tableSizeFor(initialCapacity + (initialCapacity >>> 1) + 1));
        this.sizeCtl = cap;
    }

    // 使用与给定Map创建一个相同的新Map。
    public ConcurrentHashMap(Map<? extends K, ? extends V> m) {
        this.sizeCtl = DEFAULT_CAPACITY;
        putAll(m);
    }

    // 指定初始容量和负载因子
    public ConcurrentHashMap(int initialCapacity, float loadFactor) {
        this(initialCapacity, loadFactor, 1);
    }

    // 创建一个新的空映射，定初始容量、负载因子和并发数（concurrencyLevel）。

    public ConcurrentHashMap(int initialCapacity,
                             float loadFactor, int concurrencyLevel) {
        if (!(loadFactor > 0.0f) || initialCapacity < 0 || concurrencyLevel <= 0)
            throw new IllegalArgumentException();
        // 定初始容量 < 并发数 时 ， 将定初始容量 = 并发数
        if (initialCapacity < concurrencyLevel)   // Use at least as many bins
            initialCapacity = concurrencyLevel;   // as estimated threads
        // 表空间大小为(1.0 + (long)initialCapacity / loadFactor) ，再去跟MAXIMUM_CAPACITY比较，如果大于MAXIMUM_CAPACITY,则表空间大小等于MAXIMUM_CAPACITY，反之通过tableSizeFor计算实际的表空间大小。
        long size = (long)(1.0 + (long)initialCapacity / loadFactor);
        int cap = (size >= (long)MAXIMUM_CAPACITY) ?
            MAXIMUM_CAPACITY : tableSizeFor((int)size);
        this.sizeCtl = cap;
    }
```

## 其他方法

1. put 方法

```java
    public V put(K key, V value) {
        return putVal(key, value, false);
    }

    /** Implementation for put and putIfAbsent */
    final V putVal(K key, V value, boolean onlyIfAbsent) {
        if (key == null || value == null) throw new NullPointerException();
        // 根据 key 计算出 hashcode 
        int hash = spread(key.hashCode());
        int binCount = 0;
        for (Node<K,V>[] tab = table;;) {
            Node<K,V> f; int n, i, fh;
            // 如果数组还未初始化，那么进行初始化，这里会通过一个CAS操作将sizeCtl设置为-1，设置成功的，可以进行初始化操作
            if (tab == null || (n = tab.length) == 0)
                tab = initTable();
            // f 即为当前 key 定位出的 Node，如果为空表示当前位置可以写入数据，利用 CAS 尝试写入，失败则自旋保证成功。
            else if ((f = tabAt(tab, i = (n - 1) & hash)) == null) {
                //根据key的hash值找到对应的桶，如果桶还不存在，那么通过一个CAS操作来设置桶的第一个元素，失败的继续执行下面的逻辑即向桶中插入或更新
                if (casTabAt(tab, i, null,
                             new Node<K,V>(hash, key, value, null)))
                    break;                   // no lock when adding to empty bin
            }
            // 如果当前位置的 hashcode == MOVED == -1,则需要进行扩容。
            else if ((fh = f.hash) == MOVED)
                tab = helpTransfer(tab, f);
            // 如果都不满足，则利用 synchronized 锁写入数据。
            else {
                V oldVal = null;
                synchronized (f) {
                    if (tabAt(tab, i) == f) {
                        // 如果找到的桶存在，那么要么是链表结构要么是红黑树结构，此时需要获取该桶的锁，在锁定的情况下执行链表或者红黑树的插入或更新
                        // 如果桶中第一个元素的hash值大于0，说明是链表结构，则对链表插入或者更新
                        if (fh >= 0) {
                            binCount = 1;
                            for (Node<K,V> e = f;; ++binCount) {
                                K ek;
                                if (e.hash == hash &&
                                    ((ek = e.key) == key ||
                                     (ek != null && key.equals(ek)))) {
                                    oldVal = e.val;
                                    if (!onlyIfAbsent)
                                        e.val = value;
                                    break;
                                }
                                Node<K,V> pred = e;
                                if ((e = e.next) == null) {
                                    pred.next = new Node<K,V>(hash, key,
                                                              value, null);
                                    break;
                                }
                            }
                        }
                        // 如果桶中的第一个元素类型是TreeBin，说明是红黑树结构，则按照红黑树的方式进行插入或者更新
                        else if (f instanceof TreeBin) {
                            Node<K,V> p;
                            binCount = 2;
                            if ((p = ((TreeBin<K,V>)f).putTreeVal(hash, key,
                                                           value)) != null) {
                                oldVal = p.val;
                                if (!onlyIfAbsent)
                                    p.val = value;
                            }
                        }
                    }
                }
                // 如果数量大于 TREEIFY_THRESHOLD 则要转换为红黑树。
                if (binCount != 0) {
                    if (binCount >= TREEIFY_THRESHOLD)
                        treeifyBin(tab, i);
                    if (oldVal != null)
                        return oldVal;
                    break;
                }
            }
        }
        addCount(1L, binCount);
        return null;
    }
```

- 根据 key 计算出 hashcode 。

- 如果数组还未初始化，那么进行初始化，这里会通过一个CAS操作将sizeCtl设置为-1，设置成功的，可以进行初始化操作

- f 即为当前 key 定位出的 Node，如果为空表示当前位置可以写入数据，利用 CAS 尝试写入，失败则自旋保证成功。

- 如果当前位置的 hashcode == MOVED == -1,则需要进行扩容。

- 如果都不满足，则利用 synchronized 锁写入数据。
    - 如果找到的桶存在，那么要么是链表结构要么是红黑树结构，此时需要获取该桶的锁，在锁定的情况下执行链表或者红黑树的插入或更新
        - 如果桶中第一个元素的hash值大于0，说明是链表结构，则对链表插入或者更新
        - 如果桶中的第一个元素类型是TreeBin，说明是红黑树结构，则按照红黑树的方式进行插入或者更新

- 如果数量大于 TREEIFY_THRESHOLD 则要转换为红黑树。

2 . get方法

```java
    public V get(Object key) {
        Node<K,V>[] tab; Node<K,V> e, p; int n, eh; K ek;
        int h = spread(key.hashCode());
        if ((tab = table) != null && (n = tab.length) > 0 &&
            (e = tabAt(tab, (n - 1) & h)) != null) {
            if ((eh = e.hash) == h) {
                if ((ek = e.key) == key || (ek != null && key.equals(ek)))
                    return e.val;
            }
            else if (eh < 0)
                return (p = e.find(h, key)) != null ? p.val : null;
            while ((e = e.next) != null) {
                if (e.hash == h &&
                    ((ek = e.key) == key || (ek != null && key.equals(ek))))
                    return e.val;
            }
        }
        return null;
    }

```
- 首先进行哈希值的扰动，获取一个新的哈希值，找到对应的数组e

- 如果该e位置无元素则直接返回null

- 如果该e位置有元素
    - 如果第一个元素的hash值小于0，则该节点可能为ForwardingNode或者红黑树节点TreeBin
      如果是ForwardingNode（表示当前正在进行扩容），使用新的数组来进行查找
      如果是红黑树节点TreeBin，使用红黑树的查找方式来进行查找

- 如果第一个元素的hash大于等于0，则为链表结构，依次遍历即可找到对应的元素

3 . 扩容方法
```java
    private final void treeifyBin(Node<K,V>[] tab, int index) {
        Node<K,V> b; int n, sc;
        if (tab != null) {
            if ((n = tab.length) < MIN_TREEIFY_CAPACITY)
                tryPresize(n << 1);
            else if ((b = tabAt(tab, index)) != null && b.hash >= 0) {
                synchronized (b) {
                    if (tabAt(tab, index) == b) {
                        TreeNode<K,V> hd = null, tl = null;
                        for (Node<K,V> e = b; e != null; e = e.next) {
                            TreeNode<K,V> p =
                                new TreeNode<K,V>(e.hash, e.key, e.val,
                                                  null, null);
                            if ((p.prev = tl) == null)
                                hd = p;
                            else
                                tl.next = p;
                            tl = p;
                        }
                        setTabAt(tab, index, new TreeBin<K,V>(hd));
                    }
                }
            }
        }
    }
```
重点来看看这个扩容过程，即看下上述tryPresize方法，也可以看到上述是2倍扩容的方式
```java
    private final void tryPresize(int size) {
        int c = (size >= (MAXIMUM_CAPACITY >>> 1)) ? MAXIMUM_CAPACITY :
            tableSizeFor(size + (size >>> 1) + 1);
        int sc;
        while ((sc = sizeCtl) >= 0) {
            Node<K,V>[] tab = table; int n;
            if (tab == null || (n = tab.length) == 0) {
                n = (sc > c) ? sc : c;
                if (U.compareAndSwapInt(this, SIZECTL, sc, -1)) {
                    try {
                        if (table == tab) {
                            @SuppressWarnings("unchecked")
                            Node<K,V>[] nt = (Node<K,V>[])new Node<?,?>[n];
                            table = nt;
                            sc = n - (n >>> 2);
                        }
                    } finally {
                        sizeCtl = sc;
                    }
                }
            }
            else if (c <= sc || n >= MAXIMUM_CAPACITY)
                break;
            else if (tab == table) {
                int rs = resizeStamp(n);
                if (sc < 0) {
                    Node<K,V>[] nt;
                    if ((sc >>> RESIZE_STAMP_SHIFT) != rs || sc == rs + 1 ||
                        sc == rs + MAX_RESIZERS || (nt = nextTable) == null ||
                        transferIndex <= 0)
                        break;
                    if (U.compareAndSwapInt(this, SIZECTL, sc, sc + 1))
                        transfer(tab, nt);
                }
                else if (U.compareAndSwapInt(this, SIZECTL, sc,
                                             (rs << RESIZE_STAMP_SHIFT) + 2))
                    transfer(tab, null);
            }
        }
    }
```

第一个执行的线程会首先设置sizeCtl属性为一个负值，然后执行transfer(tab, null)，其他晚进来的线程会检查当前扩容是否已经完成，没完成则帮助进行扩容，完成了则直接退出。

该ConcurrentHashMap的扩容操作可以允许多个线程并发执行，那么就要处理好任务的分配工作。每个线程获取一部分桶的迁移任务，如果当前线程的任务完成，查看是否还有未迁移的桶，

若有则继续领取任务执行，若没有则退出。在退出时需要检查是否还有其他线程在参与迁移工作，如果有则自己什么也不做直接退出，如果没有了则执行最终的收尾工作。

- 问题1：当前线程如何感知其他线程也在参与迁移工作？

靠sizeCtl的值，它初始值是一个负值=(rs << RESIZE_STAMP_SHIFT) + 2)，每当一个线程参与进来执行迁移工作，则该值进行CAS自增，该线程的任务执行完毕要退出时对该值进行CAS自减操作，
所以当sizeCtl的值等于上述初值则说明了此时未有其他线程还在执行迁移工作，可以去执行收尾工作了。见如下代码
```java
    if (i < 0 || i >= n || i + n >= nextn) {
        int sc;
        // 收尾工作，将新的数组替换老的数组，代表迁移工作全部完成
        if (finishing) {
            nextTable = null;
            table = nextTab;
            sizeCtl = (n << 1) - (n >>> 1);
            return;
        }
        // 每个线程完成自己的任务后对sizeCtl 进行自减操作，同时判断是否全部完成任务
        if (U.compareAndSwapInt(this, SIZECTL, sc = sizeCtl, sc - 1)) {
            if ((sc - 2) != resizeStamp(n) << RESIZE_STAMP_SHIFT)
                return;
            finishing = advance = true;
            i = n; // recheck before commit
        }
    }
```

- 问题2：任务按照何规则进行分片？

```java
        if ((stride = (NCPU > 1) ? (n >>> 3) / NCPU : n) < MIN_TRANSFER_STRIDE)
            stride = MIN_TRANSFER_STRIDE; // subdivide range
```
上述stride即是每个分片的大小，目前有最低要求16，即每个分片至少需要16个桶。stride的计算依赖于CPU的核数，如果只有1个核，那么此时就不用分片，即stride=n。其他情况就是 (n >>> 3) / NCPU。

- 问题3：如何记录目前已经分出去的任务？

ConcurrentHashMap含有一个属性transferIndex（初值为最后一个桶），表示从transferIndex开始到后面所有的桶的迁移任务已经被分配出去了。
所以每次线程领取扩容任务，则需要对该属性进行CAS的减操作，即一般是transferIndex-stride。

- 问题4：每个线程如何处理分到的部分桶的迁移工作

第一个获取到分片的线程会创建一个新的数组，容量是之前的2倍。

遍历自己所分到的桶：
- 桶中元素不存在，则通过CAS操作设置桶中第一个元素为ForwardingNode，其Hash值为MOVED（-1）,同时该元素含有新的数组引用

此时若其他线程进行put操作，发现第一个元素的hash值为-1则代表正在进行扩容操作（并且表明该桶已经完成扩容操作了，可以直接在新的数组中重新进行hash和插入操作），
该线程就可以去参与进去，或者没有任务则不用参与，此时可以去直接操作新的数组了

- 桶中元素存在且hash值为-1，则说明该桶已经被处理了（本不会出现多个线程任务重叠的情况，这里主要是该线程在执行完所有的任务后会再次进行检查，再次核对）

- 桶中为链表或者红黑树结构，则需要获取桶锁，防止其他线程对该桶进行put操作，然后处理方式同HashMap的处理方式一样，对桶中元素分为2类，分别代表当前桶中和要迁移到新桶中的元素。
设置完毕后代表桶迁移工作已经完成，旧数组中该桶可以设置成ForwardingNode了

```java
    private final void transfer(Node<K,V>[] tab, Node<K,V>[] nextTab) {
        int n = tab.length, stride;
        // 对任务进行分片
        if ((stride = (NCPU > 1) ? (n >>> 3) / NCPU : n) < MIN_TRANSFER_STRIDE)
            stride = MIN_TRANSFER_STRIDE; // subdivide range
        // 虽然该段代码未被保护，但是是在外围进行保护的，只有第一个线程调用该方法时候 nextTab 为null,其他线程调用时都不为null
        if (nextTab == null) {            // initiating
            try {
                @SuppressWarnings("unchecked")
                Node<K,V>[] nt = (Node<K,V>[])new Node<?,?>[n << 1];
                nextTab = nt;
            } catch (Throwable ex) {      // try to cope with OOME
                sizeCtl = Integer.MAX_VALUE;
                return;
            }
            nextTable = nextTab;
            transferIndex = n;
        }
        int nextn = nextTab.length;
        ForwardingNode<K,V> fwd = new ForwardingNode<K,V>(nextTab);
        boolean advance = true;
        boolean finishing = false; // to ensure sweep before committing nextTab
        for (int i = 0, bound = 0;;) {
            Node<K,V> f; int fh;
            while (advance) {
                int nextIndex, nextBound;
                if (--i >= bound || finishing)
                    advance = false;
                // 全部任务已经分配出去了
                else if ((nextIndex = transferIndex) <= 0) {
                    i = -1;
                    advance = false;
                }
                // 获取本次线程的分片范围，同时CAS更新 transferIndex属性值
                else if (U.compareAndSwapInt
                         (this, TRANSFERINDEX, nextIndex,
                          nextBound = (nextIndex > stride ?
                                       nextIndex - stride : 0))) {
                    bound = nextBound;
                    i = nextIndex - 1;
                    advance = false;
                }
            }
            if (i < 0 || i >= n || i + n >= nextn) {
                int sc;
                // 收尾工作，只有最后一个线程来执行
                if (finishing) {
                    nextTable = null;
                    table = nextTab;
                    sizeCtl = (n << 1) - (n >>> 1);
                    return;
                }
                // 参与扩容的线程未全部结束时，其他线程完成任务后的处理逻辑
                if (U.compareAndSwapInt(this, SIZECTL, sc = sizeCtl, sc - 1)) {
                    if ((sc - 2) != resizeStamp(n) << RESIZE_STAMP_SHIFT)
                        return;
                    finishing = advance = true;
                    i = n; // recheck before commit
                }
            }
            else if ((f = tabAt(tab, i)) == null)
                advance = casTabAt(tab, i, null, fwd);
            else if ((fh = f.hash) == MOVED)
                advance = true; // already processed
            else {
                synchronized (f) {
                    if (tabAt(tab, i) == f) {
                        Node<K,V> ln, hn;
                        if (fh >= 0) {
                            int runBit = fh & n;
                            Node<K,V> lastRun = f;
                            for (Node<K,V> p = f.next; p != null; p = p.next) {
                                int b = p.hash & n;
                                if (b != runBit) {
                                    runBit = b;
                                    lastRun = p;
                                }
                            }
                            if (runBit == 0) {
                                ln = lastRun;
                                hn = null;
                            }
                            else {
                                hn = lastRun;
                                ln = null;
                            }
                            for (Node<K,V> p = f; p != lastRun; p = p.next) {
                                int ph = p.hash; K pk = p.key; V pv = p.val;
                                if ((ph & n) == 0)
                                    ln = new Node<K,V>(ph, pk, pv, ln);
                                else
                                    hn = new Node<K,V>(ph, pk, pv, hn);
                            }
                            setTabAt(nextTab, i, ln);
                            setTabAt(nextTab, i + n, hn);
                            setTabAt(tab, i, fwd);
                            advance = true;
                        }
                        else if (f instanceof TreeBin) {
                            TreeBin<K,V> t = (TreeBin<K,V>)f;
                            TreeNode<K,V> lo = null, loTail = null;
                            TreeNode<K,V> hi = null, hiTail = null;
                            int lc = 0, hc = 0;
                            for (Node<K,V> e = t.first; e != null; e = e.next) {
                                int h = e.hash;
                                TreeNode<K,V> p = new TreeNode<K,V>
                                    (h, e.key, e.val, null, null);
                                if ((h & n) == 0) {
                                    if ((p.prev = loTail) == null)
                                        lo = p;
                                    else
                                        loTail.next = p;
                                    loTail = p;
                                    ++lc;
                                }
                                else {
                                    if ((p.prev = hiTail) == null)
                                        hi = p;
                                    else
                                        hiTail.next = p;
                                    hiTail = p;
                                    ++hc;
                                }
                            }
                            ln = (lc <= UNTREEIFY_THRESHOLD) ? untreeify(lo) :
                                (hc != 0) ? new TreeBin<K,V>(lo) : t;
                            hn = (hc <= UNTREEIFY_THRESHOLD) ? untreeify(hi) :
                                (lc != 0) ? new TreeBin<K,V>(hi) : t;
                            setTabAt(nextTab, i, ln);
                            setTabAt(nextTab, i + n, hn);
                            setTabAt(tab, i, fwd);
                            advance = true;
                        }
                    }
                }
            }
        }
    }
```

## 问题分析
1. ConcurrentHashMap读为什么不需要锁？

我们通常使用读写锁来保护对一堆数据的读写操作。读时加读锁，写时加写锁。在什么样的情况下可以不需要读锁呢？

如果对数据的读写是一个原子操作，那么此时是可以不需要读锁的。如ConcurrentHashMap对数据的读写，写操作是不需要分2次写的（没有中间状态），读操作也是不需要2次读取的。
假如一个写操作需要分多次写，必然会有中间状态，如果读不加锁，那么可能就会读到中间状态，那就不对了。

假如ConcurrentHashMap提供put(key1,value1,key2,value2)，写入的时候必然会存在中间状态即key1写完成，但是key2还未写，此时如果读不加锁，那么就可能读到key1是新数据而key2是老数据的中间状态。

虽然ConcurrentHashMap的读不需要锁，但是需要保证能读到最新数据，所以必须加volatile。即数组的引用需要加volatile，同时一个Node节点中的val和next属性也必须要加volatile。

2. ConcurrentHashMap是否可以在无锁的情况下进行迁移？

目前1.8的ConcurrentHashMap迁移是在锁定旧桶的前提下进行迁移的，然而并没有去锁定新桶。那么就可能提出如下问题：

- 在某个桶的迁移过程中，别的线程想要对该桶进行put操作怎么办？

一旦某个桶在迁移过程中了，必然要获取该桶的锁，所以其他线程的put操作要被阻塞，一旦迁移完毕，该桶中第一个元素就会被设置成ForwardingNode节点，所以其他线程put时需要重新判断下桶中第一个元素是否被更改了，
如果被改了重新获取重新执行逻辑，如下代码
```java
    synchronized (f) {
        if (tabAt(tab, i) == f) {
            if (fh >= 0) {
                ...
            }
            else if (f instanceof TreeBin) {
                ...
            }
        }
    }
```
- 某个桶已经迁移完成（其他桶还未完成），别的线程想要对该桶进行put操作怎么办？

该线程会首先检查是否还有未分配的迁移任务，如果有则先去执行迁移任务，如果没有即全部任务已经分发出去了，那么此时该线程可以直接对新的桶进行插入操作（映射到的新桶必然已经完成了迁移，所以可以放心执行操作）

从上面看到我们在迁移的时候还是需要对旧桶锁定的，能否在无锁的情况下实现迁移？

可以参考这篇论文[Split-Ordered Lists: Lock-Free Extensible Hash Tables](http://people.csail.mit.edu/shanir/publications/Split-Ordered_Lists.pdf)

一旦扩容就涉及到迁移桶中元素的操作，将一个桶中的元素迁移到另一个桶中的操作不是一个原子操作，所以需要在锁的保护下进行迁移。如果扩容操作是移动桶的指向，那么就可以通过一个CAS操作来完成扩容操作。
上述Split-Ordered Lists就是把所有元素按照一定的顺序进行排列。该list被分成一段一段的，每一段都代表某个桶中的所有元素。每个桶中都有一个指向第一个元素的指针，如下图结构所示：

![Split-Ordered Lists](https://static.oschina.net/uploads/img/201701/03160126_OFsv.png)

每一段其实也是分成2类的，如同前面所说的HashMap在扩容是分成2类的情况是一样的，此时Split-Ordered Lists在扩容时就只需要将新桶的指针指向这2类的分界点即可。

这一块之后再详细说明吧。

3. ConcurrentHashMap曾经的弱一致性
具体详见这篇针对老版本的ConcurrentHashMap的说明文章为什么ConcurrentHashMap是弱一致的

文中已经解释到：对数组的引用是volatile来修饰的，但是数组中的元素并不是。即读取数组的引用总是能读取到最新的值，但是读取数组中某一个元素的时候并不一定能读到最新的值。所以说是弱一致性的。

我觉得这个只需要稍微改动下就可以实现强一致性：

- 对于新加的key，通过写入到链表的末尾即可。因为一个元素的next属性是volatile的，可以保证写入后立马看的到，如下1.8的方式

- 或者对数组中元素的更新采用volatile写的方式，如1.7的形式,但是现在1.7版本的ConcurrentHashMap对于数组中元素的写也是加了volatile的，

1.8的方式就是：直接将新加入的元素写入next属性（含有volatile修饰）中而不是修改桶中的第一个元素。
```java
    synchronized (f) {
        if (tabAt(tab, i) == f) {
            if (fh >= 0) {
                binCount = 1;
                for (Node<K,V> e = f;; ++binCount) {
                    K ek;
                    ...
                    Node<K,V> pred = e;
                    // 直接将新加入的元素写入next属性（含有volatile修饰）中而不是修改桶中的第一个元素。
                    if ((e = e.next) == null) {
                        pred.next = new Node<K,V>(hash, key,
                                                  value, null);
                        break;
                    }
                }
            }
            else if (f instanceof TreeBin) {
                ...
            }
        }
    }
```
所以在1.7和1.8版本的ConcurrentHashMap中不再是弱一致性，写入的数据是可以立马本读到的。

> 1.8 在 1.7 的数据结构上做了大的改动，采用红黑树之后可以保证查询效率（O(logn)），甚至取消了 ReentrantLock 改为了 synchronized，这样可以看出在新版的 JDK 中对 synchronized 优化是很到位的。

> 为什么不用ReentrantLock而用synchronized ?
>
>锁的粒度：首先锁的粒度并没有变粗，甚至变得更细了。每当扩容一次，ConcurrentHashMap的并发度就扩大一倍。
>
>Hash冲突：JDK1.7中，ConcurrentHashMap从过二次hash的方式（Segment -> HashEntry）能够快速的找到查找的元素。在1.8中通过链表加红黑树的形式弥补了put、get时的性能差距。
>
>扩容：JDK1.8中，在ConcurrentHashmap进行扩容时，其他线程可以通过检测数组中的节点决定是否对这条链表（红黑树）进行扩容，减小了扩容的粒度，提高了扩容的效率。
>
>减少内存开销:如果使用ReentrantLock则需要节点继承AQS来获得同步支持，增加内存开销，而1.8中只有头节点需要进行同步。
>
>内部优化:synchronized则是JVM直接支持的，JVM能够在运行时作出相应的优化措施：锁粗化、锁消除、锁自旋等等。
>
>ConcurrentHashMap中变量使用final和volatile修饰有什么用呢？
>
> Final域使得确保初始化安全性（initialization safety）成为可能，初始化安全性让不可变形对象不需要同步就能自由地被访问和共享。
>使用volatile来保证某个变量内存的改变对其他线程即时可见，在配合CAS可以实现不加锁对并发操作的支持。get操作可以无锁是由于Node的元素val和指针next是用volatile修饰的，
>在多线程环境下线程A修改结点的val或者新增节点的时候是对线程B可见的。

> 总结：
>通过源码可以看出 使用 CAS + synchronized 方式时 加锁的对象是每个链条的头结点，也就是 锁定 的是冲突的链表，所以再次提高了并发度，
>并发度等于链表的条数或者说 桶的数量。那为什么sement 不把段的大小设置为一个桶呢，因为在粒度比较小的情况下，如果使用ReentrantLock则需要节点继承AQS来获得同步支持，
>增加内存开销，而1.8中只有头节点需要进行同步，粒度表较小，相对来说内存开销就比较大。所以不把segment的大小设置为一个桶。


> 参考 [jdk1.8的HashMap和ConcurrentHashMap](https://my.oschina.net/pingpangkuangmo/blog/817973)