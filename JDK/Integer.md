# Integer

## Integer中的缓存实现（享元模式）
> 引用：英文原文：[Java Integer Cache](http://javapapers.com/java/java-integer-cache/) 翻译地址：[Java中整型的缓存机制](http://www.hollischuang.com/?p=1174) 原文作者：[Java Papers](http://javapapers.com/) 翻译作者：[Hollis](http://www.hollischuang.com/) 转载请注明出处。
```java
Integer a = 100;
Integer b = 100;

Integer c = 200;
Integer d = 200;
        
System.out.println(a == b);//true
System.out.println(c == d);//false
```

在Java 5中，在Integer的操作上引入了一个新功能来节省内存和提高性能。整型对象通过使用相同的对象引用实现了缓存和重用。

> 适用于整数值区间-128 至 +127。
>
> 只适用于自动装箱。使用构造函数创建对象不适用。

Java的编译器把基本数据类型自动转换成封装类对象的过程叫做`自动装箱`，相当于使用`valueOf`方法：

```java
Integer a = 10; //这是自动装箱
Integer b = Integer.valueOf(10); 
```

先看源码：

```java
public static Integer valueOf(int i) {
    if (i >= IntegerCache.low && i <= IntegerCache.high)
        return IntegerCache.cache[i + (-IntegerCache.low)];
    return new Integer(i);
}
```

- 在创建对象之前先从IntegerCache.cache中(这一个私有的内部类,这个类缓存了`low - high`之间数字的包装类。)寻找。如果没找到才使用new新建对象。

IntegerCache
先上源码：

```java
    /**
     * Cache to support the object identity semantics of autoboxing for values between
     * -128 and 127 (inclusive) as required by JLS.
     *
     * The cache is initialized on first usage.  The size of the cache
     * may be controlled by the {@code -XX:AutoBoxCacheMax=<size>} option.
     * During VM initialization, java.lang.Integer.IntegerCache.high property
     * may be set and saved in the private system properties in the
     * sun.misc.VM class.
     */

    private static class IntegerCache {
        static final int low = -128;
        static final int high;
        static final Integer cache[];

        static {
            // high value may be configured by property
            int h = 127;
            String integerCacheHighPropValue =
                sun.misc.VM.getSavedProperty("java.lang.Integer.IntegerCache.high");
            if (integerCacheHighPropValue != null) {
                try {
                    int i = parseInt(integerCacheHighPropValue);
                    i = Math.max(i, 127);
                    // Maximum array size is Integer.MAX_VALUE
                    h = Math.min(i, Integer.MAX_VALUE - (-low) -1);
                } catch( NumberFormatException nfe) {
                    // If the property cannot be parsed into an int, ignore it.
                }
            }
            high = h;

            cache = new Integer[(high - low) + 1];
            int j = low;
            for(int k = 0; k < cache.length; k++)
                cache[k] = new Integer(j++);

            // range [-128, 127] must be interned (JLS7 5.1.7)
            assert IntegerCache.high >= 127;
        }

        private IntegerCache() {}
    }
```

- 以上可以知道这个类是私有的且是静态的，并且他有三个被final修饰的静态filed外加一个静态块和一个私有的构造器；很简单很普通的一个类.
- 被缓存的包装类就介于`low - high`之间，low的值已经写死-128，而high的值由你的虚拟机决定`sun.misc.VM.getSavedProperty("java.lang.Integer.IntegerCache.high")`，既然是一个参数也就意味着你可以动态设置，通过`-XX:AutoBoxCacheMax=size`修改。 
- 缓存通过一个for循环实现。从低到高并创建将`low - high`之间数字的装箱后方法`cache[]`这个Integer类型的数组中。这个缓存会在Integer类第一次被使用的时候被初始化出来。以后，就可以使用缓存中包含的实例对象，而不是创建一个新的实例(在自动装箱的情况下)。

## Java语言规范中的缓存行为

在[Boxing Conversion](http://docs.oracle.com/javase/specs/jls/se8/html/jls-5.html#jls-5.1.7)部分的Java语言规范(JLS)规定如下：

> 如果一个变量p的值是：
>
> -128至127之间的整数(§3.10.1)
>
> true 和 false的布尔值 (§3.10.3)
>
> ‘\u0000’至 ‘\u007f’之间的字符(§3.10.4)
>
> 中时，将p包装成a和b两个对象时，可以直接使用a==b判断a和b的值是否相等。

## 其他缓存的对象

这种缓存行为不仅适用于Integer对象。我们针对所有的整数类型的类都有类似的缓存机制。

> 有ByteCache用于缓存Byte对象
>
> 有ShortCache用于缓存Short对象
>
> 有LongCache用于缓存Long对象
>
> 有CharacterCache用于缓存Character对象

`Byte`, `Short`, `Long`有固定范围: -128 到 127。对于`Character`, 范围是 0 到 127。除了`Integer`以外，这个范围都不能改变。

##  源码阅读

### 类图
实现了`java.io.Serializable, Comparable<T>`接口,继承了`Number`抽象类

- java.io.Serializable ：序列化接口没有任何方法和域，仅用于标识序列化的语意。

- Comparable<T> ：这个接口只有一个compareTo(T o)方法对两个实例化对象比较大小。

- Number类是java.lang包下的一个抽象类，提供了将包装类型拆箱成基本类型的方法，所有基本类型（数据类型）的包装类型都继承了该抽象类，并且是final声明不可继承改变；

<div align=center>
<img src="https://i.loli.net/2021/03/01/ENPlRIfFeD1rgpC.png" alt="Integer继承关系图" style="zoom:100%;" />
</div>

### 构造函数

```java
public Integer(int value) {
    this.value = value;
}
public Integer(String s) throws NumberFormatException {
    this.value = parseInt(s, 10);
}
```

在`Integer`类中提供了两个构造函数，分别针对构造参数为基本类型`int`和引用类型`String`。这两个方法都是给当前的实例的`value`属性赋值，参数为`int`类型的构造器直接将参数赋值给value属性，参数为`String`是将`parseInt(String s, int radix)`方法的返回值赋值。

### 方法

toString(int i): 照例先上源码：

```java
public static String toString(int i) {
    if (i == Integer.MIN_VALUE)
        return "-2147483648";
    int size = (i < 0) ? stringSize(-i) + 1 : stringSize(i);
    char[] buf = new char[size];
    getChars(i, size, buf);
    return new String(buf, true);
}
```

这个方法内部调用了两个方法：`stringSize`和`getChars`，toString方法的实现也就是由这两个方法合作完成，`stringSize`实现上很是比较简单的，源码我就不上了，它是用来计算参数i的位数也就是转成字符串之后的字符串的长度。内部结合一个已经初始化好的int类型的数组`sizeTable`来完成这个计算。稍动头脑就能想明白，很巧妙的一个方法。然后来说说`toString`这个方法实现的最大功臣`getChars`这个方法，上源码：

```java
 static void getChars(int i, int index, char[] buf) {
    int q, r;
    int charPos = index;
    char sign = 0;

    if (i < 0) {
        sign = '-';
        i = -i;
    }

    // Generate two digits per iteration
    while (i >= 65536) {
        q = i / 100;
    // really: r = i - (q * 100);
        r = i - ((q << 6) + (q << 5) + (q << 2));
        i = q;
        buf [--charPos] = DigitOnes[r];
        buf [--charPos] = DigitTens[r];
    }

    // Fall thru to fast mode for smaller numbers
    // assert(i <= 65536, i);
    for (;;) {
        q = (i * 52429) >>> (16+3);
        r = i - ((q << 3) + (q << 1));  // r = i-(q*10) ...
        buf [--charPos] = digits [r];
        i = q;
        if (i == 0) break;
    }
    if (sign != 0) {
        buf [--charPos] = sign;
    }
}
```

三个参数：`i`:被初始化的数字，`index`:这个数字的长度(包含了负数的符号“-”)，`buf`:字符串的容器-一个char型数组。第一个`if`判断，如果i<0,sign记下它的符号“-”，同时将i转成整数。下面所有的操作也就只针对整数了，最后在判断`sign`如果不等于零将sig你的值放在char数组的首位`buf [--charPos] = sign;`。 最后来分析方法中的两个循环：`while`和`for`，其实这两个循环做的事情一样。只是`while`循环来处理i>65535的情况，且每次取两位数：

```
buf [--charPos] = DigitOnes[r];
buf [--charPos] = DigitTens[r];
```

剩下的情况由`for`循环处理，且每次去一个数字。至于为什么这么做：`// Fall thru to fast mode for smaller numbers`，这是官方注释，意思就是真对小的数字使用快速方式。针对这块的理解我也是参考了知乎上的网友的回答[java源码中Integer.class中有个getChars方法，里面有个52429是怎么确定的？ ](https://www.zhihu.com/question/34948884/answer/60497785)表示感谢。

还有就是我们比较常用的parseInt

```java
    public static int parseInt(String s, int radix)
                throws NumberFormatException
    {
        /*
         * WARNING: This method may be invoked early during VM initialization
         * before IntegerCache is initialized. Care must be taken to not use
         * the valueOf method.
         */

        if (s == null) {
            throw new NumberFormatException("null");
        }

        if (radix < Character.MIN_RADIX) {
            throw new NumberFormatException("radix " + radix +
                                            " less than Character.MIN_RADIX");
        }

        if (radix > Character.MAX_RADIX) {
            throw new NumberFormatException("radix " + radix +
                                            " greater than Character.MAX_RADIX");
        }

        int result = 0;
        boolean negative = false;
        int i = 0, len = s.length();
        int limit = -Integer.MAX_VALUE;
        int multmin;
        int digit;

        if (len > 0) {
            char firstChar = s.charAt(0);
            if (firstChar < '0') { // Possible leading "+" or "-"
                if (firstChar == '-') {
                    negative = true;
                    limit = Integer.MIN_VALUE;
                } else if (firstChar != '+')
                    throw NumberFormatException.forInputString(s);

                if (len == 1) // Cannot have lone "+" or "-"
                    throw NumberFormatException.forInputString(s);
                i++;
            }
            multmin = limit / radix;
            while (i < len) {
                // Accumulating negatively avoids surprises near MAX_VALUE
                digit = Character.digit(s.charAt(i++),radix);
                if (digit < 0) {
                    throw NumberFormatException.forInputString(s);
                }
                if (result < multmin) {
                    throw NumberFormatException.forInputString(s);
                }
                result *= radix;
                if (result < limit + digit) {
                    throw NumberFormatException.forInputString(s);
                }
                result -= digit;
            }
        } else {
            throw NumberFormatException.forInputString(s);
        }
        return negative ? result : -result;
    }
```
这个方法首先对传入的字符串进行判空和对传入的进制数进行判断（2<=radix<=36）,开头可能会有 '+' 和 '-'，表示整数的正负，不能只有一个"+"或者"-"，对此进行处理。
以指定的进制转换整数字符，然后对字符串中包含非数字字符进行判断，最后就是溢出的判断。溢出的判断可能让人难以理解，让我们来仔细看一下：
```java
    if (result < multmin) {
        throw NumberFormatException.forInputString(s);
    }
    result *= radix;
    if (result < limit + digit) {
        throw NumberFormatException.forInputString(s);
    }
    result -= digit;
```
溢出判断思想是通过除法的方式，这个在[v_JULY_v](https://blog.csdn.net/v_july_v/article/details/9024123)的文章中有提到（比较好的解决方法是用除法和取模）， 
可以发现 JDK 将所有的整数先当做负数来处理，那他为什么这么处理？

首先int 的上下限是-2147483648 ~ 2147483647，可见负数表达的范围比正数多一个，这样就好理解为什么在开头要把 limit 全部表达为负数（下限）。
这样的操作减少了后续的判断，相当于二者选择取其大一样，大的包含了小的，不用正负数情况都判断一次。同理，那么 multmin 也都是负数了，
而且可以认为是只和进制参数 radix 有关系。在这里 multmin 就是-MAX_VALUE/radix，当负数时就是MIN_VALUE/radix。所以第一个条件就是若-result < -MAX_VALUE/radix则是溢出。
不满足第一个条件的情况下，result*radix肯定不会溢出了。所以第二个条件判断若 - result*radix < -INT_MAX + digit，则是溢出。

比如对于最大的负数来说，当扫描到最后一位时，result = -214748364，multmin=-214748364
第一个条件result < multmin不满足， 执行result *= radix之后，result = -2147483640
第二个条件result < limit + digit，即 -2147483640<-2147483648+10 也不满足条件。
所以正常输出。