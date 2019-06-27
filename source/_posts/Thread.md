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
    public final native void wait(long timeout) throws InterruptedException;
    public static native void sleep(long millis) throws InterruptedException;
    public static native void yield();

    private native void resume0();//resume() has been Deprecated
    private native void suspend0();//suspend() has been Deprecated
    // others ......
}
```

由代码可知join的内部实现由wait(mills)实现
wait(),sleep(),yield()均由C/C++代码（JNI）实现




### sleep & yield
>[yield与sleep源码解析](https://guanyuespace.github.io/temp/thread/simpread-Java%20%E7%BA%BF%E7%A8%8B%E6%BA%90%E7%A0%81%E8%A7%A3%E6%9E%90%E4%B9%8B%20yield%20%E5%92%8C%20sleep.html)
>[Java 线程源码解析之 yield 和 sleep](https://www.jianshu.com/p/0964124ae822)

<!-- jvm.cpp 理解... ... 操作？？？ -->
由于 `Thread` 的 `yield` 和 `sleep` 有一定的相似性，因此放在一起进行分析。`yield` 会释放 CPU 资源，~~让优先级更高（至少是相同）的线程获得执行机会；~~ `sleep` 当传入参数为 0 时，和 `yield` 相同；当传入参数大于 0 时，也是释放 CPU 资源，但可以让其它任何优先级的线程获得执行机会；

基于优先级的调度程序使用一种有优先权的方式实现，这意味着当一个有更高优先权的线程到来时，无论低优先级的线程是否在运行，都会中断 (抢占) 它。这个约定对于操作系统来说并不总是这样，*这意味着操作系统有时可能会选择运行一个更低优先级的线程。*

---

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
        System.out.println("current thread: " + Thread.currentThread().getName() + "\t my join start");
        while (this.isAlive()) {
            System.out.println("current thread: " + Thread.currentThread().getName());
            wait(0);//释放consumer.this的锁对象, main stop here ... ... , consumer子线程获取所对象执行, 当consumer子线程退出后自动唤醒main
        }
        System.out.println("current thread: " + Thread.currentThread().getName() + "\t my join end");
    }
}
```

**Result：**

```
consumer start
start sleep 2000
consumer end
class thread.Consumer
current thread: main	 my join start
current thread: main
current thread: main	 my join end
main end
producer start
producer end
```

# BlockingQueue
>。。。



# ThreadExecutorPool
