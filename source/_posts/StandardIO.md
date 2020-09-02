---
title: Java程序输入输出
date: 2020-09-02 12:20:09
categories:
- Java
- IO
tags:
- Java
- IO
---

<!-- TOC depthFrom:1 depthTo:6 withLinks:1 updateOnSave:1 orderedList:0 -->

	- [前言](#前言)
	- [实操](#实操)
		- [编码环境](#编码环境)
		- [Paragram Params](#paragram-params)
		- [标准输入System.in](#标准输入systemin)
			- [System类回顾](#system类回顾)
		- [封装System.in](#封装systemin)
			- [Scanner简述](#scanner简述)
			- [Scanner类常用方法](#scanner类常用方法)

<!-- /TOC -->

## 前言
最近突然收到好友提问如何在IDE里想Java程序传递参数，首先想到了Paragram Params,但是具体位置及基本配置竟一时无法想起。。。~近一年来基本没有进行编码，悲哀。~

## 实操

### 编码环境
IDE: IntelliJ IDEA


### Paragram Params

Run(<kbd>Alt+Shift+F10</kbd>) --&gt; Edit Config --&gt;

![](./Config.jpg "一目了然")

>VM Options,Environment variables, Redicct input 等配置

### 标准输入System.in

The "standard" input stream. **This stream is already open and ready to supply input data.** Typically this stream corresponds to ***keyboard input*** or ***another input source specified by the host environment or user.***   <!-- anpther input: 参数设置重定向输入-->


#### System类回顾

Among the facilities provided by the <code>System</code> class are standard input, standard output, and error output streams; access to externally defined properties and environment variables; a means of loading files and libraries; and a utility method for quickly copying a portion of an array.

- init

```java
/**
 * Initialize the system class.  Called after thread initialization.
 */
private static void initializeSystemClass() {

    // VM might invoke JNU_NewStringPlatform() to set those encoding sensitive properties (user.home, user.name, boot.class.path, etc.) during "props" initialization,
    //in which it may need access, via System.getProperty(), to the related system encoding property that have been initialized (put into "props") at early stage of the initialization.

    //So make sure the "props" is available at the very beginning of the initialization and all system properties to be put into it directly.
    props = new Properties();
    initProperties(props);  // initialized by the VM

    // There are certain system configurations that may be controlled by
    // *** VM options such as the maximum amount of direct memory and Integer cache size used to support the object identity semantics of autoboxing.***
    //  Typically, the library will obtain these values from the properties set by the VM.  
    //
    // If the properties are for internal implementation use only, these properties should be removed from the system properties.
    //
    // See java.lang.Integer.IntegerCache and the sun.misc.VM.saveAndRemoveProperties method for example.
    //
    // Save a private copy of the system properties object that
    // can only be accessed by the internal implementation.  Remove
    // certain system properties that are not intended for public access.
    sun.misc.VM.saveAndRemoveProperties(props);


    lineSeparator = props.getProperty("line.separator");
    sun.misc.Version.init();

    // private static native long set(int d);      //又是native方法... 底层实现
    // private static FileDescriptor standardStream(int fd) {...}
    FileInputStream fdIn = new FileInputStream(FileDescriptor.in);   
        FileDescriptor desc = new FileDescriptor();
        desc.handle = set(fd);
        return desc;
    }
    FileOutputStream fdOut = new FileOutputStream(FileDescriptor.out);
    FileOutputStream fdErr = new FileOutputStream(FileDescriptor.err);
    setIn0(new BufferedInputStream(fdIn));
    setOut0(newPrintStream(fdOut, props.getProperty("sun.stdout.encoding")));
    setErr0(newPrintStream(fdErr, props.getProperty("sun.stderr.encoding")));

    // Load the zip library now in order to keep java.util.zip.ZipFile from trying to use itself to load this library later.
    loadLibrary("zip");

    // Setup Java signal handlers for HUP, TERM, and INT (where available).
    Terminator.setup();

    // Initialize any miscellenous operating system settings that need to be set for the class libraries.
    // Currently this is no-op everywhere except for Windows where the process-wide error mode is set before the java.io classes are used.
    sun.misc.VM.initializeOSEnvironment();

    // The main thread is not added to its thread group in the same
    // way as other threads; we must do it ourselves here.
    Thread current = Thread.currentThread();
    current.getThreadGroup().add(current);

    // register shared secrets
    setJavaLangAccess();

    // Subsystems that are invoked during initialization can invoke
    // sun.misc.VM.isBooted() in order to avoid doing things that should
    // wait until the application class loader has been set up.
    // IMPORTANT: Ensure that this remains the last initialization action!
    sun.misc.VM.booted();
}
```

- standard in, standard out, standard err
> 以System.in为例介绍

 ```java
public final static InputStream in = null; //The "standard" input stream. This stream is already open and ready to supply input data.
/**
 * Reassigns the "standard" input stream.
 * <p>First, if there is a security manager, its <code>checkPermission</code>
 * method is called with a <code>RuntimePermission("setIO")</code> permission
 * to see if it's ok to reassign the "standard" input stream.
 */
public static void setIn(InputStream in) {}
```

- externally defined properties and environment variables
- a utility method for quickly copying a portion of an array

```java
/**
 * Copies an array from the specified source array, beginning at the
 * specified position, to the specified position of the destination array.
 */
 public static native void arraycopy(Object src,  int  srcPos,
                                        Object dest, int destPos,
                                        int length);


// 使用范例： Arrays.copyOf  以当前数组为模板增删创建一个*新*数组   
// Copies the specified array, truncating or padding with <tt>false</tt> (if necessary) so the copy has the specified length.      
// For all indices that are valid in both the original array and the copy, the two arrays will contain identical values. 相同元素
//
// For any indices that are valid in the copy but not the original, the copy will contain <tt>false</tt>.          &lt--  padding  with false
// Such indices will exist if and only if the specified length is greater than that of the original array.                              
// public static boolean[] copyOf(boolean[] original, int newLength) {
//         boolean[] copy = new boolean[newLength];
//         System.arraycopy(original, 0, copy, 0,
//                          Math.min(original.length, newLength));
//         return copy;
// }
```

### 封装System.in

```java
 Scanner scanner = new Scanner(System.in);
 scanner.nextInt(16);//  获取一个16进制的int
```


#### Scanner简述
> 如果仅仅获取输入可以对System.in进行自定义的任何封装

A <code>Scanner</code> breaks its input into tokens using a delimiter pattern, which by default matches whitespace.
The resulting tokens may then be converted into values of different types using the various <tt>next</tt> methods.

**The scanner can also use delimiters other than whitespace.** This example reads several items in from a string:

```java
String input = "1 fish 2 fish red fish blue fish";
Scanner s = new Scanner(input).useDelimiter("\\s*fish\\s*");
System.out.println(s.nextInt());
System.out.println(s.nextInt());
System.out.println(s.next());
System.out.println(s.next());
s.close();
```
> result: 1     2     red     blue


**The same output can be generated with this code, which uses a regular expression to parse all four tokens at once**:

```java
String input = "1 fish 2 fish red fish blue fish";
Scanner s = new Scanner(input);
s.findInLine("(\\d+) fish (\\d+) fish (\\w+) fish (\\w+)");
MatchResult result = s.match();
for (int i=1; i<=result.groupCount(); i++)
    System.out.println(result.group(i));
s.close();
```

- The default whitespace delimiter used by a scanner is as recognized by  <a>java.lang.Character</a>.<a>java.lang.Character#isWhitespace(char)</a> isWhitespace.

- The <a>reset</a> method will reset the value of the scanner's delimiter to the default whitespace delimiter regardless of whether it was previously changed.

#### Scanner类常用方法

```java
/**
 * Finds and returns the next complete token from this scanner.
 * A complete token is preceded and followed by input that matches the delimiter pattern.
 *
 * This method may block while waiting for input to scan, even if a previous invocation of {@link #hasNext} returned <code>true</code>.
 */
public String next() {
    ensureOpen();
    clearCaches();

    while (true) {
        /**
         * getCompleteTokenInBuffer
         * Triple return:
         * 1. valid string means it was found
         * 2. null with needInput=false means we won't ever find it
         * 3. null with needInput=true means try again after readInput
         */
        String token = getCompleteTokenInBuffer(null); //Returns a "complete token" that matches the specified pattern
        if (token != null) {
            matchValid = true;
            skipped = false;
            return token;
        }
        if (needInput)
            readInput(); // Tries to read more input. May block.
        else
            throwFor(); // Throw Exception
    }
}
private String getCompleteTokenInBuffer(Pattern pattern) {
    // Skip delims first
    matcher.usePattern(delimPattern);
    //... 跳过
    // If we are sitting at the end, no more tokens in buffer
    //...
    //...


    //  Attempt to match against the desired pattern
    matcher.usePattern(pattern);
    matcher.region(position, tokenEnd);
    if (matcher.matches()) {
        String s = matcher.group();
        position = matcher.end();
        return s;
    } else { // Complete token but it does not match
        return null;
    }

}

public Scanner reset() {
    delimPattern = WHITESPACE_PATTERN;
    useLocale(Locale.getDefault(Locale.Category.FORMAT));
    useRadix(10);
    clearCaches();
    return this;
}
```
