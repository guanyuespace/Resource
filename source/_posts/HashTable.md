---
title: Hashtable
date: 2019-03-20 13:44:27
categories:
- Java
- SourceCode
- Hashtable
tags:  
- Java
- Hashtable
---
# Hashtable
```java
public class Hashtable<K,V>
    extends Dictionary<K,V>
    implements Map<K,V>, Cloneable, java.io.Serializable
```
- This class implements a hash table, which maps keys to values. Any non-<code>null</code> object can be used as a key or as a value.       
- <em>fail-fast</em>: if the Hashtable is structurally modified at any time after the iterator is created, in any way except through the iterator's own <tt>remove</tt> method, the iterator will throw a {@link ConcurrentModificationException}.    
The Enumerations returned by Hashtables keys and elements methods are <em>not</em> fail-fast.    

Unlike the new collection implementations, {@code Hashtable} is synchronized.       
- If a thread-safe implementation is not needed, it is recommended to use {@link HashMap} in place of {@code Hashtable}.     
- If a thread-safe highly-concurrent implementation is desired, then it is recommended to use {@link java.util.concurrent.ConcurrentHashMap} in place of {@code Hashtable}.   

<!-- more -->

## Dictionary
```java
public abstract class Dictionary<K,V>
```
The <code>Dictionary</code> class is the abstract parent of any class, such as <code>Hashtable</code>, which maps keys to values.   
Any non-<code>null</code> object can be used as a key and as a value.    
<strong>NOTE: This class is obsolete.  New implementations should implement the Map interface, rather than extending this class.</strong>
```java
public abstract class Dictionary<K, V> {
    public Dictionary() {
    }

    public abstract int size();

    public abstract boolean isEmpty();

    public abstract Enumeration<K> keys();

    public abstract Enumeration<V> elements();

    public abstract V get(Object var1);

    public abstract V put(K var1, V var2);

    public abstract V remove(Object var1);
}
```

## Hashtable   
An instance of <code>Hashtable</code> has two parameters that affect its performance: <i>initial capacity</i> and <i>load factor</i>.  
- The <i>capacity</i> is the number of <i>buckets</i> in the hash table, and the <i>initial capacity</i> is simply the capacity at the time the hash table is created.  
Note that the hash table is <i>open</i>: in the case of a "hash collision", a single bucket stores multiple entries, which must be searched sequentially.
- The <i>load factor</i> is a measure of how full the hash table is allowed to get before its capacity is automatically increased.

**If many entries are to be made into a Hashtable, creating it with a sufficiently large capacity may allow the entries to be inserted more efficiently than letting it perform automatic rehashing as needed to grow the table.**      
