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
public interface ConcurrentMap<K, V> extends Map<K, V>{...}
```

A java.util.Map providing thread safety and atomicity guarantees.<!-- 线程安全，原子性操作 -->   

Memory consistency effects( *内存一致性* ): As with other concurrent collections, actions in a thread prior to placing an object into a ConcurrentMap as a key or value [ *happen-before* ]() actions subsequent to the access or removal of that object from the  ConcurrentMap in another thread.     
<!-- 与其他并发集合一样，在将对象放入ConcurrentMap作为键或值之前的线程中的操作发生在从另一个线程中的 ConcurrentMap访问或删除该对象之后的操作之前。 -->     
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
    implements ConcurrentMap<K,V>, Serializable{...}
```

- All operations are thread-safe,retrieval operations do not entail locking, and **there is not any support for locking the entire table in a way that prevents all access.**               
<!-- 所有操作都是线程安全的，检索操作不需要锁定，并且不支持以阻止所有访问的方式锁定整个表。 -->       
This class is fully interoperable with Hashtable in programs that rely on its thread safety **but not on its synchronization details.**     
<!-- 这个类在依赖线程安全而不依赖synchronization细节的程序中与Hashtable完全互操作。 -->
Retrievals reflect the results of the most recently completed update operations holding upon their onset.   
<!-- 检索反映了最新完成的更新操作在开始时的结果。-->   
More formally, an update operation for a given key bears a happens-before relation with any (non-null) retrieval for that key reporting the updated value.<!-- 更新操作在检索之前发生 -->

- Bear in mind that the results of aggregate status methods including `size`, `isEmpty`, and `containsValue` are typically useful only when a map is not undergoing concurrent updates in other threads.<!-- aggregate status methods 聚合状态方法的结果仅在其他线程未进行更新操作时返回 -->       
Otherwise, the results of these methods reflect transient states that may be adequate for monitoring or estimation purposes, but not for program control.<!-- 否则，聚合状态结果仅表示当前瞬时的集合状态，虽然可能符合检测与评估的目的，但是无法满足编程要求 -->

- scalable frequency map form of histogram or multiset) by using {@linkjava.util.concurrent.atomic.LongAdder} values and initializing via {@link #computeIfAbsent computeIfAbsent}.      
For example, to add a count to a {@code ConcurrentHashMap<String,LongAdder> freqs}, you can use {@code freqs.computeIfAbsent(k -> new LongAdder()).increment();} <!-- LongAdder 与 AtomicLong -->

- Like `Hashtable` but unlike `HashMap`, this class does <em>not</em> allow null to be used as a key or value.


---
Methods:   
`forEach`, `search`, `reduce`    
<!-- reductions: 约减，减少  -->
Plain reductions: (There is not a form of this method for (key, value) function arguments since there is no corresponding return type.)     
Mapped reductions: that accumulate the results of a given function applied to each element.    
Reductions to scalar doubles, longs and ints , using a given basis value.   

---
These bulk operations accept a {@code parallelismThreshold} argument.   
这些批量操作接受@code parallelismThreshold参数。     
Methods proceed sequentially if the current map size is estimated to be less than the given threshold.   
如果当前映射大小估计小于给定阈值，则方法按顺序进行。   
Using a value of {@code Long.MAX_VALUE} **suppresses all parallelism.**   
**使用@code long.max_value的值将禁止所有并行。**   

Using a value of {@code 1} results in maximal parallelism by partitioning into enough subtasks to fully utilize the {@link ForkJoinPool#commonPool()} that is used for all parallel computations.   
使用{@代码1 }的值通过划分成足够多的子任务来最大化并行性，以便充分利用{ Link Link FokCuangPooSyPooPoCo（）}，用于所有并行计算。   

<!--
Normally, you would initially choose one of these extreme values, and then measure performance of using in-between values that trade off overhead versus throughput.   
通常，您将首先选择这些极值中的一个，然后测量使用开销与吞吐量之间的值之间的使用性能。   
<p>The concurrency properties of bulk operations follow from those of ConcurrentHashMap: Any non-null result returned from {@code get(key)} and related access methods bears a happens-before relation with the associated insertion or update.   
<P>批量操作的并发属性遵循CONCURNESHASMAP：从{@代码获取（key）}返回的任何非空结果和相关的访问方法在与相关插入或更新关系之前发生一个发生。      
The result of any bulk operation reflects the composition of these per-element relations (but is not necessarily atomic with respect to the map as a whole unless it is somehow known to be quiescent).      
任何批量操作的结果都反映了这些每元素关系的组成（但对于整个映射来说不一定是原子关系，除非知道它是静态的）。      
Conversely, because keys and values in the map are never null, null serves as a reliable atomic indicator of the current lack of any result.      
相反，由于映射中的键和值从不为空，因此空可以作为当前缺少任何结果的可靠原子指示器。      
To maintain this property, null serves as an implicit basis for all non-scalar reduction operations.      
要维护此属性，空值用作所有非标量缩减操作的隐式基础。      
For the double, long, and int versions, the basis should be one that, when combined with any other value, returns that other value (more formally, it should be the identity element for the reduction).      
对于double、long和int版本，基础应该是当与任何其他值组合时返回该其他值的基础（更正式地说，它应该是用于减少的标识元素）。      
Most common reductions have these properties; for example, computing a sum with basis 0 or a minimum with basis MAX_VALUE.      
最常见的约简具有这些性质；例如，用基数0计算和或用基数最大值计算最小值。      
<p>Search and transformation functions provided as arguments should similarly return null to indicate the lack of any result (in which case it is not used).      
<p>作为参数提供的搜索和转换函数同样应返回空值，以指示缺少任何结果（在这种情况下，不使用该结果）。      
In the case of mapped reductions, this also enables transformations to serve as filters, returning null (or, in the case of primitive specializations, the identity basis) if the element should not be combined.      
在映射缩减的情况下，这还允许转换充当过滤器，如果不应组合元素，则返回空值（或者在原始专门化的情况下返回标识基础）。      
You can create compound transformations and filterings by composing them yourself under this "null means there is nothing there now" rule before using them in search or reduce operations.      
在搜索或减少操作中使用复合转换和过滤之前，您可以通过自己在“空意味着现在没有”规则下组合它们来创建复合转换和过滤。      
<p>Methods accepting and/or returning Entry arguments maintain key-value associations.      
<p>接受和/或返回入口参数的方法维护键值关联。      
They may be useful for example when finding the key for the greatest value.      
例如，当找到最大值的键时，它们可能很有用。      
Note that "plain" Entry arguments can be supplied using {@code new AbstractMap.SimpleEntry(k,v)}.      
请注意，可以使用@code new abstractmap.simpleentry（k，v）提供“plain”条目参数。      
<p>Bulk operations may complete abruptly, throwing an exception encountered in the application of a supplied function.      
<p>批量操作可能会突然完成，从而引发在应用提供的函数时遇到的异常。      
Bear in mind when handling such exceptions that other concurrently executing functions could also have thrown exceptions, or would have done so if the first exception had not occurred.      
在处理此类异常时，请记住，其他同时执行的函数也可能引发异常，或者如果没有发生第一个异常，则会引发异常。      
<p>Speedups for parallel compared to sequential forms are common but not guaranteed.      
<p>与顺序形式相比，并行加速是常见的，但不能保证。      
Parallel operations involving brief functions on small maps may execute more slowly than sequential forms if the underlying work to parallelize the computation is more expensive than the computation itself.      
如果并行计算的底层工作比计算本身更昂贵，那么在小地图上涉及简短函数的并行操作可能比顺序形式执行得慢。      
Similarly, parallelization may not lead to much actual parallelism if all processors are busy performing unrelated tasks.      
同样，如果所有处理器都忙于执行无关的任务，并行化可能不会导致太多实际的并行性。      
<p>All arguments to all task methods must be non-null.      
<p>所有任务方法的所有参数都必须为非空。
-->

Insertion (via put or its variants) of the first node in an empty bin is performed by just CASing it to the bin.  This is by far the most common case for put operations under most key/hash distributions.  

Other update operations (insert, delete, and replace) require locks.  
We do not want to waste the space required to associate a distinct lock object with each bin, so instead use the first node of a bin list itself as a lock.
其他更新操作（插入、删除和替换）需要锁。
我们不想浪费将不同的锁对象与每个bin关联所需的空间，因此使用bin列表本身的第一个节点作为锁。   

---
# Code
[ConcurrentHashMap](https://guanyuespace.github.io/2019/05/23/ConcurrentHashMap_code.html)
