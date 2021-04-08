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

```java
        private void buildFromSorted(int size, Iterator<?> it,
                                     java.io.ObjectInputStream str,
                                     V defaultVal)
            throws  java.io.IOException, ClassNotFoundException {
            this.size = size;
            root = buildFromSorted(0, 0, size-1, computeRedLevel(size),
                                   it, str, defaultVal);
        }

    @SuppressWarnings("unchecked")
    private final Entry<K,V> buildFromSorted(int level, int lo, int hi,
                                             int redLevel,
                                             Iterator<?> it,
                                             java.io.ObjectInputStream str,
                                             V defaultVal)
        throws  java.io.IOException, ClassNotFoundException {
        /*
         * Strategy: The root is the middlemost element. To get to it, we
         * have to first recursively construct the entire left subtree,
         * so as to grab all of its elements. We can then proceed with right
         * subtree.
         *
         * The lo and hi arguments are the minimum and maximum
         * indices to pull out of the iterator or stream for current subtree.
         * They are not actually indexed, we just proceed sequentially,
         * ensuring that items are extracted in corresponding order.
         */

        if (hi < lo) return null;

        int mid = (lo + hi) >>> 1;

        Entry<K,V> left  = null;
        if (lo < mid)
            left = buildFromSorted(level+1, lo, mid - 1, redLevel,
                                   it, str, defaultVal);

        // extract key and/or value from iterator or stream
        K key;
        V value;
        if (it != null) {
            if (defaultVal==null) {
                Map.Entry<?,?> entry = (Map.Entry<?,?>)it.next();
                key = (K)entry.getKey();
                value = (V)entry.getValue();
            } else {
                key = (K)it.next();
                value = defaultVal;
            }
        } else { // use stream
            key = (K) str.readObject();
            value = (defaultVal != null ? defaultVal : (V) str.readObject());
        }

        Entry<K,V> middle =  new Entry<>(key, value, null);

        // color nodes in non-full bottommost level red
        if (level == redLevel)
            middle.color = RED;

        if (left != null) {
            middle.left = left;
            left.parent = middle;
        }

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