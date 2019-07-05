# AQS框架--AbstractQueuedSynchronizer
>ReentrantReadWriteLock, CountDownLatch, ReentrantLock...

<!-- more -->

## code

```java
/**
 * Provides a framework for implementing blocking locks and related synchronizers (semaphores, events, etc) that rely on first-in-first-out (FIFO) wait queues.
 * This class is designed to be a useful basis for most kinds of synchronizers that rely on a single atomic {@code int} value to represent state.
 *
 * Subclasses must define the protected methods that change this state, and which define what that state means in terms of this object being acquired or released.
 *
 * Given these, the other methods in this class carry out all queuing and blocking mechanics.
 *
 * Subclasses can maintain other state fields, but only the atomically updated {@code int} value manipulated using methods {@link #getState}, {@link #setState} and {@link #compareAndSetState} is tracked with respect to synchronization.
 */
public abstract class AbstractQueuedSynchronizer extends AbstractOwnableSynchronizer implements java.io.Serializable {
/**
 * Condition implementation for a {@link AbstractQueuedSynchronizer} serving as the basis of a {@link Lock} implementation.
 *
 * <p>Method documentation for this class describes mechanics, not behavioral specifications from the point of view of Lock and Condition users.
 * Exported versions of this class will in general need to be accompanied by documentation describing condition semantics that rely on those of the associated {@code AbstractQueuedSynchronizer}.
 *
 * <p>This class is Serializable, but all fields are transient, so deserialized conditions have no waiters.
 */
  public class ConditionObject implements Condition, java.io.Serializable {
  }
 /**
  * Wait queue node class.
  * The wait queue is a variant of a "CLH" (Craig, Landin, and Hagersten) lock queue. FIFO queue.<!-- CLH锁队列的变体 -->
  *
  * CLH锁常用于自旋锁，不用他们来完成同步而是用来保存前一个线程的控制信息
  * "status"域用来跟踪一个线程是否应当block("status"域不用来控制线程是否被用来授予锁🔒),当一个node的前驱结点释放锁它才会被唤醒
  * Each node of the queue otherwise serves as a specific-notification-style monitor holding a single waiting thread.
  *
  * 队列中的第一个会尽可能的获取锁🔒，但是无法保证一定成功，仅仅有可能罢了。
  *
  */
  static final class Node {
     /** Marker to indicate a node is waiting in shared mode */
     static final Node SHARED = new Node();//the value for nextWaiter
     /** Marker to indicate a node is waiting in exclusive mode */
     static final Node EXCLUSIVE = null;//the value for nextWaiter

     /** waitStatus value to indicate thread has cancelled */
     static final int CANCELLED =  1;
     /** waitStatus value to indicate successor's thread needs unparking唤醒 */
     static final int SIGNAL    = -1;
     /** waitStatus value to indicate thread is waiting on condition */
     static final int CONDITION = -2;
     /** waitStatus value to indicate the next acquireShared should unconditionally propagate 无条件的propagate传播 */
     static final int PROPAGATE = -3;

     // the above value
     volatile int waitStatus;
     volatile Node prev;//前驱结点
     volatile Node next;//后继节点

     // The thread that enqueued this node. Initialized on construction and nulled out after use.
     volatile Thread thread;

     /**
      * Link to next node waiting on condition, or the special value SHARED.
      * 指向下一个等待在某一条件或者特定值--SHARED的下一节点
      * Because condition queues are accessed only when holding in exclusive mode, we just need a simple linked queue to hold nodes while they are waiting on conditions.
      * 由于条件队列只有在独占模式下可以被访问，当节点等待条件时，我们只需要一个简单的队列来保存节点
      * They are then transferred to the queue to re-acquire.
      * 然后将它们转移到队列以重新获取。
      * And because conditions can only be exclusive, we save a field by using special value to indicate shared mode.
      * 由于条件只能是排他的，所以我们使用特殊值来表示共享模式来保存字段。
      */
     Node nextWaiter;
     final boolean isShared() { return nextWaiter == SHARED;}
     final Node predecessor() throws NullPointerException{ prev==null?throw new NullPointerException:return prev;}

     Node() {    // Used to establish initial head or SHARED marker
     }

    Node(Thread thread, Node mode) {     // Used by addWaiter...联系ReentrantReadWriteLock:等待队列中写锁/读锁
        this.nextWaiter = mode;
        this.thread = thread;
    }

    Node(Thread thread, int waitStatus) { // Used by Condition
        this.waitStatus = waitStatus;
        this.thread = thread;
    }
  }//end of Node
  //等待队列的首尾指针
  private transient volatile Node head;//lazily initialized
  private transient volatile Node tail;//lazily initialized

  // The synchronization state.
  private volatile int state;//联系--ReentrantReadWriteLock:读锁/写锁重入次数
  //get/setState()

  protected final boolean compareAndSetState(int expect, int update) {
      // See below for intrinsics setup to support this
      return unsafe.compareAndSwapInt(this, stateOffset, expect, update);//stateOffset = unsafe.objectFieldOffset(AbstractQueuedSynchronizer.class.getDeclaredField("state"));
  }
  /**
   * Sets head of queue to be node, thus dequeuing. Called only by acquire methods.
   * Also nulls out unused fields for sake of GC and to suppress unnecessary signals and traversals.
   * @param node the node
   */
  private void setHead(Node node) {
      head = node;
      node.thread = null;
      node.prev = null;
  }
  //队列操作
}
```

## Node
>入队节点，AQS队列

## Constants
- SHARED:共享锁--ReentrantReadWriteLock.ReadLock
- EXCLUSIVE:独占锁--ReentrantReadWriteLock.WriteLock

nextWaiter

### waitStatus

**Status field, taking on only the values:**
- **SIGNAL**: The successor of this node is (or will soon be) blocked (via park), so *the current node must unpark its successor when it releases or cancels. To avoid races, acquire methods must first indicate they need a signal, then retry the atomic acquire, and then, on failure, block.*
- **CANCELLED**:  *This node is cancelled due to timeout or interrupt*. Nodes never leave this state. In particular, a thread with cancelled node never again blocks.
- **CONDITION**:  This node is currently on a condition queue. *It will not be used as a sync queue node until transferred, at which time the status will be set to 0*. (Use of this value here has nothing to do with the other uses of the field, but simplifies mechanics.)
- **PROPAGATE**:  *A releaseShared should be propagated to other nodes*.<!-- 释放共享锁🔒信息的传播 --> This is set (for head node only) in doReleaseShared to ensure propagation continues, even if other operations have since intervened.
- **0**:  None of the above

## AbstractQueuedSynchronizer
>AQS队列操作

### 队列操作--Queuing utilities
```java
/**
 * Inserts node into queue, initializing if necessary. See picture above.
 * @param node the node to insert
 * @return node's predecessor
 */
private Node enq(final Node node) {
    for (;;) {
        Node t = tail;
        if (t == null) { // Must initialize head and tail.
            if (compareAndSetHead(new Node()))//unsafe.compareAndSwapObject(this, headOffset, null, update);
            //headOffset = unsafe.objectFieldOffset(AbstractQueuedSynchronizer.class.getDeclaredField("head"));
                tail = head;
        } else {//insert the node to the tail.
            node.prev = t;
            if (compareAndSetTail(t, node)) {//CAS: set tail=node.
                t.next = node;
                return t;
            }
        }
    }
}
/**
 * Creates and enqueues node for current thread and given mode.
 * @param mode Node.EXCLUSIVE for exclusive, Node.SHARED for shared
 * @return the new node
 */
private Node addWaiter(Node mode) {//加入到等待队列
    Node node = new Node(Thread.currentThread(), mode);
    // Try the fast path of enq; backup to full enq on failure
    Node pred = tail;
    if (pred != null) {//just try once.maybe succeed and then just be ok.
        node.prev = pred;
        if (compareAndSetTail(pred, node)) {//maybe other thread modify.example:other thread insert a node just when ..., so the tail will change.
            pred.next = node;
            return node;
        }
    }
    //try it with the result: failure, so while-loop enqueue.
    enq(node);
    return node;
}
/**
 * Wakes up node's successor, if one exists. 唤醒后继节点
 * @param node the node
 */
private void unparkSuccessor(Node node) {
    /*
     * If status is negative (i.e., possibly needing signal) try to clear in anticipation of signalling.
     * It is OK if this fails or if status is changed by waiting thread.
     */
    int ws = node.waitStatus;
    if (ws < 0)
        compareAndSetWaitStatus(node, ws, 0);//等待状态。。。

    /*
     * Thread to unpark is held in successor, which is normally just the next node.
     * But if cancelled or apparently null, traverse backwards from tail to find the actual non-cancelled successor.
     * 如果后继结点的等待状态为"CANCELLED"或者null,那么从后向前寻找，距离当前节点最近的non-cancelled后继节点
     */
    Node s = node.next;
    if (s == null || s.waitStatus > 0) {
        s = null;
        for (Node t = tail; t != null && t != node; t = t.prev)
            if (t.waitStatus <= 0)
                s = t;//覆盖更新为最近的non-cancelled节点
    }
    if (s != null)
        LockSupport.unpark(s.thread);//唤醒节点对应的Thread
}
/**
 * 释放共享锁--唤醒后继并且确保传播
 * Release action for shared mode -- signals successor and ensures propagation.
 * (Note: For exclusive mode, release just amounts to calling unparkSuccessor of head if it needs signal.)
 */
private void doReleaseShared() {
    /*
     * Ensure that a release propagates, even if there are other in-progress acquires/releases.
     * This proceeds in the usual way of trying to unparkSuccessor of head if it needs signal.唤醒首部node
     * But if it does not, status is set to PROPAGATE to ensure that upon release, propagation continues.否则释放共享锁信息传播
     * Additionally, we must loop in case a new node is added while we are doing this.循环以防节点插入
     * Also, unlike other uses of unparkSuccessor, we need to know if CAS to reset status fails, if so rechecking.
     */
    for (;;) {
        Node h = head;
        if (h != null && h != tail) {
            int ws = h.waitStatus;
            if (ws == Node.SIGNAL) {//唤醒首部
              //unsafe.compareAndSwapInt(node, ~~waitStatusOffset,~~ expect, update);
                if (!compareAndSetWaitStatus(h, Node.SIGNAL, 0))
                    continue;            // loop to recheck cases
                unparkSuccessor(h);//
            }
            else if (ws == 0 &&
                     !compareAndSetWaitStatus(h, 0, Node.PROPAGATE))
                continue;                // loop on failed CAS
        }
        if (h == head)                   // loop if head changed
            break;
    }
}
/**
 * Sets head of queue, and checks if successor may be waiting in shared mode, if so propagating if either propagate > 0 or PROPAGATE status was set.
 * 更新AQS队列头节点，唤醒后继节点为共享模式锁🔒
 * @param node the node
 * @param propagate the return value from a tryAcquireShared
 */
private void setHeadAndPropagate(Node node, int propagate) {
    Node h = head; // Record old head for check below
    setHead(node);
    /*
     * Try to signal next queued node if:
     *   Propagation was indicated by caller,
     *     or was recorded (as h.waitStatus either before
     *     or after setHead) by a previous operation
     *     (note: this uses sign-check of waitStatus because
     *      PROPAGATE status may transition to SIGNAL.)
     * and
     *   The next node is waiting in shared mode,
     *     or we don't know, because it appears null
     *
     * The conservatism in both of these checks may cause
     * unnecessary wake-ups, but only when there are multiple
     * racing acquires/releases, so most need signals now or soon
     * anyway.
     */
    if (propagate > 0 || h == null || h.waitStatus < 0 ||
        (h = head) == null || h.waitStatus < 0) {
        Node s = node.next;
        if (s == null || s.isShared())
            doReleaseShared();
    }
}
```

### 获取锁相关--Utilities for various versions of acquire
```java
/**
 * Cancels an ongoing attempt to acquire. 取消正在尝试获取🔒
 * @param node the node
 */
private void cancelAcquire(Node node) {
    // Ignore if node doesn't exist
    if (node == null)
        return;

    node.thread = null;//release

    // Skip cancelled predecessors
    Node pred = node.prev;
    while (pred.waitStatus > 0)//删去Canceled-node
        node.prev = pred = pred.prev;

    // predNext is the apparent node to unsplice.
    // CASes below will fail if not, in which case, we lost race vs another cancel or signal, so no further action is necessary.
    Node predNext = pred.next;//暂存旧的predNext

    // Can use unconditional write instead of CAS here.
    // After this atomic step, other Nodes can skip past us.
    // Before, we are free of interference from other threads.
    node.waitStatus = Node.CANCELLED;//设置当前节点状态CANCELLED

    // set pred next pointer.
    // If we are the tail, remove ourselves.
    if (node == tail && compareAndSetTail(node, pred)) {//remove the tail: node ,set tail= pred.
        compareAndSetNext(pred, predNext, null);//set pred--next-->null
    } else {
        // If successor needs signal, try to set pred's next-link so it will get one.
        // Otherwise wake it up to propagate.
        int ws;
        if (pred != head &&
            ((ws = pred.waitStatus) == Node.SIGNAL ||
             (ws <= 0 && compareAndSetWaitStatus(pred, ws, Node.SIGNAL))) &&
            pred.thread != null) {
            Node next = node.next;
            if (next != null && next.waitStatus <= 0)
                compareAndSetNext(pred, predNext, next);//set next pointer
        } else {//1. pred==head ok 首部唤醒 2. pred.waitStatus == Node.{~~SIGNAL~~, 0, PROPAGATE}, CAS failed so 唤醒？？  3. pred.thread==null pred刚释放，唤醒ok --->唤醒
            unparkSuccessor(node);
        }

        node.next = node; // help GC
    }
}
/**
 * Checks and updates status for a node that failed to acquire.
 * Returns true if thread should block.
 * This is the main signal control in all acquire loops.  Requires that pred == node.prev.
 * @param pred node's predecessor holding status
 * @param node the node
 * @return {@code true} if thread should block
 */
private static boolean shouldParkAfterFailedAcquire(Node pred, Node node) {
    int ws = pred.waitStatus;
    if (ws == Node.SIGNAL)
        // This node has already set status asking a release to signal it, so it can safely park.
        return true;
    if (ws > 0) {
        // Predecessor was cancelled. Skip over predecessors and indicate retry.
        do {
            node.prev = pred = pred.prev;
        } while (pred.waitStatus > 0);
        pred.next = node;
    } else {
        /*
         * waitStatus must be 0 or PROPAGATE.
         * Indicate that we need a signal, but don't park yet.
         * Caller will need to retry to make sure it cannot acquire before parking.  want to be signalled
         */
        compareAndSetWaitStatus(pred, ws, Node.SIGNAL);
    }
    return false;
}
//interrupt the thread to invoke park--status--“waitting”
static void selfInterrupt() {
    Thread.currentThread().interrupt();
}
//挂起当前线程
private final boolean parkAndCheckInterrupt() {
   LockSupport.park(this);
   return Thread.interrupted();
}
```
**Various flavors of acquire, varying in *exclusive/shared* and *control* modes**.

Each is mostly the same, but annoyingly different.
Only a little bit of factoring is possible due to interactions of exception mechanics (including ensuring that we cancel if tryAcquire throws exception) and other control, at least not without hurting performance too much.

### 加入等待队列
```java
/**
 * Acquires in exclusive uninterruptible mode for thread already in queue.非中断独占模式
 * Used by condition wait methods as well as acquire. 尝试获取独占🔒失败，加入AQS队列
 * @param node the node
 * @param arg the acquire argument
 * @return {@code true} if interrupted while waiting
 */
final boolean acquireQueued(final Node node, int arg) {
    boolean failed = true;
    try {
        boolean interrupted = false;
        for (;;) {
            final Node p = node.predecessor();
            if (p == head && tryAcquire(arg)) {//~~再次~~变成首部节点时再次尝试
                setHead(node);
                p.next = null; // help GC
                failed = false;
                return interrupted;
            }

            ////////等待/////////////////
            if (shouldParkAfterFailedAcquire(p, node) &&
                parkAndCheckInterrupt())
                interrupted = true;
        }
    } finally {
        if (failed)//err
            cancelAcquire(node);
    }
}
/**
 * Acquires in exclusive interruptible mode.中断独占模式
 * @param arg the acquire argument
 */
private void doAcquireInterruptibly(int arg)
    throws InterruptedException {
    final Node node = addWaiter(Node.EXCLUSIVE);//首先加入队列尾部
    boolean failed = true;
    try {
        for (;;) {
            final Node p = node.predecessor();
            if (p == head && tryAcquire(arg)) {
                setHead(node);
                p.next = null; // help GC
                failed = false;
                return;
            }
            ////////////等待//////////////
            if (shouldParkAfterFailedAcquire(p, node) &&//node.waitStatus==SIGNAL, successor need to signal
                parkAndCheckInterrupt())//发生了中断
                throw new InterruptedException();//throw异常
        }
    } finally {
        if (failed)
            cancelAcquire(node);
    }
}
//-----------------------------------------------------------------------------------
/**
 * Acquires in shared uninterruptible mode.  非中断共享模式
 * @param arg the acquire argument
 */
private void doAcquireShared(int arg) {
    final Node node = addWaiter(Node.SHARED);
    boolean failed = true;
    try {
        boolean interrupted = false;
        for (;;) {
            final Node p = node.predecessor();
            if (p == head) {
                int r = tryAcquireShared(arg);
                if (r >= 0) {
                    setHeadAndPropagate(node, r);
                    p.next = null; // help GC
                    if (interrupted)
                        selfInterrupt();
                    failed = false;
                    return;
                }
            }
            if (shouldParkAfterFailedAcquire(p, node) &&
                parkAndCheckInterrupt())
                interrupted = true;
        }
    } finally {
        if (failed)
            cancelAcquire(node);
    }
}
/**
 * Acquires in shared interruptible mode.
 * @param arg the acquire argument
 */
private void doAcquireSharedInterruptibly(int arg)
    throws InterruptedException {
    final Node node = addWaiter(Node.SHARED);
    boolean failed = true;
    try {
        for (;;) {
            final Node p = node.predecessor();
            if (p == head) {
                int r = tryAcquireShared(arg);
                if (r >= 0) {
                    setHeadAndPropagate(node, r);
                    p.next = null; // help GC
                    failed = false;
                    return;
                }
            }
            if (shouldParkAfterFailedAcquire(p, node) &&
                parkAndCheckInterrupt())
                throw new InterruptedException();
        }
    } finally {
        if (failed)
            cancelAcquire(node);
    }
}
```

### Main exported methods
```java
protected boolean tryAcquire(int arg)  { throw new UnsupportedOperationException();}
protected boolean tryRelease(int arg)  { throw new UnsupportedOperationException();}
protected boolean tryAcquireShared(int arg)  { throw new UnsupportedOperationException();}
protected boolean tryReleaseShared(int arg)  { throw new UnsupportedOperationException();}
```

### Acquire
```java
public final void acquire(int arg) {
    if (!tryAcquire(arg) &&
        acquireQueued(addWaiter(Node.EXCLUSIVE), arg))
        selfInterrupt();
}
public final void acquireInterruptibly(int arg) throws InterruptedException {
    if (Thread.interrupted())
        throw new InterruptedException();
    if (!tryAcquire(arg))
        doAcquireInterruptibly(arg);
}
public final void acquireShared(int arg) {
    if (tryAcquireShared(arg) < 0)
        doAcquireShared(arg);
}
public final void acquireSharedInterruptibly(int arg) throws InterruptedException {
    if (Thread.interrupted())
        throw new InterruptedException();
    if (tryAcquireShared(arg) < 0)
        doAcquireSharedInterruptibly(arg);
}
```

### Release
```java
public final boolean release(int arg) {
    if (tryRelease(arg)) {
        Node h = head;
        if (h != null && h.waitStatus != 0)
            unparkSuccessor(h);
        return true;
    }
    return false;
}
public final boolean releaseShared(int arg) {
    if (tryReleaseShared(arg)) {
        doReleaseShared();
        return true;
    }
    return false;
}
```

## ConditionObject
<!-- later -->

## LockSupport
>此处用于唤醒/挂起线程，UNSAFE.park/unpark/parkUntil

```java
/**Basic thread blocking primitives for creating locks and other synchronization classes.*/
public class LockSupport {
    private LockSupport() {} // Cannot be instantiated.
    /**
     * Makes available the permit for the given thread, if it was not already available.
     * If the thread was blocked on {@code park} then it will unblock.
     * Otherwise, its next call to {@code park} is guaranteed not to block.
     * This operation is not guaranteed to have any effect at all if the given thread has not been started.
     *
     * @param thread the thread to unpark, or {@code null}, in which case
     *        this operation has no effect
     */
    public static void unpark(Thread thread) {
        if (thread != null)
            UNSAFE.unpark(thread);
    }
    /**Disables the current thread for thread scheduling purposes unless the permit is available.*/
    public static void park(Object blocker) {
        Thread t = Thread.currentThread();
        setBlocker(t, blocker);
        UNSAFE.park(false, 0L);
        setBlocker(t, null);
    }
}
```

## CLH lock queue
