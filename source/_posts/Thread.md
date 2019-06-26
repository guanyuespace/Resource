---
title: Thread
date: 2019-03-11 13:37:31
categories:
tags:
---
# Thread
```java
public class Thread implements Runnable{...}
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
    /////////////////ThreadLocal////////////////////////////
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


    //////////////////////priority//////////////////////
    private int priority;//
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


    public final synchronized void join(long millis)
    throws InterruptedException {
        long base = System.currentTimeMillis();
        long now = 0;

        if (millis < 0) {
            throw new IllegalArgumentException("timeout value is negative");
        }

        if (millis == 0) {
            while (isAlive()) {
                wait(0);/////this.wait()
                //如果输入 0 不代表不暂停，而是需要特殊情况自己苏醒或者 notify 唤醒，这里有个特殊点，wait(0) 是可以自己苏醒的。
            }
        } else {
            while (isAlive()) {
                long delay = millis - now;
                if (delay <= 0) {
                    break;
                }
                wait(delay);//ok  current thread wait...
                now = System.currentTimeMillis() - base;
            }
        }
    }
    ///////////////native methods////////////////////////////
    public static native void sleep(long millis) throws InterruptedException;
    public static native void yield();

    private native void resume0();//resume() has been Deprecated
    private native void suspend0();//suspend() has been Deprecated
    // others ......
}
```
<!-- Later ... ... -->

### sleep & yield
[参考](https://www.jianshu.com/p/0964124ae822)
<!-- jvm.cpp 理解... ... 操作？？？ -->

### wait



Object.c

```c
static JNINativeMethod methods[] = {
    {"hashCode",    "()I",                    (void \*)&JVM_IHashCode},
    {"wait",        "(J)V",                   (void \*)&JVM_MonitorWait},/\* java method -- JNI-Methods \*/
    {"notify",      "()V",                    (void \*)&JVM_MonitorNotify},
    {"notifyAll",   "()V",                    (void \*)&JVM_MonitorNotifyAll},
    {"clone",       "()Ljava/lang/Object;",   (void \*)&JVM_Clone},
};
```

jvm.h

```c
JNIEXPORT void JNICALL
JVM_MonitorWait(JNIEnv \*env, jobject obj, jlong ms);

JNIEXPORT void JNICALL
JVM_Yield(JNIEnv \*env, jclass threadClass);/* implementation where? */

JNIEXPORT void JNICALL
JVM_Sleep(JNIEnv \*env, jclass threadClass, jlong millis);/* implementation where? */
```

~~jvm.cpp~~

### Test

```java
class Producer extends Thread {
    @Override
    public void run() {
        System.out.println("producer start");
        System.out.println("producer end");
    }
}

public class Test {
    public static void main(String[] args) throws InterruptedException {
        Consumer consumer = new Consumer();
        consumer.setPriority(1);
        Producer producer = new Producer();
        producer.setPriority(10);

        consumer.start();
        Thread.sleep(20);
        consumer.myJoin();/// current thread --main-- wait
        producer.start();
        System.out.println("main end");
    }
}

class Consumer extends Thread {
    @Override
    public synchronized void run() {
        System.out.println("consumer start");
        //do others
        System.out.println("start sleep 2000");
        try {
            Thread.sleep(2000);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
        System.out.println("consumer end");
    }

    synchronized void myJoin() throws InterruptedException {
        System.out.println(this.getClass());
        System.out.println("my join start");
        while (this.isAlive()) {
            System.out.println("current thread: "+Thread.currentThread().getName());
            wait(0);//释放consumer.this的锁对象, main-thread wait(0)  here ... ... , consumer子线程获取所对象执行, 当consumer子线程退出后自动唤醒main
        }
        System.out.println("my join end");
    }
}
```

# BlockingQueue

# ThreadExecutorPool
