---
title: Thread
date: 2019-03-11 13:37:31
categories:
tags:
---
# Thread
```java
public class Thread implements Runnable
```

<!-- more -->
## Runnable
```java
package java.lang;

@FunctionalInterface
public interface Runnable {
    void run();
}
```

## Thread
```java
public class Thread implements Runnable {
    private volatile String name;//
    private int priority;//
    private Thread threadQ;//////
    private long eetop;//////////
    private boolean single_step;//////////
    private boolean daemon = false;//
    private boolean stillborn = false;//
    private Runnable target;//
    private ThreadGroup group;//////////////
    private ClassLoader contextClassLoader;//
    private AccessControlContext inheritedAccessControlContext;//
    private static int threadInitNumber;//
    ThreadLocalMap threadLocals = null;///////
    ThreadLocalMap inheritableThreadLocals = null;///////
    private long stackSize;//////
    private long nativeParkEventPointer;////
    private long tid;//
    private static long threadSeqNumber;//
    private volatile int threadStatus = 0;///////
    volatile Object parkBlocker;/////
    private volatile Interruptible blocker;//////
    private final Object blockerLock = new Object();//
    public static final int MIN_PRIORITY = 1;
    public static final int NORM_PRIORITY = 5;
    public static final int MAX_PRIORITY = 10;
    private static final StackTraceElement[] EMPTY_STACK_TRACE;//////
    private static final RuntimePermission SUBCLASS_IMPLEMENTATION_PERMISSION;//
    private volatile Thread.UncaughtExceptionHandler uncaughtExceptionHandler;//
    private static volatile Thread.UncaughtExceptionHandler// defaultUncaughtExceptionHandler;
    @Contended("tlr")
    long threadLocalRandomSeed;
    @Contended("tlr")
    int threadLocalRandomProbe;
    @Contended("tlr")
    int threadLocalRandomSecondarySeed;//

}
```


Later ... ...
