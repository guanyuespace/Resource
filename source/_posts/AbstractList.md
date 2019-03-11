---
title: AbstractList
date: 2019-02-26 15:22:42
categories:
- Java
- SourceCode
- List
tags:
- Java
- List
---
# AbstractList
> ```java
public abstract class AbstractList<E> extends AbstractCollection<E> implements List<E>
```

<!-- more -->
## AbstractCollection
- To implement an unmodifiable collection, the programmer needs only to extend this class and provide implementations for the <tt>iterator</tt> and <tt>size</tt> methods.    
(The iterator returned by the <tt>iterator</tt> method must implement <tt>hasNext</tt> and <tt>next</tt>.)    

- To implement a modifiable collection, the programmer must additionally override this class's <tt>add</tt> method (which otherwise throws an <tt>UnsupportedOperationException</tt>), and the iterator returned by the <tt>iterator</tt> method must additionally implement its <tt>remove</tt> method.   
>need to implement <font color="red">**iterator()**, size() ,,, add()</font></br>

```java
public abstract class AbstractCollection<E> implements Collection<E> {
     public abstract Iterator<E> iterator();
     public abstract int size();

     //....
     public <T> T[] toArray(T[] a) {
        // Estimate size of array; be prepared to see more or fewer elements
        int size = size();
        // a[] has less size than size()   reflection --> new Instance
        T[] r = a.length >= size ? a :
                  (T[])java.lang.reflect.Array
                  .newInstance(a.getClass().getComponentType(), size);

        Iterator<E> it = iterator();
        for (int i = 0; i < r.length; i++) {
            if (! it.hasNext()) { // fewer elements than expected
                if (a == r) {//预留空间充足
                    r[i] = null; // null-terminate
                } else if (a.length < i) {//预留空间不足:不足以容纳实际元素
                    return Arrays.copyOf(r, i);
                } else {//预留空间不足：足以容纳实际元素
                    System.arraycopy(r, 0, a, 0, i);
                    if (a.length > i) {
                        a[i] = null;
                    }
                }
                return a;
            }
            r[i] = (T)it.next();
        }
        // more elements than expected
        return it.hasNext() ? finishToArray(r, it) : r;
    }

    // more elements than expected， just stored the first i elements.
    private static <T> T[] finishToArray(T[] r, Iterator<?> it) {
        int i = r.length;
        while (it.hasNext()) {
            int cap = r.length;
            if (i == cap) {//内存扩容
                int newCap = cap + (cap >> 1) + 1;
                // overflow-conscious code
                if (newCap - MAX_ARRAY_SIZE > 0)
                    newCap = hugeCapacity(cap + 1);
                r = Arrays.copyOf(r, newCap);
            }
            r[i++] = (T)it.next();
        }
        // trim if overallocated
        return (i == r.length) ? r : Arrays.copyOf(r, i);
    }

    private static int hugeCapacity(int minCapacity) {
        if (minCapacity < 0) // overflow
            throw new OutOfMemoryError
                ("Required array size too large");
        return (minCapacity > MAX_ARRAY_SIZE) ?
            Integer.MAX_VALUE :
            MAX_ARRAY_SIZE;
    }
    //... others
}
```
提供了部分实现...


## AbstractList
- To implement an unmodifiable list, the programmer needs only to extend this class and provide implementations for the`get(int)` and `size() `methods.
- To implement a modifiable list, the programmer must additionally override the `set(int, E)` method (which otherwise throws an `UnsupportedOperationException`).  If the list is variable-size the programmer must additionally override the `add(int, E)` and`remove(int)`methods.
>need to implement <font color="red">get(), size() ,,, set(), add()</font></br>  
>In this class, it has implemented Iterable#iterator(), List#listIterator

```java
public abstract class AbstractList<E> extends AbstractCollection<E> implements List<E> {
    public Iterator<E> iterator() {
        return new Itr();
    }
    public ListIterator<E> listIterator() {
       return listIterator(0);
    }

    ///////////////////////////////////////////
    public boolean equals(Object o) {
        if (o == this)
            return true;
        if (!(o instanceof List))
            return false;

        ListIterator<E> e1 = listIterator();
        ListIterator<?> e2 = ((List<?>) o).listIterator();
        while (e1.hasNext() && e2.hasNext()) {
            E o1 = e1.next();
            Object o2 = e2.next();
            if (!(o1==null ? o2==null : o1.equals(o2)))//ok
                return false;
        }
        return !(e1.hasNext() || e2.hasNext());
    }
    public int hashCode() {
        int hashCode = 1;
        for (E e : this)
            hashCode = 31*hashCode + (e==null ? 0 : e.hashCode());//
        return hashCode;
    }
}
```
### AbstractList#Itr
```java
private class Itr implements Iterator<E> {
      //Index of element to be returned by subsequent call to next.
      int cursor = 0;

      /**
       * Index of element returned by most recent call to next or previous.  
       * Reset to -1 if this element is deleted by a call to remove.
       */
      int lastRet = -1;

      /**
       * The modCount value that the iterator believes that the backing List should have.
       * If this expectation is violated, the iterator has detected concurrent modification.
       */
      int expectedModCount = modCount;

      public boolean hasNext() {
          return cursor != size();
      }

      public E next() {
          checkForComodification();
          try {
              int i = cursor;
              E next = get(i);
              lastRet = i;
              cursor = i + 1;
              return next;
          } catch (IndexOutOfBoundsException e) {
              checkForComodification();
              throw new NoSuchElementException();
          }
      }

      public void remove() {
          if (lastRet < 0)
              throw new IllegalStateException();
          checkForComodification();
          try {
              AbstractList.this.remove(lastRet);
              if (lastRet < cursor)
                  cursor--;//回退
              lastRet = -1;//Reset to -1 if this element is deleted by a call to remove.
              expectedModCount = modCount;
          } catch (IndexOutOfBoundsException e) {
              throw new ConcurrentModificationException();
          }
      }

      final void checkForComodification() {
          if (modCount != expectedModCount)
              throw new ConcurrentModificationException();
      }
  }
```
实现游标（指针）cursor, lastRet的移动: 例--remove()      

### AbstractList#ListItr
```java
private class ListItr extends Itr implements ListIterator<E> {
    ListItr(int index) {
        cursor = index;
    }

    public boolean hasPrevious() {
        return cursor != 0;
    }

    public E previous() {
        checkForComodification();
        try {
            int i = cursor - 1;
            E previous = get(i);
            lastRet = cursor = i;
            return previous;
        } catch (IndexOutOfBoundsException e) {
            checkForComodification();
            throw new NoSuchElementException();
        }
    }

    public int nextIndex() {
        return cursor;
    }

    public int previousIndex() {
        return cursor-1;
    }

    public void set(E e) {
        if (lastRet < 0)
            throw new IllegalStateException();
        checkForComodification();

        try {
            AbstractList.this.set(lastRet, e);
            expectedModCount = modCount;
        } catch (IndexOutOfBoundsException ex) {
            throw new ConcurrentModificationException();
        }
    }

    public void add(E e) {
        checkForComodification();

        try {
            int i = cursor;
            AbstractList.this.add(i, e);
            lastRet = -1;
            cursor = i + 1;
            expectedModCount = modCount;
        } catch (IndexOutOfBoundsException ex) {
            throw new ConcurrentModificationException();
        }
    }
}
```
field: modCount(每次操作时，都会对该变量进行检查)；实现基本同Itr   

---
## ArrayList
```java
public class ArrayList<E> extends AbstractList<E>
        implements List<E>, RandomAccess, Cloneable, java.io.Serializable{

   //Default initial capacity.
   private static final int DEFAULT_CAPACITY = 10;
   transient Object[] elementData; // non-private to simplify nested class access

   public ArrayList(int initialCapacity) {
        if (initialCapacity > 0) {
            this.elementData = new Object[initialCapacity];
        } else if (initialCapacity == 0) {
            this.elementData = EMPTY_ELEMENTDATA;
        } else {
            throw new IllegalArgumentException("Illegal Capacity: "+
                                               initialCapacity);
        }
    }

    public E get(int index) {
        rangeCheck(index);

        return elementData(index);
    }

    public void add(int index, E element) {
        rangeCheckForAdd(index);

        ensureCapacityInternal(size + 1);  // Increments modCount!!
        System.arraycopy(elementData, index, elementData, index + 1,
                         size - index);
        elementData[index] = element;
        size++;
    }
    private void ensureCapacityInternal(int minCapacity) {
        ensureExplicitCapacity(calculateCapacity(elementData, minCapacity));
    }

    private void ensureExplicitCapacity(int minCapacity) {
        modCount++;// will be checked when use Itr, ListItr

        // overflow-conscious code
        if (minCapacity - elementData.length > 0)
            grow(minCapacity);
    }
    private void grow(int minCapacity) {
        // overflow-conscious code
        int oldCapacity = elementData.length;
        int newCapacity = oldCapacity + (oldCapacity >> 1);
        if (newCapacity - minCapacity < 0)
            newCapacity = minCapacity;
        if (newCapacity - MAX_ARRAY_SIZE > 0)
            newCapacity = hugeCapacity(minCapacity);
        // minCapacity is usually close to size, so this is a win:
        elementData = Arrays.copyOf(elementData, newCapacity);
    }//扩容：ensureCapacity, calculateCapacity, grow: oldCapacity + (oldCapacity >> 1)

    private void writeObject(java.io.ObjectOutputStream s)
        throws java.io.IOException{}
    private void readObject(java.io.ObjectInputStream s)
        throws java.io.IOException, ClassNotFoundException {}
    //others
}
```

---
## Iterator
An iterator over a collection.  {@code Iterator} takes the place of {@link Enumeration} in the Java Collections Framework.          
Iterators differ from enumerations in two ways:    
<ul>
    <li> Iterators allow the caller to remove elements from the underlying collection during the iteration with well-defined semantics.
    <li> Method names have been improved.
</ul>

```java
public interface Iterator<E> {
    boolean hasNext();
    E next();

    default void remove() {
        throw new UnsupportedOperationException("remove");
    }

    default void forEachRemaining(Consumer<? super E> var1) {
        Objects.requireNonNull(var1);//null? throw Exception : return it;

        while(this.hasNext()) {
            var1.accept(this.next());
        }
    }
}
```

## Iterable(for-each loop)
Implementing this interface allows an object to be the target of the "for-each loop" statement. See
```java
public interface Iterable<T> {
    Iterator<T> iterator();

    default void forEach(Consumer<? super T> var1) {
        Objects.requireNonNull(var1);
        Iterator var2 = this.iterator();

        while(var2.hasNext()) {
            Object var3 = var2.next();
            var1.accept(var3);
        }

    }

    default Spliterator<T> spliterator() {
        return Spliterators.spliteratorUnknownSize(this.iterator(), 0);
    }
}
```
## Enumeration
An object that implements the Enumeration interface generates a series of elements, one at a time. Successive calls to the <code>nextElement</code> method return successive elements of the series.   
For example, to print all elements of a <tt>Vector&lt;E&gt;</tt> <i>v</i>:   
```java
for (Enumeration<E> e = v.elements(); e.hasMoreElements();)  
       System.out.println(e.nextElement());  
```
Methods are provided to enumerate through the elements of a vector, the keys of a hashtable, and the values in a hashtable.    
Enumerations are also used to specify the input streams to a <code>SequenceInputStream</code>.
```java
public interface Enumeration<E> {
    /**
     * Tests if this enumeration contains more elements.
     *
     * @return  <code>true</code> if and only if this enumeration object
     *           contains at least one more element to provide;
     *          <code>false</code> otherwise.
     */
    boolean hasMoreElements();

    /**
     * Returns the next element of this enumeration if this enumeration
     * object has at least one more element to provide.
     *
     * @return     the next element of this enumeration.
     * @exception  NoSuchElementException  if no more elements exist.
     */
    E nextElement();
}
```

---
## ListIterator
```java
public interface ListIterator<E> extends Iterator<E> {
    boolean hasNext();

    E next();

    boolean hasPrevious();

    E previous();

    int nextIndex();

    int previousIndex();

    void remove();

    void set(E var1);

    void add(E var1);
}
```

---
## Consumer
函数式接口(method: accept())
```java
@FunctionalInterface
public interface Consumer<T> {
    void accept(T var1);

    default Consumer<T> andThen(Consumer<? super T> var1) {
        Objects.requireNonNull(var1);
        return (var2) -> {
            this.accept(var2);
            var1.accept(var2);
        };
    }
}
```

---

...
Arrays.copyOf();
System.arraycopy();

---
## Test about <font color="red">Itr: modCount&expectedModCount</font>
```java
List list = new ArrayList();
for (int i = 0; i < 10; i++) {
   list.add(i);
}
System.out.println("ori:" + list.toArray() + "\t" + Arrays.toString(list.toArray()) + "\t" + list.size());

Field theUnsafe = Unsafe.class.getDeclaredField("theUnsafe");
theUnsafe.setAccessible(true);
Unsafe unsafe = (Unsafe) theUnsafe.get(Unsafe.class);//or (Unsafe) theUnsafe.get(null);

Field modCountField = AbstractList.class.getDeclaredField("modCount");
modCountField.setAccessible(true);
//can't get the private class : ArrayList.Itr
//Field expectedModCountField=ArrayList.Itr.class.getDeclaredField("Itr");
long offset = unsafe.objectFieldOffset(modCountField);
for (Object o : list) {
   list.remove(o);//Exception
   //缺少回退当前指针
   unsafe.putInt(list, offset, (int) modCountField.get(list) - 1);//but can not back ...  and the modCount is wrong !  modCount： the modified times
}
System.out.println("list: " + list.toArray() + "\t" + Arrays.toString(list.toArray()) + "\t" + list.size());


Iterator iterator = list.iterator();
Field expectedModCountField = iterator.getClass().getDeclaredField("expectedModCount");
long expectedModCountOffset = unsafe.objectFieldOffset(expectedModCountField);
Field cursorField = iterator.getClass().getDeclaredField("cursor");
cursorField.setAccessible(true);
long cursorOffset = unsafe.objectFieldOffset(cursorField);
Field lastRetField = iterator.getClass().getDeclaredField("lastRet");
lastRetField.setAccessible(true);
long lastRetOffset = unsafe.objectFieldOffset(lastRetField);
while (iterator.hasNext()) {
   Object o = iterator.next();
   list.remove(o);
   unsafe.putInt(iterator, cursorOffset, (int) lastRetField.get(iterator));//回退指针
   //unsafe.putInt(iterator, lastRetOffset, -1);//删除后要重置-1
   unsafe.putInt(iterator, expectedModCountOffset, (int) modCountField.get(list));

}
System.out.println("list: " + list.toArray() + "\t" + Arrays.toString(list.toArray()) + "\t" + list.size());
```

Result:   
```
ori:[Ljava.lang.Object;@5e481248	[0, 1, 2, 3, 4, 5, 6, 7, 8, 9]	10
list: [Ljava.lang.Object;@63947c6b	[1, 3, 5, 7, 9]	5
list: [Ljava.lang.Object;@355da254	[]	0

Process finished with exit code 0
```

Class文件:  
```java
List list = new ArrayList();

for(int i = 0; i < 10; ++i) {
   list.add(i);
}

System.out.println("ori:" + list.toArray() + "\t" + Arrays.toString(list.toArray()) + "\t" + list.size());
Field theUnsafe = Unsafe.class.getDeclaredField("theUnsafe");
theUnsafe.setAccessible(true);
Unsafe unsafe = (Unsafe)theUnsafe.get(Unsafe.class);
Field modCountField = AbstractList.class.getDeclaredField("modCount");
modCountField.setAccessible(true);
long offset = unsafe.objectFieldOffset(modCountField);
Iterator iterator = list.iterator();//for-each 循环

while(iterator.hasNext()) {
   Object o = iterator.next();
   list.remove(o);
   unsafe.putInt(list, offset, (Integer)modCountField.get(list) - 1);
}

System.out.println("list: " + list.toArray() + "\t" + Arrays.toString(list.toArray()) + "\t" + list.size());
iterator = list.iterator();
Field expectedModCountField = iterator.getClass().getDeclaredField("expectedModCount");
long expectedModCountOffset = unsafe.objectFieldOffset(expectedModCountField);
Field cursorField = iterator.getClass().getDeclaredField("cursor");
cursorField.setAccessible(true);
long cursorOffset = unsafe.objectFieldOffset(cursorField);
Field lastRetField = iterator.getClass().getDeclaredField("lastRet");
lastRetField.setAccessible(true);
unsafe.objectFieldOffset(lastRetField);

while(iterator.hasNext()) {
   Object o = iterator.next();
   list.remove(o);
   unsafe.putInt(iterator, cursorOffset, (Integer)lastRetField.get(iterator));
   unsafe.putInt(iterator, expectedModCountOffset, (Integer)modCountField.get(list));
}

System.out.println("list: " + list.toArray() + "\t" + Arrays.toString(list.toArray()) + "\t" + list.size());
```

补充反射：尝试--> ArrayList.class.getDeclaredClasses() , class.forName("java.util.ArrayList$Itr")  ...    
