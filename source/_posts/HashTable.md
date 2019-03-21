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

### Constructors
```java
private transient Entry<?,?>[] table;
public Hashtable(int initialCapacity, float loadFactor) {
   if (initialCapacity < 0)
       throw new IllegalArgumentException("Illegal Capacity: " + initialCapacity);
   if (loadFactor <= 0 || Float.isNaN(loadFactor))
       throw new IllegalArgumentException("Illegal Load: "+loadFactor);

   if (initialCapacity==0)
       initialCapacity = 1;
   this.loadFactor = loadFactor;
   table = new Entry<?,?>[initialCapacity];//just this.
   threshold = (int)Math.min(initialCapacity * loadFactor, MAX_ARRAY_SIZE + 1);//ok
}
```
new Hashtable(1000)  here capacity only 1000     
new HashMap（1000)   threshold--capacity:1024  
### Contains   
```java
// Hashtable bucket collision list entry
private static class Entry<K,V> implements Map.Entry<K,V> {
    final int hash;
    final K key;
    V value;
    Entry<K,V> next;
    protected Entry(int hash, K key, V value, Entry<K,V> next) {
        this.hash = hash;
        this.key =  key;
        this.value = value;
        this.next = next;
    }
}
//the same structure with HashMap.
public synchronized boolean containsKey(Object key) {
    Entry<?,?> tab[] = table;
    int hash = key.hashCode();
    int index = (hash & 0x7FFFFFFF) % tab.length;//index
    for (Entry<?,?> e = tab[index] ; e != null ; e = e.next) {
        if ((e.hash == hash) && e.key.equals(key)) {
            return true;
        }
    }
    return false;
}
```

### get&put
```java
public synchronized V get(Object key) {
   Entry<?,?> tab[] = table;
   int hash = key.hashCode();
   int index = (hash & 0x7FFFFFFF) % tab.length;//the bucket index to store the list with specific value
   for (Entry<?,?> e = tab[index] ; e != null ; e = e.next) {
       if ((e.hash == hash) && e.key.equals(key)) {//find the specific one in the list
           return (V)e.value;
       }
   }
   return null;
}
public synchronized V put(K key, V value) {
    // Make sure the value is not null
    if (value == null) {
        throw new NullPointerException();
    }

    // Makes sure the key is not already in the hashtable.
    Entry<?,?> tab[] = table;
    int hash = key.hashCode();
    int index = (hash & 0x7FFFFFFF) % tab.length;
    @SuppressWarnings("unchecked")
    Entry<K,V> entry = (Entry<K,V>)tab[index];
    for(; entry != null ; entry = entry.next) {
        if ((entry.hash == hash) && entry.key.equals(key)) {
            V old = entry.value;
            entry.value = value;
            return old;
        }
    }

    addEntry(hash, key, value, index);
    return null;
}
private void addEntry(int hash, K key, V value, int index) {
    modCount++;//change structurally.
    Entry<?,?> tab[] = table;
    if (count >= threshold) {
        // Rehash the table if the threshold is exceeded
        rehash();//reconstruct the hashtable.

        tab = table;
        hash = key.hashCode();
        index = (hash & 0x7FFFFFFF) % tab.length;///ok
    }

    // Creates the new entry.
    @SuppressWarnings("unchecked")
    Entry<K,V> e = (Entry<K,V>) tab[index];
    tab[index] = new Entry<>(hash, key, value, e);
    count++;
}
```

### rehash
Increases the capacity of and internally reorganizes this hashtable,
in order to accommodate and access its entries more efficiently.  
This method is called automatically when the number of keys in the hashtable
exceeds this hashtable's capacity and load factor.
```java
protected void rehash() {
    int oldCapacity = table.length;
    Entry<?,?>[] oldMap = table;

    // overflow-conscious code
    int newCapacity = (oldCapacity << 1) + 1;//new size: 2X+1
    if (newCapacity - MAX_ARRAY_SIZE > 0) {//may be to large
        if (oldCapacity == MAX_ARRAY_SIZE)
            // Keep running with MAX_ARRAY_SIZE buckets
            return;
        newCapacity = MAX_ARRAY_SIZE;
    }
    Entry<?,?>[] newMap = new Entry<?,?>[newCapacity];//ok...

    modCount++;//change structurally here.
    threshold = (int)Math.min(newCapacity * loadFactor, MAX_ARRAY_SIZE + 1);//just be ok.
    table = newMap;

    for (int i = oldCapacity ; i-- > 0 ;) {
        for (Entry<K,V> old = (Entry<K,V>)oldMap[i] ; old != null ; ) {
            Entry<K,V> e = old;
            old = old.next;

            int index = (e.hash & 0x7FFFFFFF) % newCapacity;//new index here.
            e.next = (Entry<K,V>)newMap[index];
            newMap[index] = e;
        }
    }
}
```

### Serializable(Write/Read)
just like the HashMap does.
```java
private void writeObject(java.io.ObjectOutputStream s) throws IOException {
    Entry<Object, Object> entryStack = null;

    synchronized (this) {
        // Write out the threshold and loadFactor
        s.defaultWriteObject();

        // Write out the length and count of elements
        s.writeInt(table.length);
        s.writeInt(count);

        // Stack copies of the entries in the table
        for (int index = 0; index < table.length; index++) {
            Entry<?,?> entry = table[index];

            while (entry != null) {
                entryStack =
                    new Entry<>(0, entry.key, entry.value, entryStack);
                entry = entry.next;
            }
        }
    }

    // Write out the key/value objects from the stacked entries
    while (entryStack != null) {
        s.writeObject(entryStack.key);
        s.writeObject(entryStack.value);
        entryStack = entryStack.next;
    }
}
```

... ...
