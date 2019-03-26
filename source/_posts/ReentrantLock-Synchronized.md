---
title: ReentrantLock-Synchronized
date: 2019-03-04 15:40:27
categories:
tags:
---
# ReentrantLock
>A reentrant mutual exclusion {@link Lock} with the same basic behavior and semantics as the implicit monitor lock accessed using {@code synchronized} methods and statements, but with extended capabilities.         

```java
public class ReentrantLock implements Lock, java.io.Serializable
```
<!-- more -->

## Lock
>`Lock` implementations provide more extensive locking operations than can be obtained using `synchronized` methods and statements.  
They allow more flexible structuring, may have quite different properties, and may support multiple associated `Condition` objects.


```java
public interface Lock
```

## ReentrantLock
- The constructor for this class accepts an optional <em>fairness</em> parameter.  When set {@code true}, under contention, locks favor granting access to the longest-waiting thread.Otherwise this lock does not guarantee any particular access order.      
- **Programs using fair locks accessed by many threads may display lower overall throughput (*i.e., are slower; often much slower*) than those using the default setting, but have smaller variances in times to obtain locks and guarantee lack of starvation.**  
- Note however, that fairness of locks does not guarantee fairness of thread scheduling. Thus, one of many threads using a fair lock may obtain it multiple times in succession while other active threads are not progressing and not currently holding the lock.        
- Also note that the untimed {@link #tryLock()} method does not honor the fairness setting. It will succeed if the lock is available even if other threads are waiting.

### Constructor
```java
public ReentrantLock() {
    sync = new NonfairSync();
}
public ReentrantLock(boolean fair) {
    sync = fair ? new FairSync() : new NonfairSync();
}
```


## Sync

### NonfairSync
```java
static final class NonfairSync extends Sync
```
### FairSync
```java
static final class FairSync extends Sync
```


### Abstract
```java
abstract static class Sync extends AbstractQueuedSynchronizer
```

### AbstractQueuedSynchronizer
```java
public abstract class AbstractQueuedSynchronizer
    extends AbstractOwnableSynchronizer
    implements java.io.Serializable
```
### AbstractOwnableSynchronizer
```java
public abstract class AbstractOwnableSynchronizer
    implements java.io.Serializable
```



Later ........
