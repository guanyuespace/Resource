---
title: ConcurrentHashMap代码分析
date: 2019-05-23 08:40:46
categories:
- Java
- SourceCode
- Map

tags:
- Java
- Map
---
# ConcurrentHashMap
>源码分析

```java
public class ConcurrentHashMap<K,V> extends AbstractMap<K,V>
    implements ConcurrentMap<K,V>, Serializable {...}
```
<!-- more -->

## Abstract
>概要

```java
public class ConcurrentHashMap<K,V> extends AbstractMap<K,V>
    implements ConcurrentMap<K,V>, Serializable {

    transient volatile Node<K,V>[] table;//数据存储... ...
    private transient volatile Node<K,V>[] nextTable;//
    private transient volatile int sizeCtl;///Table initialization and resizing control.


    /////////////////////////////record the size of ConcurrentHashMap.Node
    private transient volatile long baseCount;
    private transient volatile CounterCell[] counterCells;//get the sum of this array value, for what? baseCount+counterCells.value


    //////////Constructors//////////
    public ConcurrentHashMap() {//DEFAULT_CAPACITY:16}
    public ConcurrentHashMap(int initialCapacity) {
        if (initialCapacity < 0)
            throw new IllegalArgumentException();
        int cap = ((initialCapacity >= (MAXIMUM_CAPACITY >>> 1)) ?
                   MAXIMUM_CAPACITY :
                   tableSizeFor(initialCapacity + (initialCapacity >>> 1) + 1));//tableSizeFor(1.5X)   大于1.5X的最小2^n
        this.sizeCtl = cap;
    }
    public ConcurrentHashMap(int initialCapacity, float loadFactor) {
        this(initialCapacity, loadFactor, 1);
    }
    //concurrencyLevel：the estimated number of concurrently updating threads. The implementation may use this value as a sizing hint.
    public ConcurrentHashMap(int initialCapacity, float loadFactor, int concurrencyLevel) {
        if (initialCapacity < concurrencyLevel)   // Use at least as many bins
            initialCapacity = concurrencyLevel;   // as estimated threads. make sure the number of nodes can cover the num of threads
        long size = (long)(1.0 + (long)initialCapacity / loadFactor);
        int cap = (size >= (long)MAXIMUM_CAPACITY) ?//ok
            MAXIMUM_CAPACITY : tableSizeFor((int)size);
        this.sizeCtl = cap;//guess:.... maybe the ...
    }

    //////////important Methods//////////////////
    public int size() {
        long n = sumCount();
        return ((n < 0L) ? 0 :
                (n > (long)Integer.MAX_VALUE) ? Integer.MAX_VALUE :
                (int)n);
    }
    /////so CounterCell记录了{每一段元素集}中NODE的个数
    final long sumCount() {
        CounterCell[] as = counterCells;////记录个数
        CounterCell a;
        long sum = baseCount;////记录个数:baseCount ??
        if (as != null) {
            for (int i = 0; i < as.length; ++i) {
                if ((a = as[i]) != null)
                    sum += a.value;
            }
        }
        return sum;
    }
    public boolean isEmpty() {
        return sumCount() <= 0L; // ignore transient negative values
    }
}
```

## Methods

### Node
```java
static class Node<K,V> implements Map.Entry<K,V> {
        final int hash;
        final K key;
        volatile V val;
        volatile Node<K,V> next;

        Node(int hash, K key, V val, Node<K,V> next) {
            this.hash = hash;
            this.key = key;
            this.val = val;
            this.next = next;
        }

        //to ameliorate impact, which is from the same key.  to avoid collisions
        public final int hashCode()   { return key.hashCode() ^ val.hashCode(); }

        Node<K,V> find(int h, Object k) {//在碰撞链表中查找
            Node<K,V> e = this;
            if (k != null) {
                do {
                    K ek;
                    if (e.hash == h &&
                        ((ek = e.key) == k || (ek != null && k.equals(ek))))
                        return e;
                } while ((e = e.next) != null);
            }
            return null;
        }
}
```


### Utils
```java
//////////////////what's this ? meaning ???
static final int MOVED     = -1; // hash for forwarding nodes
static final int TREEBIN   = -2; // hash for roots of trees
static final int RESERVED  = -3; // hash for transient reservations
static final int HASH_BITS = 0x7fffffff; // usable bits of normal node hash

static final int TREEIFY_THRESHOLD = 8;

//用以替代hashCode,并且确保该值始终为正数
int spread(int h){return (h ^ (h >>> 16)) & HASH_BITS;}//static final int HASH_BITS = 0x7fffffff;

//////////////////Unsafe操作///////////////////////////////
static final <K,V> Node<K,V> tabAt(Node<K,V>[] tab, int i) {
    return (Node<K,V>)U.getObjectVolatile(tab, ((long)i << ASHIFT) + ABASE);//获取数组Node[]中第i个元素NODE
}
static final <K,V> boolean casTabAt(Node<K,V>[] tab, int i, Node<K,V> c, Node<K,V> v) {
    return U.compareAndSwapObject(tab, ((long)i << ASHIFT) + ABASE, c, v);//Compare And Swap
}
static final <K,V> void setTabAt(Node<K,V>[] tab, int i, Node<K,V> v) {
    U.putObjectVolatile(tab, ((long)i << ASHIFT) + ABASE, v);//设定值
}
```

### put
```java
public V put(K key, V value) {
    return putVal(key, value, false);
}

/** Implementation for put and putIfAbsent */
final V putVal(K key, V value, boolean onlyIfAbsent) {//多线程操作
    if (key == null || value == null) throw new NullPointerException();
    int hash = spread(key.hashCode());//hash值
    int binCount = 0;
    for (Node<K,V>[] tab = table;;) {
        Node<K,V> f; int n, i, fh;
        if (tab == null || (n = tab.length) == 0)//首次加入元素，初始化建ConcurrentHashMap
            tab = initTable();
        else if ((f = tabAt(tab, i = (n - 1) & hash)) == null) {//当前hash应在位置处i 节点为f
            if (casTabAt(tab, i, null,
                         new Node<K,V>(hash, key, value, null)))//比较插入,当前位置处没有元素,直接插入ok
                break;                   // no lock when adding to empty bin
        }
        else if ((fh = f.hash) == MOVED)//-1 ???
            tab = helpTransfer(tab, f);
        else {//
            V oldVal = null;
            synchronized (f) {///同步锁🔒  对象：f   冲突链表加锁    当前hash应在位置---Node<f>
                if (tabAt(tab, i) == f) {//tab=table    tab节点i处仍为 f
                    if (fh >= 0) {//f的hash>0
                        binCount = 1;
                        for (Node<K,V> e = f;; ++binCount) {//记录子元素个数binCount
                            K ek;
                            if (e.hash == hash &&
                                ((ek = e.key) == key ||
                                 (ek != null && key.equals(ek)))) {//找到该元素
                                oldVal = e.val;
                                if (!onlyIfAbsent)//put if absent,otherwise return;
                                    e.val = value;//更新ok
                                break;
                            }
                            Node<K,V> pred = e;
                            if ((e = e.next) == null) {//加入链尾
                                pred.next = new Node<K,V>(hash, key, value, null);
                                break;
                            }
                        }
                    }//f的hash>0结束
                    else if (f instanceof TreeBin) {//冲突链为二叉树结构
                        Node<K,V> p;
                        binCount = 2;
                        if ((p = ((TreeBin<K,V>)f).putTreeVal(hash, key, value)) != null) {
                            oldVal = p.val;
                            if (!onlyIfAbsent)
                                p.val = value;
                        }
                    }
                }///
            }///同步锁结束
            if (binCount != 0) {///冲突链,二叉树结构中
                if (binCount >= TREEIFY_THRESHOLD)//若冲突链中个数>8,转换成二叉树结构存储
                    treeifyBin(tab, i);
                if (oldVal != null)
                    return oldVal;
                break;
            }
        }
    }
    addCount(1L, binCount);///
    return null;
}
```

#### put-related
```java
///首次初始化----单例模式
private final Node<K,V>[] initTable() {
    Node<K,V>[] tab; int sc;
    while ((tab = table) == null || tab.length == 0) {
        if ((sc = sizeCtl) < 0)//sizeCtl initialized with DEFAULT(16) or tableSizeFor(1.5*initialCapacity)
            Thread.yield(); // lost initialization race; just spin  //yield ...让步 some(unspecified) time later run again
        else if (U.compareAndSwapInt(this, SIZECTL, sc, -1)) {//sizeCtl == sc则赋值为-1,令其他线程等待在yield() some time, 确保只有第一个线程进入初始化
            try {
                if ((tab = table) == null || tab.length == 0) {
                    int n = (sc > 0) ? sc : DEFAULT_CAPACITY;
                    @SuppressWarnings("unchecked")
                    Node<K,V>[] nt = (Node<K,V>[])new Node<?,?>[n];
                    table = tab = nt;
                    sc = n - (n >>> 2);///变为阈值,控制增长
                }
            } finally {
                sizeCtl = sc;////...ok
            }
            break;
        }else{///其他线程,无法完成替换
          continue;
        }
    }
    return tab;
}

// static final class CounterCell {
//     volatile long value;
//     CounterCell(long x) { value = x; }
// }

/**
 * when put-putVal x=1  binCount=链中子元素个数 or 2
 *
 * @param x the count to add
 * @param check if <0, don't check resize, if <=1 only check if uncontended
 */
private final void addCount(long x, int check) {
    CounterCell[] as; long b, s;

    //加操作
    if ((as = counterCells) != null ||//尚未进行addCount操作时,counterCells=null
        !U.compareAndSwapLong(this, BASECOUNT, b = baseCount, s = b + x)) {//交换操作失败:baseCount赋值失败
        // 1. 进行过addCount操作
        // 2. baseCount赋值失败(首次)
        CounterCell a; long v; int m;
        boolean uncontended = true;
        if (as == null || (m = as.length - 1) < 0 ||
            (a = as[ThreadLocalRandom.getProbe() & m]) == null ||
            !(uncontended =
              U.compareAndSwapLong(a, CELLVALUE, v = a.value, v + x))) {// Class<?> ck = CounterCell.class;
                                                                        // CELLVALUE = U.objectFieldOffset(ck.getDeclaredField("value"));
            //首次初始化or
            fullAddCount(x, uncontended);//uncounted=true 1. baseCount赋值失败(首次) 2. CounterCell.value复制成功 ; false 1. CounterCell赋值失败
            return;
        }
        if (check <= 1)
            return;
        s = sumCount();//
    }

    //check if the count > sizeCtl (增长)  resize --->
    if (check >= 0) {
        Node<K,V>[] tab, nt; int n, sc;
        while (s >= (long)(sc = sizeCtl) && (tab = table) != null &&
               (n = tab.length) < MAXIMUM_CAPACITY) {//ok
            int rs = resizeStamp(n);
            if (sc < 0) {
                if ((sc >>> RESIZE_STAMP_SHIFT) != rs || sc == rs + 1 ||
                    sc == rs + MAX_RESIZERS || (nt = nextTable) == null ||
                    transferIndex <= 0)
                    break;
                if (U.compareAndSwapInt(this, SIZECTL, sc, sc + 1))
                    transfer(tab, nt);
            }
            else if (U.compareAndSwapInt(this, SIZECTL, sc,(rs << RESIZE_STAMP_SHIFT) + 2))
                transfer(tab, null);
            s = sumCount();
        }
    }
}
```

// Later


### get
```java
public V get(Object key) {
    Node<K,V>[] tab; Node<K,V> e, p; int n, eh; K ek;
    int h = spread(key.hashCode());//当前k在tab中的序列：spread(k.hash)&（tab.length-1）
    if ((tab = table) != null && (n = tab.length) > 0 &&
        (e = tabAt(tab, (n - 1) & h)) != null) {//
        if ((eh = e.hash) == h) {//array[]中
            if ((ek = e.key) == key || (ek != null && key.equals(ek))) return e.val;
        }
        ///////
        else if (eh < 0)//why be negative ?          
            return (p = e.find(h, key)) != null ? p.val : null;//冲突链表中查找
        ///////
        while ((e = e.next) != null) {//list中... 发生hash碰撞
            if (e.hash == h &&
                ((ek = e.key) == key || (ek != null && key.equals(ek))))
                return e.val;
        }
    }
    return null;
}
```
