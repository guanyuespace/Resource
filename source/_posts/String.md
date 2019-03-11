---
title: String  
date: 2019-02-21 11:43:09  
categories:
- Java
- SourceCode
- String
tags:  
- Java
- String
---
# String
```java
public final class String
    implements java.io.Serializable, Comparable<String>, CharSequence {
    /** The value is used for character storage. */
    private final char value[];

    /** Cache the hash code for the string */
    private int hash; // Default to 0

    public int hashCode() {
        int h = hash;
        if (h == 0 && value.length > 0) {
            char val[] = value;

            for (int i = 0; i < value.length; i++) {
                h = 31 * h + val[i];//32=2^5    左移5位减去自身+字符ASCII码值
            }
            hash = h;
        }
        return h;
    }
    public boolean equals(Object anObject) {
        if (this == anObject) {//同一个引用
            return true;
        }
        if (anObject instanceof String) {
            String anotherString = (String)anObject;
            int n = value.length;
            if (n == anotherString.value.length) {//长度相同（减少比较次数）
                char v1[] = value;
                char v2[] = anotherString.value;
                int i = 0;
                while (n-- != 0) {//。。。
                    if (v1[i] != v2[i])
                        return false;
                    i++;
                }
                return true;
            }
        }
        return false;
    }
}
```
<!-- more -->

## String的存储
```
NewUnsafeTest.java
NewUnsafeTest.class
```
String s="abc"
存储在常量区？？？
String str=new String("abc")
存储在堆区

### 源代码
```java
public static void main(String[] args) throws NoSuchFieldException, IllegalAccessException {
    Unsafe unsafe = getUnsafe();
    final String s = "abc";
    String s1 = "abc";

    System.out.println(s==s1);
    //获取 String.Field:value
    Field valueInString = String.class.getDeclaredField("value");
    //获取 Field 偏移值
    long offset = unsafe.objectFieldOffset(valueInString);

    ////基址+偏移
    long base = unsafe.arrayBaseOffset(char[].class);
    long scale = unsafe.arrayIndexScale(char[].class);

    char[] values = (char[]) unsafe.getObject(s, offset);
    unsafe.putChar(values, base + scale * 2, 'd');

    System.out.println("s:" + s + " s1:" + s1);

    //将s的值改为 abc
    s1="abc";
    String s2 = "abc";
    String s3 = new String("abc");              // 源码：String(String original){this.value = original.value;}   --->  "abc"
    System.out.println(s==s1);
    System.out.println(s.equals(s1));
    System.out.println("s:" + s + " s1:" + s1 + "  s2:" + s2 + "  s3:" + s3);
    Field valueField=String.class.getDeclaredField("value");
    valueField.setAccessible(true);

    ///一个字符串在常量曲中一处内存
    System.out.println("c");
    System.out.println("c"==(s1.substring(2)));
    System.out.println(((char[])valueField.get(s1))[2]+"\t"+((char[])valueField.get("c"))[0]);
    System.out.println(((char[])valueField.get(s1))[2]==((char[])valueField.get("abcd"))[2]);

    ///"abc".equals(s1)  ---> 字符比较过程
    System.out.println(((char[])valueField.get(s1))[2]+"\t"+((char[])valueField.get("abc"))[2]);
    System.out.println(((char[])valueField.get(s1))[2]==((char[])valueField.get("abc"))[2]);
    System.out.println("d".equals(s1.substring(2)));


    ///由于字符串value值一般无法更改，内存中 常量池含有字符串"abc" 直接输出---->  "abd"  常量区数据改变
    System.out.println("abc");



    ////Arrays.copyOf(value, value.length);
    char[] ori=new char[]{'a','b','c'};
    String s4=new String(ori);
    System.out.println(s4);

    values= (char[]) unsafe.getObject(s4,offset);
    unsafe.putChar(values,base+scale*2,'d');
    System.out.println(s4);
    System.out.println(Arrays.toString(ori));
    unsafe.putChar(ori,base+scale*2,'d');
    System.out.println(Arrays.toString(ori));



    String s5 = "";
    values = (char[]) unsafe.getObject(s5, offset);
    System.out.println("values: "+Arrays.toString(values));

    // where is this in NewUnsfae.class ????????????????????????????????????????????
    values = ori;//----------------------------------------------------why-----  ???
    System.out.println(values.length);
    values = ori;//----------------------------------------------------why-----  ???
    System.out.println(values.length);
    values = ori;//----------------------------------------------------why-----  ???
    values = ori;//----------------------------------------------------why-----  ???
    values = ori;//----------------------------------------------------why-----  ???
    values = ori;//----------------------------------------------------why-----  ???
    values = ori;//----------------------------------------------------why-----  ???
    values = ori;//----------------------------------------------------why-----  ???
    values = ori;//----------------------------------------------------why-----  ???
    values = ori;//----------------------------------------------------why-----  ???
    values = ori;//----------------------------------------------------why-----  ???
    values = ori;//----------------------------------------------------why-----  ???
    values = ori;//----------------------------------------------------why-----  ???
    values = ori;//----------------------------------------------------why-----  ???
    System.out.println(values.length);
   //        System.arraycopy(ori, 0, values, 0, 3);
    System.out.println("s5: " + s5);
    System.out.println("values: " + Arrays.toString(values));
    System.out.println("ori: " + Arrays.toString(ori));
    unsafe.putChar(values, base + scale * 2, 'x');
    System.out.println("s5: " + s5);
    System.out.println("values: " + Arrays.toString(values));
    System.out.println("ori: " + Arrays.toString(ori));
    unsafe.putChar(ori, base + scale * 2, 'y');
    System.out.println("s5: " + s5);
    System.out.println("values: " + Arrays.toString(values));
    System.out.println("ori: " + Arrays.toString(ori));
}

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
Result：
```
true                                        s==s1         //编译期替换  "abc" == s1                                
s:abc s1:abd
true                                        s==s1
true                                        s.equals(s1)
s:abc s1:abd  s2:abd  s3:abd
c                                           "c"
false                                       "c".equals(s1.substring(2)))  
d	c                                         ((char[])valueField.get(s1))[2]+"\t"+((char[])valueField.get("c"))[0]
false                                       ((char[])valueField.get(s1))[2]  ==   ((char[])valueField.get("c"))[0]
d	d                                         ((char[])valueField.get(s1))[2]+"\t"+((char[])valueField.get("abc"))[2]
true                                        ((char[])valueField.get(s1))[2]  == ((char[])valueField.get("abc"))[2]
true                                        "d".equals(s1.substring(2))
abd                                         new String("abc")
abc                                         new String( new char[]{'a','b','c'} )
abd                                         s4: unsafe 修改内存
[a, b, c]                                   ori: 内存不变
[a, b, d]                                   ori: unsafe修改内存

Process finished with exit code 0
```

## s=abc而s1=abd
参见字节码文件`NewUnsafeTest.class`
```java
public static void main(String[] args) throws NoSuchFieldException, IllegalAccessException, InstantiationException {
   Unsafe unsafe = getUnsafe();
   String s = "abc";
   String s1 = "abc";
   System.out.println("abc" == s1);
   Field valueInString = String.class.getDeclaredField("value");
   long offset = unsafe.objectFieldOffset(valueInString);
   long base = (long)unsafe.arrayBaseOffset(char[].class);
   long scale = (long)unsafe.arrayIndexScale(char[].class);
   char[] values = (char[])((char[])unsafe.getObject("abc", offset));
   unsafe.putChar(values, base + scale * 2L, 'd');
   System.out.println("s:abc s1:" + s1);
   s1 = "abc";
   String s2 = "abc";
   String s3 = new String("abc");
   System.out.println("abc" == s1);
   System.out.println("abc".equals(s1));
   System.out.println("s:abc s1:" + s1 + "  s2:" + s2 + "  s3:" + s3);
   Field valueField = String.class.getDeclaredField("value");
   valueField.setAccessible(true);
   System.out.println("c");
   System.out.println("c" == s1.substring(2));
   System.out.println(((char[])((char[])valueField.get(s1)))[2] + "\t" + ((char[])((char[])valueField.get("c")))[0]);
   System.out.println(((char[])((char[])valueField.get(s1)))[2] == ((char[])((char[])valueField.get("abcd")))[2]);
   System.out.println(((char[])((char[])valueField.get(s1)))[2] + "\t" + ((char[])((char[])valueField.get("abc")))[2]);
   System.out.println(((char[])((char[])valueField.get(s1)))[2] == ((char[])((char[])valueField.get("abc")))[2]);
   System.out.println("d".equals(s1.substring(2)));
   System.out.println("abc");
   char[] ori = new char[]{'a', 'b', 'c'};
   String s4 = new String(ori);
   System.out.println(s4);
   values = (char[])((char[])unsafe.getObject(s4, offset));
   unsafe.putChar(values, base + scale * 2L, 'd');
   System.out.println(s4);
   System.out.println(Arrays.toString(ori));
   unsafe.putChar(ori, base + scale * 2L, 'd');
   System.out.println("ori: " + Arrays.toString(ori));
   String s5 = "";
   values = (char[])((char[])unsafe.getObject(s5, offset));
   System.out.println("values: " + Arrays.toString(values));
   System.out.println(ori.length);
   System.out.println(ori.length);
   System.out.println(ori.length);
   System.out.println("s5: " + s5);
   System.out.println("values: " + Arrays.toString(ori));
   System.out.println("ori: " + Arrays.toString(ori));
   unsafe.putChar(ori, base + scale * 2L, 'x');
   System.out.println("s5: " + s5);
   System.out.println("values: " + Arrays.toString(ori));
   System.out.println("ori: " + Arrays.toString(ori));
   unsafe.putChar(ori, base + scale * 2L, 'y');
   System.out.println("s5: " + s5);
   System.out.println("values: " + Arrays.toString(ori));
   System.out.println("ori: " + Arrays.toString(ori));
}

private static Unsafe getUnsafe() {
   Unsafe unsafe = null;

   try {
       Field theUnsafe = Unsafe.class.getDeclaredField("theUnsafe");
       theUnsafe.setAccessible(true);
       unsafe = (Unsafe)theUnsafe.get(Unsafe.class);
       return unsafe;
   } catch (Exception var5) {
       var5.printStackTrace();
       return unsafe;
   } finally {
       ;
   }
}
```

## "abc"==s1 & "abc".equals(s1)  --> true,true
Java.lang.String: `Strings are constant; their values cannot be changed after they are created. String buffers support mutable strings. Because String objects are immutable they can be shared.`  

**Because String objects are immutable they can be shared.**
"abc" is shared......

```java
String s1="abc" //在内存中存储在常量区中  --->  更改内存：常量区 'c' ---> 'd'
字符串"abc"引用指向内存常量--->   "abd"
```
## s3="abd" & s4="abc"
```java
public String(String original) {
    this.value = original.value;                    //指向同一块内存
    this.hash = original.hash;
}
public String(char value[]) {
    this.value = Arrays.copyOf(value, value.length);//新建一块内存存储当前值
}
public static char[] copyOf(char[] original, int newLength) {
    char[] copy = new char[newLength];
    System.arraycopy(original, 0, copy, 0,
                     Math.min(original.length, newLength));
    return copy;
}
```


---
# 反射修改常量值
>不能修改的原因是编译期替换

## 避免编译期替换
1. 构造方法中对常量赋值
```java
class Test{
  private final String FINAL_STR;
  public Test(){
    this.FINAL_STR="FINAL";
  }
}
```
2. 三目运算符
```java
class Test{
  private final String FINAL_STR = null == null ? "FINAL" :"";
  public Test(){
  }
}
```
