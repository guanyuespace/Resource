---
title: 【Java 并发笔记】Unsafe 相关整理
date: 2019-04-15 16:03:02
---
<!-- TOC -->

- [1. Unsafe 类](#1-unsafe-%E7%B1%BB)
  - [1.1 获取实例](#11-%E8%8E%B7%E5%8F%96%E5%AE%9E%E4%BE%8B)
  - [1.2 常用方法](#12-%E5%B8%B8%E7%94%A8%E6%96%B9%E6%B3%95)
    - [1.2.1 Class 相关](#121-class-%E7%9B%B8%E5%85%B3)
    - [1.2.2 Object 相关](#122-object-%E7%9B%B8%E5%85%B3)
    - [1.2.3 数组相关](#123-%E6%95%B0%E7%BB%84%E7%9B%B8%E5%85%B3)
    - [1.2.4 CAS 相关](#124-cas-%E7%9B%B8%E5%85%B3)
    - [1.2.5 线程调度相关](#125-%E7%BA%BF%E7%A8%8B%E8%B0%83%E5%BA%A6%E7%9B%B8%E5%85%B3)
    - [1.2.6 volatile 相关读写](#126-volatile-%E7%9B%B8%E5%85%B3%E8%AF%BB%E5%86%99)
    - [1.2.7 内存屏障相关](#127-%E5%86%85%E5%AD%98%E5%B1%8F%E9%9A%9C%E7%9B%B8%E5%85%B3)
    - [1.2.8 内存管理（***非堆内存***）](#128-%E5%86%85%E5%AD%98%E7%AE%A1%E7%90%86%E9%9D%9E%E5%A0%86%E5%86%85%E5%AD%98)
    - [1.2.9 系统相关](#129-%E7%B3%BB%E7%BB%9F%E7%9B%B8%E5%85%B3)
    - [1.2.10 其他](#1210-%E5%85%B6%E4%BB%96)
  - [1.3 Unsafe 类的使用场景](#13-unsafe-%E7%B1%BB%E7%9A%84%E4%BD%BF%E7%94%A8%E5%9C%BA%E6%99%AF)
    - [1.3.1 避免初始化](#131-%E9%81%BF%E5%85%8D%E5%88%9D%E5%A7%8B%E5%8C%96)
    - [1.3.2 内存修改](#132-%E5%86%85%E5%AD%98%E4%BF%AE%E6%94%B9)
    - [1.3.3 动态类](#133-%E5%8A%A8%E6%80%81%E7%B1%BB)
    - [1.3.4 大数组](#134-%E5%A4%A7%E6%95%B0%E7%BB%84)
    - [1.3.5 并发应用](#135-%E5%B9%B6%E5%8F%91%E5%BA%94%E7%94%A8)
    - [1.3.6 ***挂起与恢复***](#136-%E6%8C%82%E8%B5%B7%E4%B8%8E%E6%81%A2%E5%A4%8D)
      - [1.3.6.1 park 和 unpark 灵活之处](#1361-park-%E5%92%8C-unpark-%E7%81%B5%E6%B4%BB%E4%B9%8B%E5%A4%84)

<!-- /TOC -->
【Java 并发笔记】Unsafe 相关整理
==
>[转自：【Java 并发笔记】Unsafe 相关整理](https://www.jianshu.com/p/2e5b92d0962e)

# 1. Unsafe 类

*   Java 不能直接访问操作系统底层，而是通过本地方法来访问。Unsafe 类提供了硬件级别的原子操作。
*   Unsafe 类在 **sun.misc** 包下，不属于 Java 标准。很多 Java 的基础类库，包括一些被广泛使用的高性能开发库都是基于 Unsafe 类开发，比如 Netty、Hadoop、Kafka 等。
    *   Unsafe 是用于在实质上扩展 Java 语言表达能力、便于在更高层（Java 层）代码里实现原本要在更低层（C 层）实现的核心库功能用的。
    *   这些功能包括裸内存的申请/释放/访问，低层硬件的 atomic/volatile 支持，创建未初始化对象等。
    *   它原本的设计就只应该被标准库使用，因此不建议在生产环境中使用。

## 1.1 获取实例


*   Unsafe 对象不能直接通过 **new Unsafe()** 或调用 **Unsafe.getUnsafe()** 获取。
    *   Unsafe 被设计成单例模式，构造方法私有。
    *   getUnsafe 被设计成只能从引导类加载器（bootstrap class loader）加载。
```java
private Unsafe() {
}
public static Unsafe getUnsafe() {
        Class var0 = Reflection.getCallerClass(2);
        if (var0.getClassLoader() != null) {
            throw new SecurityException("Unsafe");
        } else {
            return theUnsafe;
        }
}
```

*   非启动类加载器直接调用 Unsafe.getUnsafe() 方法会抛出 SecurityException 异常。
*   解决办法有两个:      
1）可以令代码 " 受信任 "。运行程序时，通过 JVM 参数设置 bootclasspath 选项，指定系统类路径加上使用的一个 Unsafe 路径。
```java
java -Xbootclasspath:/usr/jdk1.7.0/jre/lib/rt.jar:. com.mishadoff.magic.UnsafeClient
```
2）通过 Java 反射机制。
 * 通过将 private 单例实例暴力设置 accessible 为 true，然后通过 Field 的 get 方法，直接获取一个 Object 强制转换为 Unsafe。
```java
Field theUnsafe = Unsafe.class.getDeclaredField("theUnsafe");
theUnsafe.setAccessible(true);
unsafe = (Unsafe) theUnsafe.get(Unsafe.class);//or (Unsafe) theUnsafe.get(null);
```

* 在 IDE 中，这些方法会被标志为 Error，可以通过以下设置解决：
```
Preferences -> Java -> Compiler -> Errors/Warnings -> Deprecated and restricted API -> Forbidden reference -> Warning
```

## 1.2 常用方法


*   Unsafe 的大部分 API 都是 native 的方法。

### 1.2.1 Class 相关

*   主要提供 Class 和它的静态字段的操作方法。
```java
//静态属性的偏移量，用于在对应的 Class 对象中读写静态属性
public native long staticFieldOffset(Field f);
public native Object staticFieldBase(Field f);
//判断是否需要初始化一个类
public native boolean shouldBeInitialized(Class c);
//确保类被初始化
public native void ensureClassInitialized(Class c);
//定义一个类，可用于动态创建类
// Tell the VM to define a class, without security checks.
public native Class defineClass(String name, byte[] b, int off, int len, ClassLoader loader, ProtectionDomain protectionDomain);
//动态创建类
public native Class defineClass(String var1, byte[] var2, int var3, int var4);
//定义一个匿名类，可用于动态创建类
public native Class defineAnonymousClass(Class hostClass, byte[] data, Object[] cpPatches);
```


### 1.2.2 Object 相关

*   Java 中的基本类型（boolean、byte、char、short、int、long、float、double）及对象引用类型都有以下方法。
```java
//获得对象的字段偏移量
public native long objectFieldOffset(Field f);
//获得给定对象地址偏移量的int值
public native int getInt(Object o, long offset);
//设置给定对象地址偏移量的int值
public native void putInt(Object o, long offset, int x);
//获得给定对象地址偏移量的值
public native Object getObject(Object o, long offset);
//设置给定对象地址偏移量的值
public native void putObject(Object o, long offset, Object x);
//创建对象，但并不会调用其构造方法。如果类未被初始化，将初始化类。
public native Object allocateInstance(Class cls) throws InstantiationException;
```

### 1.2.3 数组相关

*   通过 arrayBaseOffset 和 arrayIndexScale 可定位数组中每个元素在内存中的位置。   
```java
//返回数组中第一个元素的偏移地址
public native int arrayBaseOffset(Class arrayClass);
//boolean、byte、short、char、int、long、float、double，及对象类型均有以下方法
/** The value of {@code arrayBaseOffset(boolean[].class)} */
public static final int ARRAY_BOOLEAN_BASE_OFFSET = theUnsafe.arrayBaseOffset(boolean[].class);
/**
 * Report the scale factor for addressing elements in the storage
 * allocation of a given array class. However, arrays of "narrow" types
 * will generally not work properly with accessors like {@link
 * #getByte(Object, int)}, so the scale factor for such classes is reported
 * as zero.
 *
 * @see #arrayBaseOffset
 * @see #getInt(Object, long)
 * @see #putInt(Object, long, int)
 */
//返回数组中每一个元素占用的大小
public native int arrayIndexScale(Class arrayClass);
//boolean、byte、short、char、int、long、float、double，及对象类型均有以下方法
/** The value of {@code arrayIndexScale(boolean[].class)} */
public static final int ARRAY_BOOLEAN_INDEX_SCALE = theUnsafe.arrayIndexScale(boolean[].class);
```

### 1.2.4 CAS 相关
*   compareAndSwap，内存偏移地址 offset，预期值 expected，新值 x。如果变量在当前时刻的值和预期值 expected 相等，尝试将变量的值更新为 x。如果更新成功，返回 true；否则，返回 false。
```java
//更新变量值为x，如果当前值为expected
//o：对象 offset：偏移量 expected：期望值 x：新值
public final native boolean compareAndSwapObject(Object o, long offset, Object expected, Object x);
public final native boolean compareAndSwapInt(Object o, long offset, int expected, int x);
public final native boolean compareAndSwapLong(Object o, long offset, long expected, long x);
```

*   JDK 1.8 中基于 CAS 扩展。
    *   作用都是，通过 CAS 设置新的值，返回旧的值。
```
//增加
public final int getAndAddInt(Object o, long offset, int delta) {
 int v;
 do {
 v = getIntVolatile(o, offset);
 } while (!compareAndSwapInt(o, offset, v, v + delta));
 return v;
}
public final long getAndAddLong(Object o, long offset, long delta) {
 long v;
 do {
 v = getLongVolatile(o, offset);
 } while (!compareAndSwapLong(o, offset, v, v + delta));
 return v;
}
//设置
public final int getAndSetInt(Object o, long offset, int newValue) {
 int v;
 do {
 v = getIntVolatile(o, offset);
 } while (!compareAndSwapInt(o, offset, v, newValue));
 return v;
}
public final long getAndSetLong(Object o, long offset, long newValue) {
 long v;
 do {
 v = getLongVolatile(o, offset);
 } while (!compareAndSwapLong(o, offset, v, newValue));
 return v;
}
public final Object getAndSetObject(Object o, long offset, Object newValue) {
 Object v;
 do {
 v = getObjectVolatile(o, offset);
 } while (!compareAndSwapObject(o, offset, v, newValue));
 return v;
```

### 1.2.5 线程调度相关

*   主要包括监视器锁定、解锁等。
```java
//取消阻塞线程
public native void unpark(Object thread);
//阻塞线程
public native void park(boolean isAbsolute, long time);
//获得对象锁
public native void monitorEnter(Object o);
//释放对象锁
public native void monitorExit(Object o);
//尝试获取对象锁，返回 true 或 false 表示是否获取成功
public native boolean tryMonitorEnter(Object o);
```

### 1.2.6 volatile 相关读写
```java
//从对象的指定偏移量处获取变量的引用，使用 volatile 的加载语义
//相当于 getObject(Object, long) 的 volatile 版本
//从主存中获取值
public native Object getObjectVolatile(Object o, long offset);

//存储变量的引用到对象的指定的偏移量处，使用 volatile 的存储语义
//相当于 putObject(Object, long, Object) 的 volatile 版本
//设置值刷新主存
public native void putObjectVolatile(Object o, long offset, Object x);
/**
 * Version of {@link #putObjectVolatile(Object, long, Object)}
 * that does not guarantee immediate visibility of the store to
 * other threads. This method is generally only useful if the
 * underlying field is a Java volatile (or if an array cell, one
 * that is otherwise only accessed using volatile accesses).
 */
public native void putOrderedObject(Object o, long offset, Object x);

/** Ordered/Lazy version of {@link #putIntVolatile(Object, long, int)} */
public native void putOrderedInt(Object o, long offset, int x);

/** Ordered/Lazy version of {@link #putLongVolatile(Object, long, long)} */
public native void putOrderedLong(Object o, long offset, long x);
```

### 1.2.7 内存屏障相关

*   JDK 1.8 引入 ，用于定义内存屏障，避免代码重排序。
```java
//内存屏障，禁止 load 操作重排序，即屏障前的load操作不能被重排序到屏障后，屏障后的 load 操作不能被重排序到屏障前
public native void loadFence();
//内存屏障，禁止 store 操作重排序，即屏障前的 store 操作不能被重排序到屏障后，屏障后的 store 操作不能被重排序到屏障前
public native void storeFence();
//内存屏障，禁止 load、store 操作重排序
public native void fullFence();
```

### 1.2.8 内存管理（***非堆内存***）

-   allocateMemory **所分配的内存需要手动 free（不被 GC 回收）**
```java
//（boolean、byte、char、short、int、long、float、double) 都有以下 get、put 两个方法。
//获得给定地址上的 int 值
public native int getInt(long address);
//设置给定地址上的 int 值
public native void putInt(long address, int x);
//获得本地指针
public native long getAddress(long address);
//存储本地指针到给定的内存地址
public native void putAddress(long address, long x);
//分配内存
public native long allocateMemory(long bytes);
//重新分配内存
public native long reallocateMemory(long address, long bytes);
//初始化内存内容
public native void setMemory(Object o, long offset, long bytes, byte value);
//初始化内存内容
public void setMemory(long address, long bytes, byte value) {
 setMemory(null, address, bytes, value);
}
//内存内容拷贝
public native void copyMemory(Object srcBase, long srcOffset, Object destBase, long destOffset, long bytes);
//内存内容拷贝
public void copyMemory(long srcAddress, long destAddress, long bytes) {
 copyMemory(null, srcAddress, null, destAddress, bytes);
}
//释放内存
public native void freeMemory(long address);
```

### 1.2.9 系统相关
```java
//返回指针的大小。返回值为 4 或 8。
public native int addressSize();

/** The value of {@code addressSize()} */
public static final int ADDRESS_SIZE = theUnsafe.addressSize();

//内存页的大小。
public native int pageSize();
```

### 1.2.10 其他
```java
//获取系统的平均负载值，loadavg 这个 double 数组将会存放负载值的结果，nelems 决定样本数量，nelems 只能取值为 1 到 3，分别代表最近 1、5、15 分钟内系统的平均负载。
//如果无法获取系统的负载，此方法返回 -1，否则返回获取到的样本数量（loadavg 中有效的元素个数）。
public native int getLoadAverage(double[] loadavg, int nelems);
//绕过检测机制直接抛出异常。
public native void throwException(Throwable ee);
```

## 1.3 Unsafe 类的使用场景


### 1.3.1 避免初始化

*   ***当想要绕过对象构造方法、安全检查器或者没有 public 的构造方法时，allocateInstance() 方法变得非常有用。***
*   编写一个简单的 Java 类。
```java
public class TestA {
    private int a = 0;

    public TestA() {
        a = 1;
    }

    public int getA() {
        return a;
    }
}
```

*   构造方法、反射方法和 allocateInstance 方法的不同实现。
    *   ***将 public 构造方法修改为 private，allocateInstance 方法可以得到同样的结果。***
```java
// constructor
TestA constructorA = new TestA();
System.out.println(constructorA.getA()); //print 1
// reflection
try {
     TestA reflectionA = TestA.class.newInstance();
     System.out.println(reflectionA.getA()); //print 1
} catch (InstantiationException e) {
     e.printStackTrace();
} catch (IllegalAccessException e) {
     e.printStackTrace();
}
// unsafe
Field f = null;
try {
     f = Unsafe.class.getDeclaredField("theUnsafe");
     f.setAccessible(true);
     Unsafe unsafe = (Unsafe) f.get(null);
     TestA unsafeA = (TestA) unsafe.allocateInstance(TestA.class);
     System.out.println(unsafeA.getA()); //print 0
} catch (NoSuchFieldException e) {
     e.printStackTrace();
} catch (IllegalAccessException e) {
     e.printStackTrace();
} catch (InstantiationException e) {
     e.printStackTrace();
}
```

### 1.3.2 内存修改

*   Unsafe 可用于绕过安全的常用技术，直接修改内存变量。
    *   反射也可以实现相同的功能。但是 Unsafe 可以修改任何对象，甚至没有这些对象的引用。
*   编写一个简单的 Java 类。
```java
public class TestA {
    private int ACCESS_ALLOWED = 1;
    public boolean giveAccess() {
        return 40 == ACCESS_ALLOWED;
    }
}
```

*   在正常情况下，giveAccess 总会返回 false。
    *   通过计算内存偏移，并使用 putInt() 方法，类的 ACCESS_ALLOWED 被修改。
    *   在已知类结构的时候，数据的偏移总是可以计算出来（与 c++ 中的类中数据的偏移计算是一致的）。
```java
// constructor
TestA constructorA = new TestA();
System.out.println(constructorA.giveAccess()); //print false
// unsafe
Field f = null;
try {
    f = Unsafe.class.getDeclaredField("theUnsafe");
    f.setAccessible(true);
    Unsafe unsafe = (Unsafe) f.get(null);
    TestA unsafeA = (TestA) unsafe.allocateInstance(TestA.class);
    Field unsafeAField = unsafeA.getClass().getDeclaredField("ACCESS_ALLOWED");
    unsafe.putInt(unsafeA, unsafe.objectFieldOffset(unsafeAField), 40); // memory corruption
    System.out.println(unsafeA.giveAccess()); //print true
} catch (NoSuchFieldException e) {
    e.printStackTrace();
} catch (IllegalAccessException e) {
    e.printStackTrace();
} catch (InstantiationException e) {
    e.printStackTrace();
}
```

### 1.3.3 动态类

*   可以在运行时创建一个类，比如从已编译的 .class 文件中将类内容读取为字节数组，并正确地传递给 defineClass 方法。
    *   当必须动态创建类，而现有代码中有一些代理，这非常有用。
*   编写一个简单的 Java 类。
```java
public class TestA {
    private int a = 1;
    public int getA() {
        return a;
    }
    public void setA(int a) {
        this.a = a;
    }
}
```

*   动态创建类。
```java
byte[] classContents = new byte[0];
try {
      classContents = getClassContent();
      Class c = getUnsafe().defineClass("com.me.TestA", classContents, 0, classContents.length);
      System.out.println(c.getMethod("getA").invoke(c.newInstance(), null)); //print 1
} catch (Exception e) {
      e.printStackTrace();
}
private static Unsafe getUnsafe() {
        Field f = null;
        Unsafe unsafe = null;
        try {
            f = Unsafe.class.getDeclaredField("theUnsafe");
            f.setAccessible(true);
            unsafe = (Unsafe) f.get(null);
        } catch (NoSuchFieldException e) {
            e.printStackTrace();
        } catch (IllegalAccessException e) {
            e.printStackTrace();
        }
        return unsafe;
}
private static byte[] getClassContent() throws Exception {
        File f = new File("/home/test/TestA.class");
        FileInputStream input = new FileInputStream(f);
        byte[] content = new byte[(int) f.length()];
        input.read(content);
        input.close();
        return content;
}
```

### 1.3.4 大数组
*   Java 数组大小的最大值为 Integer.MAX_VALUE。使用直接内存分配，创建的数组大小受限于堆大小。
*   ***Unsafe 分配的内存，分配在非堆内存，因为不执行任何边界检查，所以任何非法访问都可能会导致 JVM 崩溃。***
    *   在需要分配大的连续区域、实时编程（不能容忍 JVM 延迟）时，可以使用它。java.nio 使用这一技术。
*   创建一个 Java 类。
```java
public class SuperArray {
    private final static int BYTE = 1;
    private long size;
    private long address;
    public SuperArray(long size) {
        this.size = size;
        address = getUnsafe().allocateMemory(size * BYTE);
    }
    public void set(long i, byte value) {
        getUnsafe().putByte(address + i * BYTE, value);
    }
    public int get(long idx) {
        return getUnsafe().getByte(address + idx * BYTE);
    }
    public long size() {
        return size;
    }
    private static Unsafe getUnsafe() {
        Field f = null;
        Unsafe unsafe = null;
        try {
            f = Unsafe.class.getDeclaredField("theUnsafe");
            f.setAccessible(true);
            unsafe = (Unsafe) f.get(null);
        } catch (NoSuchFieldException e) {
            e.printStackTrace();
        } catch (IllegalAccessException e) {
            e.printStackTrace();
        }
        return unsafe;
    }
}
```

*   使用大数组。
```java
long SUPER_SIZE = (long) Integer.MAX_VALUE * 2;
SuperArray array = new SuperArray(SUPER_SIZE);
System.out.println("Array size:" + array.size()); //print 4294967294
int sum = 0;
for (int i = 0; i < 100; i++) {
     array.set((long) Integer.MAX_VALUE + i, (byte) 3);
     sum += array.get((long) Integer.MAX_VALUE + i);
}
System.out.println("Sum of 100 elements:" + sum);  //print 300
```

### 1.3.5 并发应用

*   **compareAndSwap** 方法是原子的，并且可用来实现高性能的、无锁的数据结构。
*   创建一个 Java 类。
```java
public class CASCounter {
    private volatile long counter = 0;
    private Unsafe unsafe;
    private long offset;
    public CASCounter() throws Exception {
        unsafe = getUnsafe();
        offset = unsafe.objectFieldOffset(CASCounter.class.getDeclaredField("counter"));
    }
    public void increment() {
        long before = counter;
        while (!unsafe.compareAndSwapLong(this, offset, before, before + 1)) {
            before = counter;
        }
    }
    public long getCounter() {
        return counter;
    }
    private static Unsafe getUnsafe() {
        Field f = null;
        Unsafe unsafe = null;
        try {
            f = Unsafe.class.getDeclaredField("theUnsafe");
            f.setAccessible(true);
            unsafe = (Unsafe) f.get(null);
        } catch (NoSuchFieldException e) {
            e.printStackTrace();
        } catch (IllegalAccessException e) {
            e.printStackTrace();
        }
        return unsafe;
    }
}
```

*   使用无锁的数据结构。
```java
public static void main(String[] args) {
        final TestB b = new TestB();
        Thread threadA = new Thread(new Runnable() {
            @Override public void run() {
                b.counter.increment();
            }
        });
        Thread threadB = new Thread(new Runnable() {
            @Override public void run() {
                b.counter.increment();
            }
        });
        Thread threadC = new Thread(new Runnable() {
            @Override public void run() {
                b.counter.increment();
            }
        });
        threadA.start();
        threadB.start();
        threadC.start();
        try {
            TimeUnit.SECONDS.sleep(1);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
        System.out.println(b.counter.getCounter()); //print 3
}
private static class TestB {
        private CASCounter counter;

        public TestB() {
            try {
                counter = new CASCounter();
            } catch (Exception e) {
                e.printStackTrace();
            }
        }
}
```

### 1.3.6 ***挂起与恢复***
```java
public native void unpark(Thread jthread);  
public native void park(boolean isAbsolute, long time); // isAbsolute 参数是指明时间是绝对的，还是相对的。
```

*   将一个线程进行挂起是通过 park 方法实现，调用 park 后，线程将一直阻塞直到超时或者中断等条件出现。
*   unpark 可以终止一个挂起的线程，使其恢复正常。
    *   整个并发框架中对线程的挂起操作被封装在 LockSupport 类中，LockSupport 类中有各种版本 pack 方法，但最终都调用的 Unsafe.park() 方法。
*   unpark 函数为线程提供 " 许可（permit）"，线程调用 park 函数则等待 " 许可 "。
    *   这个有点像信号量，但是这个 " 许可 " 不能叠加，是一次性的。
        *   比如线程 B 连续调用了三次 unpark 函数，当线程 A 调用 park 函数就使用掉这个 " 许可 "，如果线程 A 再次调用 park，则进入等待状态。
```java
Thread currThread = Thread.currentThread();
getUnsafe().unpark(currThread);
getUnsafe().unpark(currThread);
getUnsafe().unpark(currThread);

getUnsafe().park(false, 0);
getUnsafe().park(false, 0);
System.out.println("execute success"); // 线程挂起，不会打印。
```

*   unpark 函数可以先于 park 调用（但最好别这样做），比如线程 B 调用 unpark 函数，给线程 A 发了一个 " 许可 "，那么当线程 A 调用 park 时，发现已经有 " 许可 "，会马上再继续运行。
*   park 遇到线程终止时，会直接返回（不同于 Thread.sleep，Thread.sleep 遇到 thread.interrupt() 会抛异常）。
*   unpark 无法恢复处于 sleep 中的线程，只能与 park 配对使用，因为 unpark 发放的许可只有 park 能监听到。

#### 1.3.6.1 park 和 unpark 灵活之处

*   因为 park 的特性，可以不用担心 park 的时序问题。
*   park / unpark 模型真正解耦了线程之间的同步，线程之间不再需要一个 Object 或者其它变量来存储状态，不再需要关心对方的状态。

参考资料
==

[https://www.jb51.net/article/140726.htm](https://www.jb51.net/article/140726.htm)  
[https://blog.csdn.net/luoyoub/article/details/79918104](https://blog.csdn.net/luoyoub/article/details/79918104)  
[https://www.cnblogs.com/suxuan/p/4948608.html](https://www.cnblogs.com/suxuan/p/4948608.html)  
[https://www.cnblogs.com/throwable/p/9139947.html](https://www.cnblogs.com/throwable/p/9139947.html)
