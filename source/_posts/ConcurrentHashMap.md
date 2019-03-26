---
title: ConcurrentHashMap
date: 2019-03-01 14:02:47
categories:
- Java
- SourceCode
- Map

tags:
- Java
- Map
---
# ConcurrentHashMap
>A hash table supporting full concurrency of retrievals and high expected concurrency for updates.

<!-- more -->
## ConcurrentMap
```java
public interface ConcurrentMap<K, V> extends Map<K, V>
```
A {@link java.util.Map} providing thread safety and atomicity guarantees.<!-- 线程安全，原子性操作 -->   

Memory consistency effects(*内存一致性*): As with other concurrent collections, actions in a thread prior to placing an object into a {@code ConcurrentMap} as a key or value <a href="package-summary.html#MemoryVisibility"><i>happen-before</i></a> actions subsequent to the access or removal of that object from the {@code ConcurrentMap} in another thread.     
<!-- 与其他并发集合一样，在将对象放入{@code ConcurrentMap}作为键或值之前的线程中的操作发生在从另一个线程中的{@code ConcurrentMap}访问或删除该对象之后的操作之前。 -->     
![ConcurrentMap结构](./ConcurrentMap_Structure.jpg "ConcurrentMap结构")      
```java
//newValue = remappingFunction(key, oldValue); if newValue != null -> replace (key, oldValue) with (key, newValue)   
default V compute(K key, BiFunction<? super K, ? super V, ? extends V> remappingFunction) {
    Objects.requireNonNull(remappingFunction);
    V oldValue = get(key);
    for(;;) {
        V newValue = remappingFunction.apply(key, oldValue);
        if (newValue == null) {
            // delete mapping
            if (oldValue != null || containsKey(key)) {
                // something to remove
                if (remove(key, oldValue)) {
                    // removed the old value as expected
                    return null;
                }
                // some other value replaced old value. try again.
                oldValue = get(key);
            } else {
                // nothing to do. Leave things as they were.
                return null;
            }
        } else {
            // add or replace old mapping
            if (oldValue != null) {
                // replace
                if (replace(key, oldValue, newValue)) {
                    // replaced as expected.
                    return newValue;
                }
                // some other value replaced old value. try again.
                oldValue = get(key);
            } else {
                // add (replace if oldValue was null)
                if ((oldValue = putIfAbsent(key, newValue)) == null) {
                    // replaced
                    return newValue;
                }
                // some other value replaced old value. try again.
            }
        }
    }
}
```
some default methods here ... ...

## ConcurrentHashMap
```java
public class ConcurrentHashMap<K,V> extends AbstractMap<K,V>
    implements ConcurrentMap<K,V>, Serializable
```
- All operations are thread-safe,retrieval operations do not entail locking, and there is not any support for locking the entire table in a way that prevents all access. <br/><br/>              
<!-- 所有操作都是线程安全的，检索操作不需要锁定，并且不支持以阻止所有访问的方式锁定整个表。 -->       
This class is fully interoperable with Hashtable in programs that rely on its thread safety but not on its synchronization details.<br/><br/>
<!-- 这个类在依赖线程安全而不依赖同步细节的程序中与Hashtable完全互操作。 -->
Retrievals reflect the results of the most recently completed update operations holding upon their onset.   
<!-- 检索反映了最新完成的更新操作在开始时的结果。-->   
More formally, an update operation for a given key bears a happens-before relation with any (non-null) retrieval for that key reporting the updated value.

- Bear in mind that the results of aggregate status methods including {@code size}, {@code isEmpty}, and {@code containsValue} are typically useful only when a map is not undergoing concurrent updates in other threads.    
Otherwise, the results of these methods reflect transient states that may be adequate for monitoring or estimation purposes, but not for program control.

- Like {@link Hashtable} but unlike {@link HashMap}, this class does <em>not</em> allow {@code null} to be used as a key or value.


Later ... ...  so crazy ... ...
