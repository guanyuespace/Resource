---
title: HashSet
date: 2019-03-22 11:10
categories:
- Java
- SourceCode
- HashSet
tags:
- Java
- HashSet
---
# HashSet
```java
public class HashSet<E>
    extends AbstractSet<E>
    implements Set<E>, Cloneable, java.io.Serializable
```
This class implements the <tt>Set</tt> interface, backed by a hash table (actually a <tt>HashMap</tt> instance).<!-- 主要依赖HashMap进行实现 -->     
<strong>Note that this implementation is not synchronized.</strong>  
`Set s = Collections.synchronizedSet(new HashSet(...));`   

<!-- more -->

... ...
## Set
Note: Great care must be exercised if mutable objects are used as set elements.(will modify the value in Set, just like clone--浅拷贝)
<!-- should also care this in HashMap,HashSet -->

... ...

***the hashcode of entry in HashMap：~~table is unmutable,~~ it is fixed when create entry, so add a mutable entry, and then you can change it(Particularly, you can make it be same with others, but its place(or hash) in table is fixed just like before.).When resize the Map, the hashcode will not change. ***     
