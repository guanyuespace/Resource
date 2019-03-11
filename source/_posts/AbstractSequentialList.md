---
title: AbstractSequentialList
date: 2019-02-27 14:26:42
categories:
- Java
- SourceCode
- List
tags:
- Java
- List
---
# AbstractSequentialList
```java
public abstract class AbstractSequentialList<E> extends AbstractList<E>
```
<!-- more -->
This class is the opposite of the <tt>AbstractList</tt> class in the sense that it implements the "random access" methods (<tt>get(int index)</tt>, <tt>set(int index, E element)</tt>, <tt>add(int index, E element)</tt> and <tt>remove(int index)</tt>) on top of the list's list iterator, instead of the other way around.   
To implement a list the programmer needs only to extend this class and provide implementations for the <tt>listIterator</tt> and <tt>size</tt> methods.  
- For an unmodifiable list, the programmer need only implement the list iterator's <tt>hasNext</tt>, <tt>next</tt>, <tt>hasPrevious</tt>, <tt>previous</tt> and <tt>index</tt> methods.    
- For a modifiable list the programmer should additionally implement the list iterator's <tt>set</tt> method.  For a variable-size list the programmer should additionally implement the list iterator's <tt>remove</tt> and <tt>add</tt> methods.
>need to implement list iterator's `hasNext/Previous()` ,`next()`,`previous()` .....  `set()`,`add()`,`remove()` ...

## AbstractSequentialList
```java
public E get(int var1) {
    try {
        return this.listIterator(var1).next();
    } catch (NoSuchElementException var3) {
        throw new IndexOutOfBoundsException("Index: " + var1);
    }
}

public E set(int var1, E var2) {
    try {
        ListIterator var3 = this.listIterator(var1);
        Object var4 = var3.next();
        var3.set(var2);
        return var4;
    } catch (NoSuchElementException var5) {
        throw new IndexOutOfBoundsException("Index: " + var1);
    }
}
public Iterator<E> iterator() {
    return this.listIterator();
}
public abstract ListIterator<E> listIterator(int var1);
```
method： get(), set(), add(), remove()   --->  AbstractList$ListItr

## LinkedList
```java
public class LinkedList<E>
    extends AbstractSequentialList<E>
    implements List<E>, Deque<E>, Cloneable, java.io.Serializable{

    transient int size = 0;
    // Pointer to first node.
    // Invariant: (first == null && last == null) ||
    //            (first.prev == null && first.item != null)
    transient Node<E> first;

    // Pointer to last node.
    // Invariant: (first == null && last == null) ||
    //            (last.next == null && last.item != null)
    transient Node<E> last;

    protected transient int modCount = 0;//structurally modified times

    ///////////////////链表操作///////////////////
    // Inserts element e before non-null Node succ.
    void linkBefore(E e, Node<E> succ) {
        // assert succ != null;
        final Node<E> pred = succ.prev;
        final Node<E> newNode = new Node<>(pred, e, succ);
        succ.prev = newNode;
        if (pred == null)
            first = newNode;
        else
            pred.next = newNode;
        size++;
        modCount++;//....
    }

    // Unlinks non-null node x.
    E unlink(Node<E> x) {
        // assert x != null;
        final E element = x.item;
        final Node<E> next = x.next;
        final Node<E> prev = x.prev;

        if (prev == null) {
            first = next;
        } else {
            prev.next = next;
            x.prev = null;
        }

        if (next == null) {
            last = prev;
        } else {
            next.prev = prev;
            x.next = null;
        }

        x.item = null;
        size--;
        modCount++;//。。。。。
        return element;
    }


    ////////////////add///remove////////////////
    public boolean add(E e) {
        linkLast(e);
        return true;
    }
    public boolean remove(Object o) {
        if (o == null) {
            for (Node<E> x = first; x != null; x = x.next) {
                if (x.item == null) {
                    unlink(x);
                    return true;
                }
            }
        } else {
            for (Node<E> x = first; x != null; x = x.next) {
                if (o.equals(x.item)) {
                    unlink(x);
                    return true;
                }
            }
        }
        return false;
    }

    private static class Node<E> {
        E item;
        Node<E> next;
        Node<E> prev;

        Node(Node<E> prev, E element, Node<E> next) {
            this.item = element;
            this.next = next;
            this.prev = prev;
        }
    }


    public ListIterator<E> listIterator(int index) {
        checkPositionIndex(index);
        return new ListItr(index);
    }

    private class ListItr implements ListIterator<E> {
        private Node<E> lastReturned;//pointer ---point to  current
        private Node<E> next;//pointer --- point to  next
        private int nextIndex;
        private int expectedModCount = modCount;

        public E next() {
            checkForComodification();//here
            if (!hasNext())
                throw new NoSuchElementException();
            //.....
            lastReturned = next;
            next = next.next;
            nextIndex++;
            return lastReturned.item;
        }
        public void remove() {
            checkForComodification();//here
            if (lastReturned == null)
                throw new IllegalStateException();

            //....
            Node<E> lastNext = lastReturned.next;
            unlink(lastReturned);
            if (next == lastReturned)
                next = lastNext;
            else
                nextIndex--;
            lastReturned = null;
            expectedModCount++;//here
        }
        //....
        final void checkForComodification() {
            if (modCount != expectedModCount)
                throw new ConcurrentModificationException();
        }
    }
}
```
....
