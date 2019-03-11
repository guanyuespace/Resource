---
title: StringBuilder&StringBuffer
date: 2019-03-01 14:02:17
categories:
- Java
- SourceCode
- StringBuilder
tags:
- Java
- String
---
# StringBuilder
>A mutable sequence of characters.  This class provides an API compatible with {@code `StringBuffer`}, but `with no guarantee of synchronization`.   
This class is designed for use as a drop-in replacement for {@code StringBuffer} in places where the string buffer was being used by a single thread (as is generally the case).   
Where possible,it is recommended that this class be used in preference to {@code StringBuffer} as it will be faster under most implementations.    
Unless otherwise noted, passing a {@code null} argument to a constructor or method in this class will cause a {@link NullPointerException} to be thrown.   
```java
public final class StringBuilder
    extends AbstractStringBuilder
    implements java.io.Serializable, CharSequence
```

<!-- more -->

# AbstractStringBuilder
```java
abstract class AbstractStringBuilder implements Appendable, CharSequence {
    // The value is used for character storage.
    char[] value;

    // The count is the number of characters used.
    int count;

    // This no-arg constructor is necessary for serialization of subclasses.
    AbstractStringBuilder() {
    }

    // Creates an AbstractStringBuilder of the specified capacity.
    AbstractStringBuilder(int capacity) {
        value = new char[capacity];
    }

    public AbstractStringBuilder append(String str) {
        if (str == null)
            return appendNull();
        int len = str.length();
        ensureCapacityInternal(count + len);//ensure enough capacity
        str.getChars(0, len, value, count);//copy to here(value[])
        count += len;
        return this;
    }
    private void ensureCapacityInternal(int minimumCapacity) {
        // overflow-conscious code
        if (minimumCapacity - value.length > 0) {
            value = Arrays.copyOf(value,
                    newCapacity(minimumCapacity));//copy current value into new value[]
        }
    }
    private int newCapacity(int minCapacity) {
        // overflow-conscious code
        int newCapacity = (value.length << 1) + 2;//2*L+2
        if (newCapacity - minCapacity < 0) {
            newCapacity = minCapacity;
        }
        return (newCapacity <= 0 || MAX_ARRAY_SIZE - newCapacity < 0)
            ? hugeCapacity(minCapacity)
            : newCapacity;
    }
}

String#getChars
public void getChars(int srcBegin, int srcEnd, char dst[], int dstBegin) {
    if (srcBegin < 0) {
        throw new StringIndexOutOfBoundsException(srcBegin);
    }
    if (srcEnd > value.length) {
        throw new StringIndexOutOfBoundsException(srcEnd);
    }
    if (srcBegin > srcEnd) {
        throw new StringIndexOutOfBoundsException(srcEnd - srcBegin);
    }
    System.arraycopy(value, srcBegin, dst, dstBegin, srcEnd - srcBegin);
}
```
implements append and insert methods

# StringBuilder
```java
public final class StringBuilder
    extends AbstractStringBuilder
    implements java.io.Serializable, CharSequence{

    public StringBuilder() {
      super(16);
    }

    public StringBuilder append(String str) {
        super.append(str);
        return this;
    }

    //customed method to writeObject
    private void writeObject(java.io.ObjectOutputStream s)
        throws java.io.IOException {
        s.defaultWriteObject();
        s.writeInt(count);
        s.writeObject(value);
    }
    private void readObject(java.io.ObjectInputStream s)
        throws java.io.IOException, ClassNotFoundException {
        s.defaultReadObject();
        count = s.readInt();
        value = (char[]) s.readObject();
    }
}
```
调用父类实现... ...  



# StringBuffer
```java
public final class StringBuffer
    extends AbstractStringBuilder
    implements java.io.Serializable, CharSequence{
    public StringBuffer() {
      super(16);
    }

    public synchronized StringBuffer append(String str) {
        toStringCache = null;
        super.append(str);
        return this;
    }

    /**
     * readObject is called to restore the state of the StringBuffer from
     * a stream.
     */
    private synchronized void writeObject(java.io.ObjectOutputStream s)
        throws java.io.IOException {
        java.io.ObjectOutputStream.PutField fields = s.putFields();
        fields.put("value", value);
        fields.put("count", count);
        fields.put("shared", false);
        s.writeFields();
    }

    /**
     * readObject is called to restore the state of the StringBuffer from
     * a stream.
     */
    private void readObject(java.io.ObjectInputStream s)
        throws java.io.IOException, ClassNotFoundException {
        java.io.ObjectInputStream.GetField fields = s.readFields();
        value = (char[])fields.get("value", null);
        count = fields.get("count", 0);
    }
}
```
synchronized ....  非公平锁        
ReentrantLock/Condition(sign,await) ....      

---
# addons
how can `ObjectOutputStream`/`ObjectInputStream` do here ?
