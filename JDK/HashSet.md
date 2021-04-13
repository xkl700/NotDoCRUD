# HashSet/LinkedHashSet/TreeSet
Set集合的底层都是使用相对应的Map。HashSet底层是HashMap,LinkedHashSet底层是使用LinkedHashMap，TreeSet底层是使用TreeMap。都相对简单，所以就放在一篇文章里讲解了。只简单讲一下HashSet的源码，因为其他两者大同小异。
HashSet
## 数据结构和基础字段
```java
    private transient HashMap<E,Object> map;

    private static final Object PRESENT = new Object();
```
显而易见，hashSet的底层和方法就是使用hashmap。Set实现值不重复的原理就是在保存值到底层hashmap里的时候，保存为key值，value值为常量PRESENT。具体可以看下面方法讲解。
方法细节
## 构造方法
```java
    public HashSet() {
        map = new HashMap<>();
    }
    public HashSet(Collection<? extends E> c) {
        map = new HashMap<>(Math.max((int) (c.size()/.75f) + 1, 16));
        addAll(c);
    }
    
    //自己初始化容量和加载因子的大小
    public HashSet(int initialCapacity, float loadFactor) {
        map = new HashMap<>(initialCapacity, loadFactor);
    }
    
    //初始化容量大小，加载因子用默认的
    public HashSet(int initialCapacity) {
        map = new HashMap<>(initialCapacity);
    }
```
简单的构造方法，就不阐述了。
## 其他方法
add方法

//这个方法就可以得知HashSet添加的元素是不能够重复的，原因是什么呢，set将每次添加的元素度是通过map中的key来保存，当有
//相同的key时，也就是添加了相同的元素，那么map会讲value给覆盖掉，而key还是原来的key，所以，这就是set不能够重复的原因。这个方法的PRESENT可以看下面的注释，
//返回值的式子的意思很好理解，map中增加元素是先通过key查找有没有相同的key值，如果有，则覆盖value，返回oldValue。没有
//，则创建一个新的entry，并且返回null值。如果key等于null，也会返回null值。所以return会有一个==null的判断
```java
    public boolean add(E e) {
        return map.put(e, PRESENT)==null;
    }
```
remove方法
```java
//set通过判断删除元素是否为常量PRESENT，来确定是否成功删除
    public boolean remove(Object o) {
        return map.remove(o)==PRESENT;
    }

    //hashmap中的remove方法
    public V remove(Object key) {
        Entry<K,V> e = removeEntryForKey(key);
        return (e == null ? null : e.value);
    }
```

## 总结

元素没有顺序(现在知道为什么没有顺序了把，因为底层用的是HashMap，HashMap本身中的元素度没有顺序，那么HashSet更不可能有了)
元素不能重复，集合元素可以是null,但只能放入一个null(这个也很好理解了，因为HashSet中存放的元素，度是当作HashMap的key来使用，HashMap中key是不能相同的，所以HashSet的元素也不能重复了)  
非线程安全。

HashSet使用成员对象来计算hashcode值，对于两个对象来说hashcode可能相同，所以equals()方法用来判断对象的相等性，如果两个对象不同的话，那么返回false。所以如果存储自定义对象，需要重写自定义对象的hashcode方法和equals方法。
TreeSet可以确保集合元素处于排序状态。TreeSet支持两种排序方式，自然排序 和定制排序，其中自然排序为默认的排序方式。
LinkedHashSet集合同样是根据元素的hashCode值来决定元素的存储位置，但是它同时使用链表维护元素的次序,当遍历该集合时候，LinkedHashSet将会以元素的添加顺序访问集合的元素。LinkedHashSet创建的时候只会创建以插入顺序排序的LinkedHashMap。

> [java基础：HashSet/LinkedHashSet/TreeSet — 源码分析](https://juejin.cn/post/6844903752277688327)

