---
title: ReentrantLock
date: 2019-07-05 15:24:20
categories:
- Java
- SourceCode
- ReentrantLock
tags:
- Java
- ReentrantLock
---

# ReentrantLock
>可重入锁，独占模式exclusive,Fair/UnFair

A `reentrant` mutual exclusion Lock with *the same basic behavior and semantics* as the implicit monitor lock accessed using `synchronized` methods and statements, but with extended capabilities.

<!-- more -->

## Code

```java
public class ReentrantLock implements Lock, java.io.Serializable {
    /** Synchronizer providing all implementation mechanics */
    private final Sync sync;//which extends the AQS.
    abstract static class Sync extends AbstractQueuedSynchronizer {//AQS实现，主要🔒逻辑
    }

    // Sync object for non-fair locks 非公平锁
    static final class NonfairSync extends Sync {

        // Performs lock.  Try immediate barge, backing up to normal acquire on failure.
        final void lock() {
            if (compareAndSetState(0, 1))//barge:冲撞，尝试获取锁🔒
                setExclusiveOwnerThread(Thread.currentThread());//成功ok
            else
                acquire(1);//tryAcquire()==>加入AQS队列
        }

        protected final boolean tryAcquire(int acquires) {
            return nonfairTryAcquire(acquires);
        }
    }

    // Sync object for fair locks
    static final class FairSync extends Sync {
        private static final long serialVersionUID = -3000897897090466540L;

        final void lock() {
            acquire(1);//tryAcquire()==>加入AQS队列
        }

        // Fair version of tryAcquire.  Don't grant access unless recursive call or no waiters or is first.
        protected final boolean tryAcquire(int acquires) {
            final Thread current = Thread.currentThread();
            int c = getState();
            if (c == 0) {//当前🔒空闲
                if (!hasQueuedPredecessors() &&//没有前驱节点
                    compareAndSetState(0, acquires)) {//尝试获取锁
                    setExclusiveOwnerThread(current);//ok
                    return true;
                }
            }
            else if (current == getExclusiveOwnerThread()) {//重入锁
                int nextc = c + acquires;//state++
                if (nextc < 0)
                    throw new Error("Maximum lock count exceeded");//32位有符号正数
                setState(nextc);
                return true;
            }
            return false;
        }
    }
    public ReentrantLock() { sync = new NonfairSync();}
    public ReentrantLock(boolean fair) { sync = fair ? new FairSync() : new NonfairSync();  }

    // methods: whose implementation is based on the AbstractQueuedSynchronizer$Sync.
    public void lock() { sync.lock();}
    //Acquires the lock unless the current thread is interrupted.
    public void lockInterruptibly() throws InterruptedException {
        sync.acquireInterruptibly(1);
    }

    //Acquires the lock only if it is not held by another thread at the time of invocation.
    public boolean tryLock() {
        return sync.nonfairTryAcquire(1);
    }
    public void unlock() { sync.release(1); }

    //conditions
    public Condition newCondition() {
        return sync.newCondition();
    }
    public boolean hasWaiters(Condition condition) {
        if (condition == null)
            throw new NullPointerException();
        if (!(condition instanceof AbstractQueuedSynchronizer.ConditionObject))
            throw new IllegalArgumentException("not owner");
        return sync.hasWaiters((AbstractQueuedSynchronizer.ConditionObject)condition);
    }
    public int getWaitQueueLength(Condition condition) {
        if (condition == null)
            throw new NullPointerException();
        if (!(condition instanceof AbstractQueuedSynchronizer.ConditionObject))
            throw new IllegalArgumentException("not owner");
        return sync.getWaitQueueLength((AbstractQueuedSynchronizer.ConditionObject)condition);
    }
}
```


## Sync

```java
/**
 * Base of synchronization control for this lock.
 * Subclassed into fair and nonfair versions below. Uses AQS state to represent the number of holds on the lock.
 */
abstract static class Sync extends AbstractQueuedSynchronizer {

    abstract void lock();//implementation in Subclasses(Fair/NonfairSync)

    /**
     * Performs non-fair tryLock.
     * tryAcquire is implemented in subclasses, but both need nonfair try for trylock method.
     */
    final boolean nonfairTryAcquire(int acquires) {
        final Thread current = Thread.currentThread();
        int c = getState();
        if (c == 0) {//ok, 无锁直接尝试
            if (compareAndSetState(0, acquires)) {
                setExclusiveOwnerThread(current);
                return true;
            }
        }
        else if (current == getExclusiveOwnerThread()) {//重入
            int nextc = c + acquires;
            if (nextc < 0) // overflow
                throw new Error("Maximum lock count exceeded");
            setState(nextc);
            return true;
        }
        return false;
    }
    protected final boolean tryRelease(int releases) {
        int c = getState() - releases;
        if (Thread.currentThread() != getExclusiveOwnerThread())
            throw new IllegalMonitorStateException();
        boolean free = false;
        if (c == 0) {//仅当🔒完全能释放 return ture.
            free = true;
            setExclusiveOwnerThread(null);
        }
        setState(c);
        return free;
    }
    final int getHoldCount() {
        return isHeldExclusively() ? getState() : 0;
    }

    final boolean isLocked() {
        return getState() != 0;
    }
}
```
