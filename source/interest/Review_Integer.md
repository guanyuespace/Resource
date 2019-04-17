# Review

<!-- TOC -->

- [Review](#review)
- [Integer](#integer)
  - [valueOf](#valueof-function%20valueOf()%20%7B%20%5Bnative%20code%5D%20%7D1)
  - [toStr](#tostr)
  - [parseInt](#parseint)
  - [位操作&最低位](#%E4%BD%8D%E6%93%8D%E4%BD%9C%E6%9C%80%E4%BD%8E%E4%BD%8D)
  - [位操作&高位零的数目](#%E4%BD%8D%E6%93%8D%E4%BD%9C%E9%AB%98%E4%BD%8D%E9%9B%B6%E7%9A%84%E6%95%B0%E7%9B%AE)
  - [位操作&低位零的数目](#%E4%BD%8D%E6%93%8D%E4%BD%9C%E4%BD%8E%E4%BD%8D%E9%9B%B6%E7%9A%84%E6%95%B0%E7%9B%AE)
  - [位操作&循环移位](#%E4%BD%8D%E6%93%8D%E4%BD%9C%E5%BE%AA%E7%8E%AF%E7%A7%BB%E4%BD%8D)
  - [位操作&符号位](#%E4%BD%8D%E6%93%8D%E4%BD%9C%E7%AC%A6%E5%8F%B7%E4%BD%8D)
  - [位操作&二进制反序](#%E4%BD%8D%E6%93%8D%E4%BD%9C%E4%BA%8C%E8%BF%9B%E5%88%B6%E5%8F%8D%E5%BA%8F)
  - [位操作&二进制中1的数目](#%E4%BD%8D%E6%93%8D%E4%BD%9C%E4%BA%8C%E8%BF%9B%E5%88%B6%E4%B8%AD1%E7%9A%84%E6%95%B0%E7%9B%AE)
- [AtomicInteger](#atomicinteger)
  - [简介](#%E7%AE%80%E4%BB%8B)
  - [Unsafe](#unsafe)
  - [补充-volatile](#%E8%A1%A5%E5%85%85-volatile)
  - [测试](#%E6%B5%8B%E8%AF%95)

<!-- /TOC -->
---
# Integer  

## valueOf
```java
public static Integer valueOf(int i) {
    if (i >= IntegerCache.low && i <= IntegerCache.high)
        return IntegerCache.cache[i + (-IntegerCache.low)];
    return new Integer(i);
}
Integer#IntegerCache(-128~127)
```

## toStr
```java
toStr(val, radix){
    while (val >= radix) {
       buf[charPos--] = digits[(val % radix)];
       val = val / radix;
    }
    buf//result array buffer
}  
```

## parseInt
```java
parseInt(s, radix){
    digit = Character.digit(s.charAt(i++),radix);
    result*=radix;
    result+=digit;
}

## 位操作&最高位
```java
public static int highestOneBit(int i) {
   // HD, Figure 3-1
   i |= (i >>  1);//右移一位，  
   i |= (i >>  2);
   i |= (i >>  4);
   i |= (i >>  8);
   i |= (i >> 16);
   return i - (i >>> 1);//
}
```
**分析：**  
  - 右移一位，最高位1右移一位，取或--&gt; 最高的两位都是1
  - 右移两位，最高两位都是1，再右移两位取或---&gt; 最高四位都是1
  - int共32位，到最后一步使得最高位之后均为1，返回最高位为1的数      

**Result:**   
```
jshell> Integer.toString(Integer.highestOneBit(15),16)
$1 ==> "8"
jshell> Integer.toString(Integer.highestOneBit(16),16)
$2 ==> "10"
jshell> Integer.toString(Integer.highestOneBit(7),16)
$3 ==> "4"
jshell> Integer.toString(Integer.highestOneBit(0x7fffffff),16)
$5 ==> "40000000"
```

## 位操作&最低位
```java
public static int lowestOneBit(int i) {
    // HD, Section 2-1
    return i & -i;
}
```
**分析：**    
  - 补码表示： 原码取反+1
  - 原码取反,  +1-->（主体：反码）最低位0变1 (0之后的1均变为0）   因此使得最低位0之后的数位与原码都相同（主体：补码）   
  - 同样（主体：原码）最低位1之后的数位与补码均相同   

省去符号位(高位)表示
```
^1([0|1]+)?|0   

1101 0000     0010 1111+1    0001 0000
```
## 位操作&高位零的数目
```java
public static int numberOfLeadingZeros(int i) {
    // HD, Figure 5-6
    if (i == 0)
        return 32;
    int n = 1;
    if (i >>> 16 == 0) { n += 16; i <<= 16; }//无符号右移16  高16位==0（i左移16位保存低16位，舍弃高16位）   
    if (i >>> 24 == 0) { n +=  8; i <<=  8; }//无符号右移24  高8位==0
    if (i >>> 28 == 0) { n +=  4; i <<=  4; }//
    if (i >>> 30 == 0) { n +=  2; i <<=  2; }
    n -= i >>> 31;
    return n;
}
```    
**Result:**   
```jshell> Integer.toString(Integer.numberOfLeadingZeros(0x1),10)
$8 ==> "31"
jshell> Integer.toString(Integer.numberOfLeadingZeros(0xffffffff),10)
$10 ==> "0"
jshell> Integer.toString(Integer.numberOfLeadingZeros(0x3fffffff),10)
$11 ==> "2"
jshell> Integer.toString(Integer.numberOfLeadingZeros(0x1fffffff),10)
$12 ==> "3"
```
## 位操作&低位零的数目
```java
public static int numberOfTrailingZeros(int i) {
    // HD, Figure 5-14
    int y;
    if (i == 0) return 32;
    int n = 31;
    y = i <<16; if (y != 0) { n = n -16; i = y; }//低十六位!=0  -=16
    y = i << 8; if (y != 0) { n = n - 8; i = y; }
    y = i << 4; if (y != 0) { n = n - 4; i = y; }
    y = i << 2; if (y != 0) { n = n - 2; i = y; }
    return n - ((i << 1) >>> 31);
}
```
**Result:**   
```
jshell> Integer.toString(Integer.numberOfTrailingZeros(0x77),10)
$16 ==> "0"
jshell> Integer.toString(Integer.numberOfTrailingZeros(0x75),10)
$17 ==> "0"
jshell> Integer.toString(Integer.numberOfTrailingZeros(0x76),10)
$18 ==> "1"
```
## 位操作&循环移位
```java
public static int rotateLeft(int i, int distance) {//循环左移
    return (i << distance) | (i >>> -distance);//左移|移出的补位
}
public static int rotateRight(int i, int distance) {//循环右移
   return (i >>> distance) | (i << -distance);//无符号右移|移出的补位
}
```

## 位操作&符号位
```java
public static int signum(int i) {
   // HD, Section 2-7
   return (i >> 31) | (-i >>> 31);//i<0?-1:0|  -i>0?0:1    -1|0=-1(负数)   0|1=1（正数） 0
}
```
... ...

## 位操作&二进制反序
```java
public static int reverse(int i) {
   // HD, Figure 7-1
   i = (i & 0x55555555) << 1 | (i >>> 1) & 0x55555555;
   i = (i & 0x33333333) << 2 | (i >>> 2) & 0x33333333;
   i = (i & 0x0f0f0f0f) << 4 | (i >>> 4) & 0x0f0f0f0f;
   i = (i << 24) | ((i & 0xff00) << 8) |
       ((i >>> 8) & 0xff00) | (i >>> 24);
   return i;
}
```

## 位操作&二进制中1的数目
```java
public static int bitCount(int i) {
    // HD, Figure 5-2
    i = i - ((i >>> 1) & 0x55555555);
    i = (i & 0x33333333) + ((i >>> 2) & 0x33333333);
    i = (i + (i >>> 4)) & 0x0f0f0f0f;
    i = i + (i >>> 8);
    i = i + (i >>> 16);
    return i & 0x3f;
}
```
<font color="red" size="+3">数学计算</font><br/>     
函数功能说明：技术整型i二进制表示中1的个数。

解释过程，设：$i=b_0\cdot2^0+b_1\cdot2^1+...+b_{30}\cdot2^{30}+b_{31}\cdot2^{31}$。      
要求的结果为：$b_0+b_1+b_2+b_3+...+b_{30}+b_{31}$ 的值。    

第一步`i = i - ((i >>> 1) & 0x55555555);`(5的二进制为0101)得到的结果为：     
$i=b_0\cdot2^0+b_1\cdot2^1+...+b_{31}\cdot2^{31}-(b_1\cdot2^0+b_3\cdot2^2+b_5\cdot2^4+b_{31}\cdot2^{30}) \\ =(b_0-b_1)\cdot2^0+b_1\cdot2^1+(b_2-b_3)\cdot2^2+...+b_{29}\cdot2^{29}+(b_{30}-b_{31})\cdot2^{30}+b_{31}\cdot2^{31}\\ =(b_0+b_1)\cdot2^0+(b_2+b_3)\cdot2^2+...+(b_{28}+b_{29})\cdot2^{28}+(b_{30}+b_{31})\cdot2^{30}$。

第二步`i = (i & 0x33333333) + ((i >>> 2) & 0x33333333);`(3的二进制为0011)得到的结果为：   
$i=((b_0+b_1)\cdot2^0+(b_4+b_5)\cdot2^4+...+(b_{28}+b_{29})\cdot2^{28})\\ +((b_2+b_3)\cdot2^0+(b_6+b_7)\cdot2^4+...+(b_{30}+b_{31})\cdot2^{28}\\ =(b_0+b_1+b_2+b_3)\cdot2^0+(b_4+b_5+b_6+b_7)\cdot2^4+...+(b_{28}+b_{29}+b_{30}+b_{31})\cdot2^{28}$  

第三步`i = (i + (i >>> 4)) & 0x0f0f0f0f;`(0f的二进制为00001111)中i + (i >>> 4)得到的结果为：
$i + (i >>> 4)=((b_0+b_1+b_2+b_3)\cdot2^0+(b_4+b_5+b_6+b_7)\cdot2^4+...\\ +(b_{24}+b_{25}+b_{26}+b_{27})\cdot2^{24}+(b_{28}+b_{29}+b_{30}+b_{31})\cdot2^{28})\\ +((b_4+b_5+b_6+b_7)\cdot2^0+(b_8+b_9+b_{10}+b_{11})\cdot2^4+...+(b_{28}+b_{29}+b_{30}+b_{31})\cdot2^{24})\\ =(b_0+b_1+b_2+...+b_6+b_7)\cdot2^0+(b_4+b_5+b_6+...+b_{10}+b_{11})\cdot2^4+...\\ +(b_{24}+b_{25}+b_{26}+...+b_{30}+b_{31})\cdot2^{24}+(b_{28}+b_{29}+b_{30}+b_{31})\cdot2^{28}$

所以`i = (i + (i >>> 4)) & 0x0f0f0f0f;`得到的结果为：
$i=(b_0+b_1+b_2+...+b_6+b_7)\cdot2^0+(b_8+b_9+b_{10}+b_{11}+...+b_{14}+b_{15})\cdot2^8+...+(b_{24}+b_{25}+b_{26}+...+b_{30}+b_{31})\cdot2^{24}$

第四步`i = i + (i >>> 8);`得到的结果为：   
$i=(b_0+b_1+b_2+...+b_{14}+b_{15})\cdot2^0+(b_8+b_9+b_{10}+...+b_{22}+b_{23})\cdot2^8\\ +(b_{16}+b_{17}+b_{18}+...+b_{30}+b_{31})\cdot2^{16}+(b_{24}+b_{25}+b_{26}+...+b_{30}+b_{31})\cdot2^{24}$

第五步`i = i + (i >>> 16);`得到的结果为：   
$i=(b_0+b_1+b_2+...+b_{30}+b_{31})\cdot2^0+(b_8+b_9+b_{10}+...+b_{30}+b_{31})\cdot2^8\\ +(b_{16}+b_{17}+b_{18}+...+b_{30}+b_{31})\cdot2^{16}+(b_{24}+b_{25}+b_{26}+...+b_{30}+b_{31})\cdot2^{24}$

最后`i & 0x3f;`得到的结果为：   
$i=\sum_{i=0}^{31}b_i$   

---
# AtomicInteger   
>An int value that may be updated atomically.
>
>An AtomicInteger is used in applications such as atomically incremented counters, and **cannot be used as a replacement for an java.lang.Integer.**   
However, this class does extend Number to allow uniform access by tools and utilities that deal with numerically-based classes.

## 简介
```java
public class AtomicInteger extends Number implements java.io.Serializable {
    // setup to use Unsafe.compareAndSwapInt for updates
    private static final Unsafe unsafe = Unsafe.getUnsafe();

    private static final long valueOffset;//field: value   --> offset  
    private volatile int value;
    static {
       try {
           valueOffset = unsafe.objectFieldOffset(AtomicInteger.class.getDeclaredField("value"));
       } catch (Exception ex) { throw new Error(ex); }
    }

    /////////////////////////////////////Function///////////////////////////////////////////////////////////////
    public final void lazySet(int newValue) {
        unsafe.putOrderedInt(this, valueOffset, newValue);
    }
    public final boolean compareAndSet(int expect, int update) {
        return unsafe.compareAndSwapInt(this, valueOffset, expect, update);
    }
    public final int getAndIncrement() {
        return unsafe.getAndAddInt(this, valueOffset, 1);
    }
    public final int incrementAndGet() {
        return unsafe.getAndAddInt(this, valueOffset, 1) + 1;
    }

    //IntBinaryOperator: FunctionalInterface
    public final int accumulateAndGet(int x,
                                      IntBinaryOperator accumulatorFunction) {
        int prev, next;
        do {
            prev = get();
            next = accumulatorFunction.applyAsInt(prev, x);
        } while (!compareAndSet(prev, next));//ok 原子操作的基础... ...
        return next;
    }
}
```
由代码可知，所有操作均有Unsafe.java类进行操作完成     


## Unsafe
```java
public final class Unsafe {
    private static final Unsafe theUnsafe;//ok

    public static Unsafe getUnsafe() {
        Class caller = Reflection.getCallerClass();
        if (!VM.isSystemDomainLoader(caller.getClassLoader())) {
            throw new SecurityException("Unsafe");
        } else {
            return theUnsafe;
        }
    }

    static{
      registerNatives();
      Reflection.registerMethodsToFilter(Unsafe.class, new String[]{"getUnsafe"});
      theUnsafe = new Unsafe();
      ... ...
    }

    ////////////////////////////About AtomicInteger/////////////////////////////////////////
    public native void putOrderedInt(Object var1, long offset, int newValue);
    public final native boolean compareAndSwapInt(Object var1, long offset, int expect, int update);//Value(var1.offset)==expect?{Value(var1.offset)=update; return true;}:return false;
    public final int getAndAddInt(Object var1, long offset, int addon) {
        int current;
        do {
            current = this.getIntVolatile(var1, offset);//取出当前值
        } while(!this.compareAndSwapInt(var1, offset, current, current + addon));//set the new value successfully ? true : false;
        return current;
    }
}
```
阻塞同步和非阻塞同步都是实现线程安全的两个保障手段，非阻塞同步对于阻塞同步而言主要解决了阻塞同步中线程阻塞和唤醒带来的性能问题，那什么叫做非阻塞同步呢？     
*在并发环境下，某个线程对共享变量先进行操作，如果没有其他线程争用共享数据那操作就成功；如果存在数据的争用冲突，那就采取补偿措施，比如不断的重试机制，直到成功为止，因为这种乐观的并发策略不需要把线程挂起，也就把这种同步操作称为非阻塞同步（操作和冲突检测具备原子性）。* ***在硬件指令集的发展驱动下，使得 "操作和冲突检测" 这种看起来需要多次操作的行为只需要一条处理器指令便可以完成，这些指令中就包括非常著名的 CAS 指令（Compare-And-Swap 比较并交换）。*** 《深入理解 Java 虚拟机第二版. 周志明》第十三章中这样描述关于 CAS 机制:
![《深入理解 Java 虚拟机第二版. 周志明》13.2.2](https://img-blog.csdnimg.cn/20190111101332407.png "《深入理解 Java 虚拟机第二版. 周志明》13.2.2")  

**原子操作：** incrementAndGet() 方法在一个无限循环体内，不断尝试将一个比当前值大 1 的新值赋给自己，如果失败则说明在执行 "获取 - 设置" 操作的时已经被其它线程修改过了，于是便再次进入循环下一次操作，直到成功为止。   

## 补充-volatile
- 保证变量在线程间可见，对 volatile 变量所有的写操作都能立即反应到其他线程中，换句话说，volatile 变量在各个线程中是一致的（得益于 java 内存模型—"先行发生原则"）；
- 禁止指令的重排序优化；

## 测试
```java
public class AtomicIntegerTest {
    private static AtomicInteger counter = new AtomicInteger(0);

    public static void main(String[] args) throws InterruptedException {
        Thread.currentThread().setPriority(Thread.MIN_PRIORITY);
        ExecutorService executors = Executors.newFixedThreadPool(50);
        for (int i = 0; i < 10000; i++) {
            executors.execute(() -> {
                increase();
            });
        }

        Thread.sleep(50);
        if (Thread.activeCount() > 1) {
            Thread.yield();
        }

        System.out.println(counter.get());
    }

    private static void increase() {
        counter.accumulateAndGet(20, (left, right) -> {
            return left + right;
        });
    }
}
```
**结果：**
```
200000
Process finished with exit code 0
```

---
# Thread
> 

## yield
## join
