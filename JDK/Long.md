# Long

## Long中的缓存实现

可参考Integer,但范围不可变

缓存初始化源码如下

```java
    private static class LongCache {
        private LongCache(){}

        static final Long cache[] = new Long[-(-128) + 127 + 1];

        static {
            for(int i = 0; i < cache.length; i++)
                cache[i] = new Long(i - 128);
        }
    }
```

- 构建一个length=256的数组，数组中的每一个元素对应一个Long值，数值范围 (-128) - 127

## 源码阅读

### 类图

实现了`java.io.Serializable, Comparable<T>`接口,继承了`Number`抽象类

- java.io.Serializable ：序列化接口没有任何方法和域，仅用于标识序列化的语意。

- Comparable<T> ：这个接口只有一个compareTo(T o)方法对两个实例化对象比较大小。

- Number类是java.lang包下的一个抽象类，提供了将包装类型拆箱成基本类型的方法，所有基本类型（数据类型）的包装类型都继承了该抽象类，并且是final声明不可继承改变；

![image-20210313100500327](https://i.loli.net/2021/03/13/zBMQxXy9KpaPqiA.png)

### 主要属性

- `@Native public static final long MIN_VALUE = 0x8000000000000000L;`
  - MIN_VALUE静态变量表示long能取的最小值，为-2的63次方，被final修饰说明不可变。
- `@Native public static final long MAX_VALUE = 0x7fffffffffffffffL;`
  - MAX_VALUE，表示long最大值为2的63次方减1。
- `@Native public static final int SIZE = 64;` 
  - SIZE用来表示二进制补码形式的long值的比特数，值为64，静态变量且不可变。
- `public static final int BYTES = SIZE / Byte.SIZE;`
  - BYTES用来表示二进制补码形式的long值的字节数，值为SIZE除于Byte.SIZE，结果为8。
- `public static final Class<Long> TYPE = (Class<Long>) Class.getPrimitiveClass("long");`
  - 表示基本类型 long 的 Class 实例

### 构造方法

```java
    public Long(long value) {
        this.value = value;
    }
    public Long(String s) throws NumberFormatException {
        this.value = parseLong(s, 10);
    }
```

在`Long`类中提供了两个构造函数，分别针对构造参数为基本类型`long`和引用类型`String`。这两个方法都是给当前的实例的`value`属性赋值，参数为`long`类型的构造器直接将参数赋值给value属性，参数为`String`是将`parseLong(String s, int radix)`方法的返回值赋值。

### 方法

1. valueOf 先上源码：

```java
	public static Long valueOf(long l) {
        final int offset = 128;
        if (l >= -128 && l <= 127) { // will cache
            return LongCache.cache[(int)l + offset];
        }
        return new Long(l);
    }
    public static Long valueOf(String s, int radix) throws NumberFormatException {
        return Long.valueOf(parseLong(s, radix));
    }
```

- 在创建对象之前先从LongCache.cache中（这一个私有的内部类,这个类缓存了`(-128) - 127`之间数字的包装类。）寻找.如果内有找到才使用new新建对象

2. parseLong

```java
    public static long parseLong(String s, int radix)
              throws NumberFormatException
    {
        //检查字符串是否有效
        if (s == null) {
            throw new NumberFormatException("null");
        }

        // 检查进制数是否合理
        if (radix < Character.MIN_RADIX) {
            throw new NumberFormatException("radix " + radix +
                                            " less than Character.MIN_RADIX");
        }
        if (radix > Character.MAX_RADIX) {
            throw new NumberFormatException("radix " + radix +
                                            " greater than Character.MAX_RADIX");
        }

        long result = 0;
        boolean negative = false;
        int i = 0, len = s.length();
        long limit = -Long.MAX_VALUE;
        long multmin;
        int digit;

        if (len > 0) {
            char firstChar = s.charAt(0);
            if (firstChar < '0') { // Possible leading "+" or "-" 第一个字符可能是正负号，即"+"或"-
                if (firstChar == '-') {
                    negative = true;
                    limit = Long.MIN_VALUE;
                } else if (firstChar != '+')
                    throw NumberFormatException.forInputString(s);

                if (len == 1) // Cannot have lone "+" or "-" 只能传正负号
                    throw NumberFormatException.forInputString(s);
                i++;
            }
            multmin = limit / radix;
            while (i < len) {
                // Accumulating negatively avoids surprises near MAX_VALUE 负积累避免 MAX_VALUE附近出错
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

- 只有在构造时传入long类型数据作为参数时，才会触发缓存机制。如果long在-128到127这个范围内，从缓存中取出会减少资源的开销。如果是使用parseLong方法，则没有这种机制。
- 当传入构造方法的是String类型时，valueOf和parseLong是等价的。

3. bitCount //获取二进制补码中1的数量

```java
     public static int bitCount(long i) { //采取平行算法进行计算
        // HD, Figure 5-14
        //1.求出i对应的二进制数的每两位之和，比如(01)(11)(10)(00)-->(01)(10)(01)(00)
        i = i - ((i >>> 1) & 0x5555555555555555L);
        //2.求出(1.)中i对应的二进制数的每四位之和，比如(0110)(0001)(1001)(1001)
        //-->(0010)(0001)(0010)(0010)
        i = (i & 0x3333333333333333L) + ((i >>> 2) & 0x3333333333333333L);
        //3.同上求出(2.)中i的二进制数的每八位之和。
        i = (i + (i >>> 4)) & 0x0f0f0f0f0f0f0f0fL;
        //4.同上求出(3.)中i的二进制数的每十六位之和。
        i = i + (i >>> 8);
        //5.同上求出(4.)中i的二进制数的每三十二位之和。
        i = i + (i >>> 16);
        //6.同上求出(5.)中i的二进制数的每六十四位之和。
        i = i + (i >>> 32);
        //1的个数最多也不会超过64个，所以只取最后7位即可。
        return (int)i & 0x7f; 
     }
```

4. hashCode //返回Long类型对象的Hash码

```java
public static int hashCode(long value) {
    //返回当前Long的值的位移操作结果
    return (int)(value ^ (value >>> 32));
}
```

5.  highestOneBit //返回Long型数二进制位左边第一个1时，剩余位补0的结果值

```java
	public static long highestOneBit(long i) {
        // HD, Figure 3-1
        i |= (i >>  1); //右移一位，并与i进行或操作，使得最高两位全为1
        i |= (i >>  2); //右移两位，并与i进行或操作，使得最高四位全为1
        i |= (i >>  4); //右移四位，并与i进行或操作，使得最高八位全为1
        i |= (i >>  8); //右移八位，并与i进行或操作，使得最高十六位全为1
        i |= (i >> 16); //右移十六位，并与i进行或操作，使得最高三十二位全为1
        i |= (i >> 32); //右移三十二位，并与i进行或操作，使得最高六十四位全为1
        return i - (i >>> 1);
    }
```