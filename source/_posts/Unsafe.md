---
title: Unsafe
date: 2019-02-20 10:01:42
categories:
- Java
- Unsafe

tags:
- Java
- Unsafe
---
# Unsafe--Java 指针（操作内存）
java 8: rt.jar\sun.misc
java 9: jdk.unsupported\sun.misc    java.base\jdk.internal.misc


<!-- more -->
---
## Java 8
```java
package sun.misc;
public final class Unsafe {
    private static final Unsafe theUnsafe;
    ....
    public final native boolean compareAndSwapObject(Object var1, long var2, Object var4, Object var5);
    public final native boolean compareAndSwapInt(Object var1, long var2, int var4, int var5);
    public final native boolean compareAndSwapLong(Object var1, long var2, long var4, long var6);
    ....
    public static Unsafe getUnsafe() {
        Class var0 = Reflection.getCallerClass();
        if (!VM.isSystemDomainLoader(var0.getClassLoader())) {
            throw new SecurityException("Unsafe");
        } else {
            return theUnsafe;
        }
    }
}

VM.class
public static boolean isSystemDomainLoader(ClassLoader var0) {
   return var0 == null;
}
```
## 获取Unsafe实例
```java
private static Unsafe getUnsafe() {
    Unsafe unsafe = null;
    try {
        Field theUnsafe = Unsafe.class.getDeclaredField("theUnsafe");
        theUnsafe.setAccessible(true);
        unsafe = (Unsafe) theUnsafe.get(Unsafe.class);//or (Unsafe) theUnsafe.get(null);
    } catch (Exception e) {
        e.printStackTrace();
    } finally {
        return unsafe;
    }
}
```
## 修改内存
>深入理解内存使用

```java
//Bits.java
//判断内存中数据存储模式：大端存储，小端存储
static {
    long a = unsafe.allocateMemory(8);//申请8字节内存
    try {
        unsafe.putLong(a, 0x0102030405060708L);//内存中写入数据
        byte b = unsafe.getByte(a);
        switch (b) {
        case 0x01: byteOrder = ByteOrder.BIG_ENDIAN;     break;//高字节低地址
        case 0x08: byteOrder = ByteOrder.LITTLE_ENDIAN;  break;//高字节高地址
        default:
            assert false;
            byteOrder = null;
        }
    } finally {
        unsafe.freeMemory(a);//释放内存
    }
}
```

## 类对象相关
putXXX(Object,offset,value);
Object: 指向修改位置的起始地址
offset: 偏移
value:  待写入值

### 实例变量修改&类变量修改

```java
class People {
    private static int num;
    static {
        System.out.println("init class");
    }
    private int age;
    private People() {
        System.out.println("init People.instance");
    }
    public static int getNum() {
        return num;
    }
    public int getAge() {
        return age;
    }
}



Field theUnsafe = Unsafe.class.getDeclaredField("theUnsafe");
theUnsafe.setAccessible(true);
Unsafe unsafe = (Unsafe) theUnsafe.get(Unsafe.class);//or (Unsafe) theUnsafe.get(null);
//theUnsafe.get(Unsafe.class.newInstance())
// exception: Class loader.UnsafeTest can not access a member of class sun.misc.Unsafe with modifiers "private"
// private Unsafe() {
//    }

People user =(People) unsafe.allocateInstance(People.class);//实例化但未经过new Constructor();


//类变量修改
Field staticNumField = People.class.getDeclaredField("num");
staticNumField.setAccessible(true);
long staticNumFieldOffset=unsafe.staticFieldOffset(staticNumField);
unsafe.putInt(People.class,staticNumFieldOffset,11);
System.out.println(staticNumField.get(user));


//实例变量修改
Field field = People.class.getDeclaredField("age");
field.setAccessible(true);
long ageOffset = unsafe.objectFieldOffset(field);
unsafe.putInt(user, ageOffset, 22);
System.out.println(field.get(user));


//...................
unsafe.compareAndSwapInt(user,staticNumFieldOffset,11,111);//类变量存储位置....
System.out.println(staticNumField.get(user));
unsafe.compareAndSwapInt(People.class,staticNumFieldOffset,11,111);
System.out.println(staticNumField.get(user));
```
结果
```
init class
11
22
11
111
```



## 理解
```java
Unsafe unsafe = getUnsafe();
final String s = "abc";
String s1 = "abc";

System.out.println(s==s1);//true
//获取s的实例变量value
Field valueInString = String.class.getDeclaredField("value");
//获取value的变量偏移值
long offset = unsafe.objectFieldOffset(valueInString);
//value本身是一个char[],要修改它元素的值，仍要获取baseOffset和indexScale
long base = unsafe.arrayBaseOffset(char[].class);
long scale = unsafe.arrayIndexScale(char[].class);

//获取value
char[] values = (char[]) unsafe.getObject(s, offset);

//为value赋值
unsafe.putChar(values, base + scale * 2, 'd');

System.out.println("s:" + s + " s1:" + s1);

//将s的值改为 abc
s1="abc";
String s2 = "abc";
String s3 = new String("abc");
System.out.println(s==s1);
System.out.println(s.equals(s1));
System.out.println("s:" + s + " s1:" + s1 + "  s2:" + s2 + "  s3:" + s3);
```
结果：
```java
true
s:abc s1:abd
true
true
s:abc s1:abd  s2:abd  s3:abd
```


---
## Java 9
```java
jdk.internal.misc.Unsafe;

private static final Unsafe theUnsafe = new Unsafe();
public static Unsafe getUnsafe() {
   return theUnsafe;
}
```

......
