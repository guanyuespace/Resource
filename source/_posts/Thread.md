---
title: Thread
date: 2019-03-11 13:37:31
categories:
- Java
- SourceCode
- Thread
tags:
- Java
- Thread
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
>BlockingQueue implementations are designed to be used primarily for producer-consumer queues, but additionally support the java.util.Collection interface.
<!-- afternoon, it's time to keep my promise. 16:18:55 -->

- wait for the queue to become non-empty when retrieving an element; become available in the queue storing an element.
- A `BlockingQueue` does not accept null elements.(Implementations throw `NullPointerException` on attempts to `add`, `offer` a `null`.A `null` element is used as a sentinel value to indicate failure of `poll` operations.) <!-- sentinel 哨兵，哨位节点 -->
- `BlockingQueue` implementations are thread-safe.However, the bulk Collection operations `addAll`, `containsAll`, `retainAll` and `removeAll` are not necessarily performed atomically unless specified otherwise in an implementation.<!-- So it is possible, for example, for `addAll(c)` to fail (throwing an exception) after adding only some of the elements in c}.-->
- `BlockingQueue` methods come in four forms:
  1. throws an exception
  2. returns a special value (either null or false, depending on the operation)
  3. blocks the current thread indefinitely until the operation can succeed
  4. blocks for only a given maximum time limit before giving up.

## Summary of BlockingQueue methods

| Func/Result | Throws exception | Special value | Blocks | Times out |
|---|---|---|---|---|
| Insert  | add add(e)   | offer offer(e)  | put put(e)  | offer(Object, long, TimeUnit) offer(e, time, unit) |
| Remove  | remove remove()   | poll poll()   | take take() | poll(long, TimeUnit) poll(time, unit)  |
| Examine | element element() | peek peek()   | not applicable  | not applicable  |


- when use a capacity-restricted queue, it's preferable to use the `offer` method to add a element than `add`.

## Code
```java
public interface BlockingQueue<E> extends Queue<E> {...}
```

### SubClass
#### ArrayBlockingQueue
>A bounded `BlockingQueue` backed by an array. This queue orders elements FIFO (first-in-first-out).
>head --&gt; out   tail --&gt; in  <!-- the new element will be added in the tail. -->


This class supports an optional fairness policy for ordering waiting producer and consumer threads.
By default, this ordering is not guaranteed.<!-- default{fair=false;}-->
However, a queue constructed with fairness set to {@code true} grants threads access in FIFO order.
Fairness generally decreases throughput but reduces variability and avoids starvation.

<!-- 提供了公平锁策略，但是默认false。 容量固定。 -->

```java
public class ArrayBlockingQueue<E> extends AbstractQueue<E> implements BlockingQueue<E>, java.io.Serializable {
  /** Main lock guarding all access */
  final ReentrantLock lock;
  /** Condition for waiting takes */
  private final Condition notEmpty;
  /** Condition for waiting puts */
  private final Condition notFull;

  /**
    * Creates an {@code ArrayBlockingQueue} with the given (fixed)
    * capacity, the specified access policy and initially containing the
    * elements of the given collection,
    * added in traversal order of the collection's iterator.
    *
    * @param capacity the capacity of this queue
    * @param fair if {@code true} then queue accesses for threads blocked
    *        on insertion or removal, are processed in FIFO order;
    *        if {@code false} the access order is unspecified.
    * @param c the collection of elements to initially contain
    * @throws IllegalArgumentException if {@code capacity} is less than
    *         {@code c.size()}, or less than 1.
    * @throws NullPointerException if the specified collection or any
    *         of its elements are null
    */
   public ArrayBlockingQueue(int capacity, boolean fair, Collection<? extends E> c) {
     this(capacity, fair);

     final ReentrantLock lock = this.lock;
     lock.lock(); // Lock only for visibility, not mutual exclusion
     try {
         int i = 0;
         try {
             for (E e : c) {
                 checkNotNull(e);
                 items[i++] = e;
             }
         } catch (ArrayIndexOutOfBoundsException ex) {
             throw new IllegalArgumentException();
         }
         count = i;
         putIndex = (i == capacity) ? 0 : i;// putIndex --> tail
     } finally {
         lock.unlock();
     }
   }
   public ArrayBlockingQueue(int capacity, boolean fair) {
    if (capacity <= 0)
        throw new IllegalArgumentException();
    this.items = new Object[capacity];
    lock = new ReentrantLock(fair);
    notEmpty = lock.newCondition();
    notFull =  lock.newCondition();
  }
  public ArrayBlockingQueue(int capacity) {
    this(capacity, false);
  }

  public void put(E e) throws InterruptedException {
      checkNotNull(e);
      final ReentrantLock lock = this.lock;
      lock.lockInterruptibly();//Acquires the lock unless the current thread is interrupted.
      try {
          while (count == items.length)
              notFull.await();
          enqueue(e);//insert
      } finally {
          lock.unlock();
      }
  }
  //insert the element
  private void enqueue(E x) {
      // assert lock.getHoldCount() == 1;
      // assert items[putIndex] == null;
      final Object[] items = this.items;
      items[putIndex] = x;
      if (++putIndex == items.length)
          putIndex = 0;
      count++;
      notEmpty.signal();
  }

  public E take() throws InterruptedException {
      final ReentrantLock lock = this.lock;
      lock.lockInterruptibly();
      try {
          while (count == 0)
              notEmpty.await();
          return dequeue();
      } finally {
          lock.unlock();
      }
  }
  private E dequeue() {
      final Object[] items = this.items;
      @SuppressWarnings("unchecked")
      E x = (E) items[takeIndex];
      items[takeIndex] = null;
      if (++takeIndex == items.length)
          takeIndex = 0;
      count--;
      if (itrs != null)
          itrs.elementDequeued();
      notFull.signal();
      return x;
  }
}
```

#### LinkedBlockingQueue
>FIFO: head --&gt; out  tail --&gt; in
>Linked queues typically have higher throughput than array-based queues but less predictable performance in most concurrent applications.<!-- 更高的吞吐量，更低的predictable -->

- The optional capacity bound constructor argument serves as a way to prevent excessive queue expansion. The capacity, if unspecified, is equal to Integer#MAX_VALUE.  Linked nodes are dynamically created upon each insertion unless this would bring the queue above capacity.
<!-- 可选容量以避免过度增长。 容量不固定。  -->

```java
public class LinkedBlockingQueue<E> extends AbstractQueue<E> implements BlockingQueue<E>, java.io.Serializable {
 /**
  * A variant of the "two lock queue" algorithm.双锁算法的变体。  The putLock gates
  * entry to put (and offer), and has an associated condition for waiting puts.
  * Similarly for the takeLock.
  */
  /** Lock held by take, poll, etc */
  private final ReentrantLock takeLock = new ReentrantLock();
  /** Wait queue for waiting takes */
  private final Condition notEmpty = takeLock.newCondition();

  /** Lock held by put, offer, etc */
  private final ReentrantLock putLock = new ReentrantLock();
  /** Wait queue for waiting puts */
  private final Condition notFull = putLock.newCondition();


  public LinkedBlockingQueue() {
      this(Integer.MAX_VALUE);
  }
  public LinkedBlockingQueue(int capacity) {
      if (capacity <= 0) throw new IllegalArgumentException();
      this.capacity = capacity;
      last = head = new Node<E>(null);
  }
  public LinkedBlockingQueue(Collection<? extends E> c) {
      this(Integer.MAX_VALUE);
      final ReentrantLock putLock = this.putLock;
      putLock.lock(); // Never contended, but necessary for visibility
      try {
          int n = 0;
          for (E e : c) {
              if (e == null)
                  throw new NullPointerException();
              if (n == capacity)
                  throw new IllegalStateException("Queue full");
              enqueue(new Node<E>(e));//enqueue(node) =>{last= last.next = node;}
              ++n;
          }
          count.set(n);//AtomicInteger count;
      } finally {
          putLock.unlock();
      }
  }

  public void put(E e) throws InterruptedException {
      if (e == null) throw new NullPointerException();
      // Note: convention in all put/take/etc is to preset local var
      // holding count negative to indicate failure unless set.
      int c = -1;
      Node<E> node = new Node<E>(e);
      final ReentrantLock putLock = this.putLock;
      final AtomicInteger count = this.count;
      putLock.lockInterruptibly();
      try {
          /*
           * Note that count is used in wait guard even though it is not protected by lock.
           * This works because count can only decrease at this point (all other puts are shut
           * out by lock), and we (or some other waiting put) are
           * signalled if it ever changes from capacity.
           * Similarly for all other uses of count in other wait guards.
           */
          while (count.get() == capacity) {
              notFull.await();
          }
          enqueue(node);
          c = count.getAndIncrement();
          if (c + 1 < capacity)
              notFull.signal();
      } finally {
          putLock.unlock();
      }
      if (c == 0)
          signalNotEmpty();
  }
  public E take() throws InterruptedException {
      E x;
      int c = -1;
      final AtomicInteger count = this.count;
      final ReentrantLock takeLock = this.takeLock;
      takeLock.lockInterruptibly();
      try {
          while (count.get() == 0) {
              notEmpty.await();
          }
          x = dequeue();
          c = count.getAndDecrement();
          if (c > 1)
              notEmpty.signal();
      } finally {
          takeLock.unlock();
      }
      if (c == capacity)
          signalNotFull();
      return x;
  }
}
```

#### PriorityBlockingQueue
```java
public class PriorityBlockingQueue<E> extends AbstractQueue<E>
    implements BlockingQueue<E>, java.io.Serializable {

    public PriorityBlockingQueue(int initialCapacity,
                                 Comparator<? super E> comparator) {
        if (initialCapacity < 1)
            throw new IllegalArgumentException();
        this.lock = new ReentrantLock();
        this.notEmpty = lock.newCondition();
        this.comparator = comparator;
        this.queue = new Object[initialCapacity];
    }
    public PriorityBlockingQueue(Collection<? extends E> c) {
      this.lock = new ReentrantLock();
      this.notEmpty = lock.newCondition();
      boolean heapify = true; // true if not known to be in heap order
      boolean screen = true;  // true if must screen for nulls
      if (c instanceof SortedSet<?>) {
          SortedSet<? extends E> ss = (SortedSet<? extends E>) c;
          this.comparator = (Comparator<? super E>) ss.comparator();
          heapify = false;
      }
      else if (c instanceof PriorityBlockingQueue<?>) {
          PriorityBlockingQueue<? extends E> pq =
              (PriorityBlockingQueue<? extends E>) c;
          this.comparator = (Comparator<? super E>) pq.comparator();
          screen = false;
          if (pq.getClass() == PriorityBlockingQueue.class) // exact match
              heapify = false;
      }
      Object[] a = c.toArray();
      int n = a.length;
      // If c.toArray incorrectly doesn't return Object[], copy it.
      if (a.getClass() != Object[].class)
          a = Arrays.copyOf(a, n, Object[].class);
      if (screen && (n == 1 || this.comparator != null)) {
          for (int i = 0; i < n; ++i)
              if (a[i] == null)
                  throw new NullPointerException();
      }
      this.queue = a;
      this.size = n;
      if (heapify)
          heapify();//堆,化：初始建堆
  }
  //Establishes the heap invariant (described above) in the entire tree, assuming nothing about the order of the elements prior to the call.
  private void heapify() {
      Object[] array = queue;
      int n = size;
      int half = (n >>> 1) - 1;//最大的父节点序列
      Comparator<? super E> cmp = comparator;
      if (cmp == null) {
          for (int i = half; i >= 0; i--)//堆： i[0:n-1];  i{2*i+1, 2*i+2}
              siftDownComparable(i, (E) array[i], array, n);//建堆
      }
      else {
          for (int i = half; i >= 0; i--)
              siftDownUsingComparator(i, (E) array[i], array, n, cmp);//依据给定的比较方式建堆
      }
  }

  private static <T> void siftDownComparable(int k, T x, Object[] array, int n) {
      if (n > 0) {
          Comparable<? super T> key = (Comparable<? super T>)x;
          int half = n >>> 1;           // loop while a non-leaf
          while (k < half) {
              int child = (k << 1) + 1; // assume left child is least
              Object c = array[child];
              int right = child + 1;
              if (right < n &&
                  ((Comparable<? super T>) c).compareTo((T) array[right]) > 0)
                  c = array[child = right];
              if (key.compareTo((T) c) <= 0)
                  break;
              array[k] = c;
              k = child;
          }
          array[k] = key;
      }
  }
  private static <T> void siftDownUsingComparator(int k, T x, Object[] array,
                                                    int n,
                                                  Comparator<? super T> cmp) {
      if (n > 0) {
          int half = n >>> 1;
          while (k < half) {
              int child = (k << 1) + 1;
              Object c = array[child];
              int right = child + 1;
              if (right < n && cmp.compare((T) c, (T) array[right]) > 0)
                  c = array[child = right];
              if (cmp.compare(x, (T) c) <= 0)
                  break;
              array[k] = c;
              k = child;
          }
          array[k] = x;
      }
  }
}
```

<!-- Later -->


# CountDownLatch
>完成先决条件latch.countdown()后并行执行latch.await()下的线程

latch.countdown() can be called in child-thread.

```java
/**
* Decrements the count of the latch, releasing all waiting threads if
* the count reaches zero.
*
* <p>If the current count is greater than zero then it is decremented.
* If the new count is zero then all waiting threads are re-enabled for
* thread scheduling purposes.
*
* <p>If the current count equals zero then nothing happens.
*/
public void countDown() {
  sync.releaseShared(1);//private final Sync sync;
}
/**
 * Implements interruptible condition wait.
 * <ol>
 * <li> If current thread is interrupted, throw InterruptedException.
 * <li> Save lock state returned by {@link #getState}.
 * <li> Invoke {@link #release} with saved state as argument,
 *      throwing IllegalMonitorStateException if it fails.
 * <li> Block until signalled or interrupted.
 * <li> Reacquire by invoking specialized version of
 *      {@link #acquire} with saved state as argument.
 * <li> If interrupted while blocked in step 4, throw InterruptedException.
 * </ol>
 */
public final void await() throws InterruptedException {
    if (Thread.interrupted())
        throw new InterruptedException();
    Node node = addConditionWaiter();
    int savedState = fullyRelease(node);
    int interruptMode = 0;
    while (!isOnSyncQueue(node)) {
        LockSupport.park(this);
        if ((interruptMode = checkInterruptWhileWaiting(node)) != 0)
            break;
    }
    if (acquireQueued(node, savedState) && interruptMode != THROW_IE)
        interruptMode = REINTERRUPT;
    if (node.nextWaiter != null) // clean up if cancelled
        unlinkCancelledWaiters();
    if (interruptMode != 0)
        reportInterruptAfterWait(interruptMode);
}
```
