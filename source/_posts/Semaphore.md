---
title: Semaphore
date: 2019-07-05 15:25:18
categories:
- Java
- SourceCode
- Semaphore
tags:
- Java
- Semaphore
---
# Semaphore
> A counting semaphore.Conceptually, a semaphore maintains a set of permits.

- Each {@link #acquire} blocks if necessary until a permit is available, and then takes it.
- Each {@link #release} adds a permit, potentially releasing a blocking acquirer.
<!-- more -->

## Example
Semaphores are often used to *restrict the number of threads than can access some (physical or logical) resource*.

```java
class Pool {
  private static final int MAX_AVAILABLE = 100;
  private final Semaphore available = new Semaphore(MAX_AVAILABLE, true);

  public Object getItem() throws InterruptedException {
    available.acquire();
    return getNextAvailableItem();
  }

  public void putItem(Object x) {
    if (markAsUnused(x))
      available.release();
  }

  // Not a particularly efficient data structure; just for demo

  protected Object[] items = new Object[]{};//... whatever kinds of items being managed.存放ok
  protected boolean[] used = new boolean[MAX_AVAILABLE];

  protected synchronized Object getNextAvailableItem() {
    for (int i = 0; i < MAX_AVAILABLE; ++i) {
      if (!used[i]) {
         used[i] = true;
         return items[i];
      }
    }
    return null; // not reached
  }

  protected synchronized boolean markAsUnused(Object item) {
    for (int i = 0; i < MAX_AVAILABLE; ++i) {
      if (item == items[i]) {
         if (used[i]) {
           used[i] = false;
           return true;
         } else
           return false;
      }
    }
    return false;
  }
}
```
Before obtaining an item **each thread must acquire a permit from the semaphore**, guaranteeing that ***an item is available for use***.

When the thread has finished with the item it is returned back to the pool and a permit is returned to the semaphore, allowing another thread to acquire that item.

**Note that no synchronization lock is held when {@link #acquire} is called** ***as that would prevent an item from being returned to the pool***.
<!-- 没有加同步锁🔒，否则的话将会阻止其他线程对pool进行操作{release} -->

The semaphore encapsulates the synchronization needed to restrict access to the pool, separately from any synchronization needed to maintain the consistency of the pool itself.

## Code

```java
public class Semaphore implements java.io.Serializable {
    /** All mechanics via AbstractQueuedSynchronizer subclass */
    private final Sync sync;
    /**
     * Synchronization implementation for semaphore.
     * Uses AQS state to represent permits.
     * Subclassed into fair and nonfair versions.
     */
    abstract static class Sync extends AbstractQueuedSynchronizer {
        Sync(int permits) {
            setState(permits);
        }

        final int getPermits() {
            return getState();
        }

        final int nonfairTryAcquireShared(int acquires) {
            for (;;) {
                int available = getState();
                int remaining = available - acquires;
                if (remaining < 0 ||//state means resource. remaining<0 will wait and enq AQS队列
                    compareAndSetState(available, remaining))//获取锁
                    return remaining;
            }
        }

        protected final boolean tryReleaseShared(int releases) {
            for (;;) {
                int current = getState();
                int next = current + releases;//release the resource
                if (next < current) // overflow  有符号数越界
                    throw new Error("Maximum permit count exceeded");
                if (compareAndSetState(current, next))
                    return true;
            }
        }

        ///////////////////////////////////////////////////////////////////////
        final void reducePermits(int reductions) {
            for (;;) {
                int current = getState();
                int next = current - reductions;
                if (next > current) // underflow
                    throw new Error("Permit count underflow");
                if (compareAndSetState(current, next))
                    return;
            }
        }
        final int drainPermits() {
            for (;;) {
                int current = getState();
                if (current == 0 || compareAndSetState(current, 0))
                    return current;
            }
        }
    }

    static final class NonfairSync extends Sync {
        NonfairSync(int permits) {
            super(permits);
        }
        protected int tryAcquireShared(int acquires) {//nofair....
            return nonfairTryAcquireShared(acquires);
        }
    }
    static final class FairSync extends Sync {
        FairSync(int permits) {
            super(permits);
        }

        protected int tryAcquireShared(int acquires) {
            for (;;) {
                if (hasQueuedPredecessors())//存在前驱节点
                    return -1;
                int available = getState();
                int remaining = available - acquires;
                if (remaining < 0 ||//remaining<0 等待
                    compareAndSetState(available, remaining))
                    return remaining;
            }
        }
    }

    ////////////////////////constructors////////////////
    public Semaphore(int permits) {
        sync = new NonfairSync(permits);
    }
    public Semaphore(int permits, boolean fair) {
        sync = fair ? new FairSync(permits) : new NonfairSync(permits);
    }
    ////////////////acquire
    public void acquire() throws InterruptedException {
        sync.acquireSharedInterruptibly(1);
    }
    public void acquire(int permits) throws InterruptedException {
      if (permits < 0) throw new IllegalArgumentException();
      sync.acquireSharedInterruptibly(permits);
    }
    public void acquireUninterruptibly() {
        sync.acquireShared(1);
    }
    public boolean tryAcquire() {
        return sync.nonfairTryAcquireShared(1) >= 0;//remaining>0
    }
    public boolean tryAcquire(int permits) {
        if (permits < 0) throw new IllegalArgumentException();
        return sync.nonfairTryAcquireShared(permits) >= 0;
    }
    //////////////////release
    public void release() {
        sync.releaseShared(1);
    }

    /**
    * Acquires and returns all permits that are immediately available.
    * @return the number of permits acquired
    */
   public int drainPermits() {
       return sync.drainPermits();
   }

   /**
    * Shrinks the number of available permits by the indicated reduction.
    * This method can be useful in subclasses that use semaphores to track resources that become unavailable.
    * This method differs from {@code acquire} in that it does not block waiting for permits to become available.
    * @param reduction the number of permits to remove
    * @throws IllegalArgumentException if {@code reduction} is negative
    */
   protected void reducePermits(int reduction) {
       if (reduction < 0) throw new IllegalArgumentException();
       sync.reducePermits(reduction);
   }
}
```
