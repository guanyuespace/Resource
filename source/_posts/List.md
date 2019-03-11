---
title: List
date: 2019-02-25 08:33:59
categories:
- Java
- SourceCode
- List
tags:
- Java
- List
---
# List
> 简述，[具体参见](/2019/02/26/AbstractList/)

```java
Interface： List(public interface List<E> extends Collection<E> )
The <tt>List</tt> interface provides two methods to search for a specified  object.   
From a performance standpoint, these methods should be used with caution.      
In many implementations they will perform costly linear searches.    

Abstract class: AbstractList(public abstract class AbstractList<E> extends AbstractCollection<E> implements List<E>)  
To implement an unmodifiable list, the programmer needs only to extend this class and provide implementations for the {@link #get(int)} and {@link List#size() size()} methods.

To implement a modifiable list, the programmer must additionally override the {@link #set(int, Object) set(int, E)} method (which otherwise throws an {@code UnsupportedOperationException}).  If the list is variable-size the programmer must additionally override the {@link #add(int, Object) add(int, E)} and {@link #remove(int)} methods.


about---> Itr, ListItr, RandomAccessSubList <---------


class: ArrayList(public class ArrayList<E> extends AbstractList<E> implements List<E>, RandomAccess, Cloneable, java.io.Serializable)
transient Object[] elementData; // non-private to simplify nested class access
```
<!-- more -->
## AbstractList
> what is the elementData ...  how can i store sth ?  

```java  
public abstract class AbstractList<E> extends AbstractCollection<E> implements List<E> {

  //Iterator--->cursor
  private class Itr implements Iterator<E> {
  }
  private class ListItr extends Itr implements ListIterator<E> {
  }
}
//
class SubList<E> extends AbstractList<E> {
}
class RandomAccessSubList<E> extends SubList<E> implements RandomAccess {
}
```


## ArrayList
```java
// The array buffer into which the elements of the ArrayList are stored.  
// The capacity of the ArrayList is the length of this array buffer.  
// Any empty ArrayList with elementData == DEFAULTCAPACITY_EMPTY_ELEMENTDATA  
// will be expanded to DEFAULT_CAPACITY when the first element is added.  
transient Object[] elementData; // non-private to simplify nested class access
```


---
## AbstractSequentialList  
```java
public abstract class AbstractSequentialList<E> extends AbstractList<E>
```

## LinkedList
>```java
public class LinkedList<E>
    extends AbstractSequentialList<E>
    implements List<E>, Deque<E>, Cloneable, java.io.Serializable
```
