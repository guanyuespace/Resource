---
title: HashMap
date: 2019-03-01 14:02:27
categories:
- Java
- SourceCode
- HashMap
tags:  
- Java
- HashMap
---
# HashMap
```java
public class HashMap<K,V> extends AbstractMap<K,V>
    implements Map<K,V>, Cloneable, Serializable
```
Hash table based implementation of the <tt>Map</tt> interface.     
This implementation provides all of the optional map operations, and permits <tt>null</tt> values and the <tt>null</tt> key.     
(The <tt>HashMap</tt> class is roughly equivalent to <tt>Hashtable</tt>, except that it is unsynchronized and permits nulls.)     
This class makes no guarantees as to the order of the map; in particular, it does not guarantee that the order will remain constant over time.

<!-- more -->
# AbstractMap
```java
public abstract class AbstractMap<K,V> implements Map<K,V>
```
- To implement an unmodifiable map, the programmer needs only to extend this class and provide an implementation for the <tt>entrySet</tt> method, which returns a set-view of the map's mappings.  Typically, the returned set will, in turn, be implemented atop <tt>AbstractSet</tt>.  This set should not support the <tt>add</tt> or <tt>remove</tt> methods, and its iterator should not support the <tt>remove</tt> method.（实现不可变的Map只需实现entrySet方法，并且返回Set和其Iterator不能实现add与remove方法）

- To implement a modifiable map, the programmer must additionally override this class's <tt>put</tt> method (which otherwise throws an <tt>UnsupportedOperationException</tt>), and the iterator returned by <tt>entrySet().iterator()</tt> must additionally implement its <tt>remove</tt> method.
> unmodifiable: `entrySet()`   
> modifiable: `entrySet()`, `put()`, `entrySet().iterator()`, `iterator.remove()`   

## 代码节选
```java
public abstract class AbstractMap<K,V> implements Map<K,V> {
    protected AbstractMap() {
    }
    //entrySet().iterator()  :Iterator  遍历
    public int size() {
        return entrySet().size();
    }
    //没有给出实现
    public abstract Set<Entry<K,V>> entrySet();

    public boolean containsKey(Object key) {
        Iterator<Map.Entry<K,V>> i = entrySet().iterator();
        if (key==null) {
            while (i.hasNext()) {
                Entry<K,V> e = i.next();
                if (e.getKey()==null)//可以包含null作为key
                    return true;
            }
        } else {
            while (i.hasNext()) {
                Entry<K,V> e = i.next();
                if (key.equals(e.getKey()))
                    return true;
            }
        }
        return false;
    }

    //remove element by iterator
    public V remove(Object key) {
        Iterator<Entry<K,V>> i = entrySet().iterator();
        Entry<K,V> correctEntry = null;
        if (key==null) {//remove key:null
            while (correctEntry==null && i.hasNext()) {
                Entry<K,V> e = i.next();
                if (e.getKey()==null)
                    correctEntry = e;
            }
        } else {//
            while (correctEntry==null && i.hasNext()) {
                Entry<K,V> e = i.next();
                if (key.equals(e.getKey()))
                    correctEntry = e;
            }
        }
        //find the Key: correctEntry.key
        V oldValue = null;
        if (correctEntry !=null) {
            oldValue = correctEntry.getValue();
            i.remove();//implementation: remove  ...
        }
        return oldValue;
    }
    //Entry：Map项
    public static class SimpleEntry<K,V>
        implements Entry<K,V>, java.io.Serializable{
          public V setValue(V value) {
              V oldValue = this.value;
              this.value = value;
              return oldValue;
          }
    }
    //Immutable： 不可变的
    public static class SimpleImmutableEntry<K,V>
        implements Entry<K,V>, java.io.Serializable{
          public V setValue(V value) {
              throw new UnsupportedOperationException();
          }
    }

}
```
实现交由entrySet.Iterator完成，而entrySet没有给出实现，需要子类提供实现
# HashMap
> `initial capacity` and `load factor`   
> multi-threads modify the `map structure`  -->  synchronized

This implementation provides constant-time performance for the basic operations (<tt>get</tt> and <tt>put</tt>), assuming the hash function disperses the elements properly among the buckets.(在key均匀分布时才有查询时间O(n))     
Iteration over collection views requires time proportional to the "capacity" of the <tt>HashMap</tt> instance (the number of buckets) plus its size (the number of key-value mappings).(集合迭代需要与capacity等比例的时间);Thus, it's very important **not to set the initial capacity too high (or the load factor too low) ** if iteration performance is important.     

- An instance of <tt>HashMap</tt> has two parameters that affect its performance: <b>initial capacity</b> and <b>load factor</b>.  

 - The <b>capacity</b> is the number of buckets in the hash table, and the initial capacity is simply the capacity at the time the hash table is created.     
 - The <b>load factor</b> is a measure of how full the hash table is allowed to get before its capacity is automatically increased.     

   **When the number of entries in the hash table exceeds the product of the load factor and the current capacity, the hash table is <i>rehashed</i> (that is, internal data structures are rebuilt) so that the hash table has approximately twice the number of buckets.**  

- **If many mappings are to be stored in a <tt>HashMap</tt> instance, creating it with a sufficiently large capacity will allow the mappings to be stored more efficiently than letting it perform automatic rehashing as needed to grow the table.**        
 - Note that using many keys with the same {@code hashCode()} is a sure way to slow down performance of any hash table.
 - To ameliorate impact, when keys are {@link Comparable}, this class may use comparison order among keys to help break ties.

#### modified structurally(Exception)
```java
If no such object exists, the map should be "wrapped" using the {@link Collections#synchronizedMap Collections.synchronizedMap} method.  

This is best done at creation time, to prevent accidental unsynchronized access to the map:

Map m = Collections.synchronizedMap(new HashMap(...));
```
## code
### Abstract(简介)
```java
public class HashMap<K,V> extends AbstractMap<K,V>
    implements Map<K,V>, Cloneable, Serializable {
      static final int DEFAULT_INITIAL_CAPACITY = 1 << 4; // aka 16
      static final float DEFAULT_LOAD_FACTOR = 0.75f;
      final float loadFactor;
      // The next size value at which to resize (capacity * load factor).
      // (The javadoc description is true upon serialization.
      // Additionally, if the table array has not been allocated, this
      // field holds the initial array capacity, or zero signifying
      // DEFAULT_INITIAL_CAPACITY.)
      int threshold;//阈值
      transient int modCount;//---expectedModCount  :   Modification Exception
      transient Node<K,V>[] table;//store values ---> table[]  index: hashCode&capacity-1
      public HashMap(int initialCapacity, float loadFactor) {
          if (initialCapacity < 0)
              throw new IllegalArgumentException("Illegal initial capacity: " +
                                                 initialCapacity);
          if (initialCapacity > MAXIMUM_CAPACITY)
              initialCapacity = MAXIMUM_CAPACITY;
          if (loadFactor <= 0 || Float.isNaN(loadFactor))
              throw new IllegalArgumentException("Illegal load factor: " +
                                                 loadFactor);
          this.loadFactor = loadFactor;
          this.threshold = tableSizeFor(initialCapacity);
      }
      //找到大于initialCapacity最小的2^n
      //tableSizeFor(100)=128;  tableSizeFor(65536)=65536; tableSizeFor(65537)=131072   
      static final int tableSizeFor(int cap) {
        int n = cap - 1;//最低位1右移一位
        n |= n >>> 1;//无符号右移，高位补0   +0.5     （|= 类比 加）
        n |= n >>> 2;//                    +0.25
        n |= n >>> 4;//                    +0.125
        n |= n >>> 8;//                    +0.0625
        n |= n >>> 16;//                   +0.03125
        return (n < 0) ? 1 : (n >= MAXIMUM_CAPACITY) ? MAXIMUM_CAPACITY : n + 1;
    }//运算结果： ？？ 1.75*cap
    //int： 32位   （右移1,2,4,8,16使得所有位均为1，later+1 ok-->最小的2^n）
}
```
### Node-Iterator(entrySet method)
```java
static class Node<K,V> implements Map.Entry<K,V> {
    final int hash;
    final K key;
    V value;
    Node<K,V> next;
    Node(int hash, K key, V value, Node<K,V> next) {
        this.hash = hash;// hash：key.hashCode
        this.key = key;
        this.value = value;
        this.next = next;//hash碰撞链表
    }
}

// Tree bins (后续分析。。。)
/**
 * Entry for Tree bins. Extends LinkedHashMap.Entry (which in turn
 * extends Node) so can be used as extension of either regular or
 * linked node.
 */
static final class TreeNode<K,V> extends LinkedHashMap.Entry<K,V> {
    TreeNode<K,V> parent;  // red-black tree links
    TreeNode<K,V> left;
    TreeNode<K,V> right;
    TreeNode<K,V> prev;    // needed to unlink next upon deletion
    boolean red;
    TreeNode(int hash, K key, V val, Node<K,V> next) {
        super(hash, key, val, next);
    }
    // Returns root of tree containing this node.
    final TreeNode<K,V> root() {
        for (TreeNode<K,V> r = this, p;;) {
            if ((p = r.parent) == null)
                return r;
            r = p;
        }
    }
    .....
}
///////HashMap#entrySet方法/////////////
public Set<Map.Entry<K,V>> entrySet() {
    Set<Map.Entry<K,V>> es;
    return (es = entrySet) == null ? (entrySet = new EntrySet()) : es;
}
abstract class HashIterator {
    Node<K,V> next;        // next entry to return
    Node<K,V> current;     // current entry
    int expectedModCount;  // for fast-fail
    int index;             // current slot
    .......
    final Node<K,V> nextNode() {
        Node<K,V>[] t;
        Node<K,V> e = next;
        if (modCount != expectedModCount)
            throw new ConcurrentModificationException();
        if (e == null)
            throw new NoSuchElementException();
        if ((next = (current = e).next) == null && (t = table) != null) {
            do {} while (index < t.length && (next = t[index++]) == null);//table中查询下一个bucket
        }
        return e;//next Node: table[nextHash]
    }
}
final class EntryIterator extends HashIterator
    implements Iterator<Map.Entry<K,V>> {
    public final Map.Entry<K,V> next() { return nextNode(); }
}
final class EntrySet extends AbstractSet<Map.Entry<K,V>> {
    public final int size()                 { return size; }
    public final void clear()               { HashMap.this.clear(); }
    public final Iterator<Map.Entry<K,V>> iterator() {
        return new EntryIterator();
    }
    //remove Entry:o
    public final boolean remove(Object o) {
        if (o instanceof Map.Entry) {
            Map.Entry<?,?> e = (Map.Entry<?,?>) o;
            Object key = e.getKey();
            Object value = e.getValue();
            return removeNode(hash(key), key, value, true, true) != null;
        }
        return false;
    }
}
```
数据存储在table中 index: hash&(capacity-1) ; modify structurally: pay attention to the expectedModCount & modCount    
***LinkedHashMap 与 TreeNode相关后续介绍***     
在EntrySet中仅有remove方法 --> HashMap#removeNode


### Put--Remove
>put()与remove()方法具体实现交由putVal()和removeNode()

```java
//初始化&空间不够时，重建table[扩容]
final Node<K,V>[] resize() {
    Node<K,V>[] oldTab = table;
    int oldCap = (oldTab == null) ? 0 : oldTab.length;
    int oldThr = threshold;
    int newCap, newThr = 0;
    if (oldCap > 0) {//已经存有数据
        if (oldCap >= MAXIMUM_CAPACITY) {//不能更大了
            threshold = Integer.MAX_VALUE;
            return oldTab;
        }
        else if ((newCap = oldCap << 1) < MAXIMUM_CAPACITY &&
                 oldCap >= DEFAULT_INITIAL_CAPACITY)//升级空间
            newThr = oldThr << 1; // double threshold
    }
    else if (oldThr > 0) // initial capacity was placed in threshold
        newCap = oldThr;
    else {               // zero initial threshold signifies using defaults
        newCap = DEFAULT_INITIAL_CAPACITY;//初始空间 10
        newThr = (int)(DEFAULT_LOAD_FACTOR * DEFAULT_INITIAL_CAPACITY);//10*0.75
    }
    if (newThr == 0) {// oldThr>0
        float ft = (float)newCap * loadFactor;
        newThr = (newCap < MAXIMUM_CAPACITY && ft < (float)MAXIMUM_CAPACITY ?
                  (int)ft : Integer.MAX_VALUE);
    }
    threshold = newThr;//阈值更新
    @SuppressWarnings({"rawtypes","unchecked"})
    Node<K,V>[] newTab = (Node<K,V>[])new Node[newCap];//新建table
    table = newTab;
    if (oldTab != null) {//旧值拷贝（new index=hash * newCapacity）
        for (int j = 0; j < oldCap; ++j) {
            Node<K,V> e;
            if ((e = oldTab[j]) != null) {//旧值e
                oldTab[j] = null;
                if (e.next == null)//bucket：e --- next链表中不存在其他元素
                    newTab[e.hash & (newCap - 1)] = e;
                else if (e instanceof TreeNode)//bucket: e --- TreeNode.......
                    ((TreeNode<K,V>)e).split(this, newTab, j, oldCap);
                else {// bucket: e --- next链表...   
                      // 进行链表复制
						          // 方法比较特殊： 它并没有重新计算元素在数组中的位置
						          // 而是采用了 原始位置:j加原数组长度:oldCap的方法计算得到位置
                    Node<K,V> loHead = null, loTail = null;//位置不变
                    Node<K,V> hiHead = null, hiTail = null;//位置改变
                    Node<K,V> next;
                    do {//构建链表存储(同一hash值)
                        /*********************************************/
                        /**
                        * 注: e本身就是一个链表的节点，它有自身的值和next(链表的值)，但是因为next值对节点扩容没有帮助，
                        * 所有在下面讨论中，我近似认为 e是一个只有自身值，而没有next值的元素。
                        */
                        /*********************************************/
                        next = e.next;
                        // 注意：不是(e.hash & (oldCap-1));而是(e.hash & oldCap)

          							// (e.hash & oldCap) 得到的是 元素的在数组中的位置是否需要移动,示例如下

          							// 示例1：
          							// e.hash=10 0000 1010
          							// oldCap=16 0001 0000
          							//	 &   =0	 0000 0000       比较高位的第一位 0
          							//结论：元素位置在扩容后数组中的位置没有发生改变
          							// 示例2：
          							// e.hash=17 0001 0001
          							// oldCap=16 0001 0000
          							//	 &   =1	 0001 0000      比较高位的第一位   1
          							//结论：元素位置在扩容后数组中的位置发生了改变，新的下标位置是原下标位置+原数组长度

                        // (e.hash & (oldCap-1))旧位置（old Index）
                      	// (e.hash & (oldCap-1)) 得到的是下标位置,示例如下
          							//   e.hash=10 0000 1010
          							// oldCap-1=15 0000 1111
          							//      &  =10 0000 1010
          							//   e.hash=17 0001 0001
          							// oldCap-1=15 0000 1111
          							//      &  =1  0000 0001

          							//新下标位置
          							//   e.hash=17 0001 0001
          							// newCap-1=31 0001 1111    newCap=32
          							//      &  =17 0001 0001    1+oldCap = 1+16

          							//元素在重新计算hash之后，因为n变为2倍，那么n-1的mask范围在高位多1bit(红色)，因此新的index就会发生这样的变化：
          							//参考博文：[Java8的HashMap详解](https://blog.csdn.net/login_sonata/article/details/76598675)  
          							// 0000 0001->0001 0001
                        if ((e.hash & oldCap) == 0) {//(e.hash & oldCap)==0  不需要改变位置
                            if (loTail == null)
                                loHead = e;
                            else
                                loTail.next = e;
                            loTail = e;
                        }else {//需要改变位置  。。。   
                            if (hiTail == null)
                                hiHead = e;
                            else
                                hiTail.next = e;
                            hiTail = e;
                        }
                    } while ((e = next) != null);
                    if (loTail != null) {
                        loTail.next = null;
                        newTab[j] = loHead;//            j
                    }
                    if (hiTail != null) {
                        hiTail.next = null;
                        newTab[j + oldCap] = hiHead;//  j+oldCap
                    }
                }
            }
        }
    }
    return newTab;
}
//put(Key,Value)
final V putVal(int hash, K key, V value, boolean onlyIfAbsent,
               boolean evict) {
    Node<K,V>[] tab; Node<K,V> p; int n, i;
    if ((tab = table) == null || (n = tab.length) == 0)//尚未初始化...
        n = (tab = resize()).length;//
    if ((p = tab[i = (n - 1) & hash]) == null)//当前index尚未插入相应值
        tab[i] = newNode(hash, key, value, null);//插入到index:n-1&hash处
    else {//当前节点 tab[(n-1)&hash]  not null --- 1.重复（更新Value） 2.Hash碰撞,插入
        Node<K,V> e; K k;
        if (p.hash == hash &&
            ((k = p.key) == key || (key != null && key.equals(k))))//Node: e-current bucket head(just have inserted ...)
            e = p;//重复
        else if (p instanceof TreeNode)//TreeNode
            e = ((TreeNode<K,V>)p).putTreeVal(this, tab, hash, key, value);
        else {//tab[(n-1)&hash]  != 当前。。。   hash碰撞...
            for (int binCount = 0; ; ++binCount) {//
                if ((e = p.next) == null) {//insert tail ...
                    p.next = newNode(hash, key, value, null);
                    if (binCount >= TREEIFY_THRESHOLD - 1) // -1 for 1st
                        treeifyBin(tab, hash);//...
                    break;
                }
                if (e.hash == hash &&
                    ((k = e.key) == key || (key != null && key.equals(k))))
                    break;
                p = e;
            }
        }
        //相同key更新value
        if (e != null) { // existing mapping for key
            V oldValue = e.value;
            if (!onlyIfAbsent || oldValue == null)
                e.value = value;
            afterNodeAccess(e);
            return oldValue;
        }
    }
    ++modCount;//modCount & expectedModCount
    if (++size > threshold)
        resize();//重构,扩容
    afterNodeInsertion(evict);//LinkedHashMap-callback  
    return null;
}
//remove Node
final Node<K,V> removeNode(int hash, Object key, Object value,
                           boolean matchValue, boolean movable) {
    Node<K,V>[] tab; Node<K,V> p; int n, index;
    if ((tab = table) != null && (n = tab.length) > 0 &&
        (p = tab[index = (n - 1) & hash]) != null) {// index: (n-1)&hash
        Node<K,V> node = null, e; K k; V v;
        if (p.hash == hash &&
            ((k = p.key) == key || (key != null && key.equals(k))))
            node = p;//find the node to delete
        else if ((e = p.next) != null) {//find in 链表
            if (p instanceof TreeNode)//...
                node = ((TreeNode<K,V>)p).getTreeNode(hash, key);
            else {//链表中。。。
                do {
                    if (e.hash == hash &&
                        ((k = e.key) == key ||
                         (key != null && key.equals(k)))) {
                        node = e;//find it .
                        break;
                    }
                    p = e;
                } while ((e = e.next) != null);
            }
        }
        if (node != null && (!matchValue || (v = node.value) == value ||
                             (value != null && value.equals(v)))) {//node节点删除
            if (node instanceof TreeNode)//...
                ((TreeNode<K,V>)node).removeTreeNode(this, tab, movable);
            else if (node == p)
                tab[index] = node.next;//ok
            else
                p.next = node.next;//ok
            ++modCount;
            --size;
            afterNodeRemoval(node);
            return node;
        }
    }
    return null;
}
```
关于扩容，链表元素：[深入解析HashMap原理（基于JDK1.8）](https://blog.csdn.net/qq_37113604/article/details/81353626)     
避免不必要的扩容:     
假设要存储1000个     
设1024 but 1024*0.75&lt;1000  : so  set  2048   
....

## later

about TreeNode ... ...
