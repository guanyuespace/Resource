---
title: Map
date: 2019-03-01 14:02:27
categories:
- Java
- SourceCode
- Map
tags:  
- Java
- Map
---
# Map
> SubClass should implements non-arguments constructor and constructor(Map)

This interface takes the place of the <tt>Dictionary</tt> class, which was a totally abstract class rather than an interface.    

*Note: great care must be exercised if mutable objects are used as map keys.*      
<!--for hashMap duplicate key --&gt; newValue will cover oldValue -->  
The behavior of a map is not specified if the value of an object is changed in a manner that affects <tt>equals</tt> comparisons while the object is a key in the map.     
A special case of this prohibition is that it is not permissible for a map to contain itself as a key.      
While it is permissible for a map to contain itself as a value, extreme caution is advised: the <tt>equals</tt> and <tt>hashCode</tt> methods are no longer well defined on such a map.   


```java
public interface Map<K,V> {
  /**
    * Replaces each entry's value with the result of invoking the given function on that entry until all entries have been processed or the function throws an exception.  
    * Exceptions thrown by the function are relayed to the caller.
    *
    * @implSpec
    * <p>The default implementation is equivalent to, for this {@code map}:
    * <pre> {@code
    * for (Map.Entry<K, V> entry : map.entrySet())
    *     entry.setValue(function.apply(entry.getKey(), entry.getValue()));
    * }</pre>
    *
    * <p>The default implementation makes no guarantees about synchronization or atomicity properties of this method.
    * Any implementation providing atomicity guarantees must override this method and document its concurrency properties.
    *
    * @param function the function to apply to each entry
    * @since 1.8
    */
   default void replaceAll(BiFunction<? super K, ? super V, ? extends V> function) {
       Objects.requireNonNull(function);
       for (Map.Entry<K, V> entry : entrySet()) {
           K k;
           V v;
           try {
               k = entry.getKey();
               v = entry.getValue();
           } catch(IllegalStateException ise) {
               // this usually means the entry is no longer in the map.
               throw new ConcurrentModificationException(ise);
           }

           // ise thrown from function is not a cme.
           v = function.apply(k, v);

           try {
               entry.setValue(v);
           } catch(IllegalStateException ise) {
               // this usually means the entry is no longer in the map.
               throw new ConcurrentModificationException(ise);
           }
       }
   }

   /**
    * @since 1.8
    */
   default V putIfAbsent(K key, V value) {
       V v = get(key);
       if (v == null) {
           v = put(key, value);
       }
       return v;
   }

  /**
   * If the specified key is not already associated with a value (or is mapped to {@code null}), attempts to compute its value using the given mapping function and enters it into this map unless {@code null}.
   *
   * <p>If the function returns {@code null} no mapping is recorded.
   * If the function itself throws an (unchecked) exception, the exception is rethrown, and no mapping is recorded.  
   * The most common usage is to construct a new object serving as an initial mapped value or memoized result, as in:
   *
   * <pre> {@code
   * map.computeIfAbsent(key, k -> new Value(f(k)));
   * }</pre>
   *
   * <p>Or to implement a multi-value map, {@code Map<K,Collection<V>>},
   * supporting multiple values per key:
   *
   * <pre> {@code
   * map.computeIfAbsent(key, k -> new HashSet<V>()).add(v);
   * }</pre>
   *
   *
   * @implSpec
   * The default implementation is equivalent to the following steps for this {@code map}, then returning the current value or {@code null} if now absent:
   *
   * <pre> {@code
   * if (map.get(key) == null) {
   *     V newValue = mappingFunction.apply(key);
   *     if (newValue != null)
   *         map.put(key, newValue);
   * }
   * }</pre>
   *
   * <p>The default implementation makes no guarantees about synchronization
   * or atomicity properties of this method. Any implementation providing
   * atomicity guarantees must override this method and document its
   * concurrency properties. In particular, all implementations of
   * subinterface {@link java.util.concurrent.ConcurrentMap} must document
   * whether the function is applied once atomically only if the value is not
   * present.
   *
   * @param key key with which the specified value is to be associated
   * @param mappingFunction the function to compute a value
   * @return the current (existing or computed) value associated with
   *         the specified key, or null if the computed value is null
   * @since 1.8
   */
  default V computeIfAbsent(K key,
          Function<? super K, ? extends V> mappingFunction) {
      Objects.requireNonNull(mappingFunction);
      V v;
      if ((v = get(key)) == null) {//旧值为空时插入mappingFunction.apply(key))
          V newValue;
          if ((newValue = mappingFunction.apply(key)) != null) {
              put(key, newValue);
              return newValue;
          }
      }

      return v;
  }
  default V compute(K key,
          BiFunction<? super K, ? super V, ? extends V> remappingFunction) {
      Objects.requireNonNull(remappingFunction);
      V oldValue = get(key);

      V newValue = remappingFunction.apply(key, oldValue);
      if (newValue == null) {
          // delete mapping
          if (oldValue != null || containsKey(key)) {
              // something to remove
              remove(key);
              return null;
          } else {
              // nothing to do. Leave things as they were.
              return null;
          }
      } else {
          // add or replace old mapping
          put(key, newValue);
          return newValue;
      }
  }
  default V merge(K key, V value,
          BiFunction<? super V, ? super V, ? extends V> remappingFunction) {
      Objects.requireNonNull(remappingFunction);
      Objects.requireNonNull(value);
      V oldValue = get(key);
      V newValue = (oldValue == null) ? value :
                 remappingFunction.apply(oldValue, value);
      if(newValue == null) {
          remove(key);
      } else {
          put(key, newValue);
      }
      return newValue;
  }
}
```  
提供了一些default方法 ... ...

- compute(key,func(key,get(key))):
```java
newValue=func(key,get(key));
newValue==null?
    remove and return null;
    :
    put(key,newValue);
```
- computeIfAbsent(key,func(key))
```java
get(key) == null ?
    newValue=func(key) != null ?
        put(key,newValue);
        return newValue;
        :
    :return get(key);
```  
- computeIfPresent(key,func(key,get(key)))
```java
get(key) != null ?
  newValue = func(key,get(key)) !=null ?
      put(key,newValue);
      return newValue;
      :
      remove(key)
      return null;
  :
  return null;
```

### 小计
| method | func |
| :--- | :--- |
| compute | 先计算，新值不空加入 |
| computeIfAbsent | 旧值空则计算，新值不空加入 |
| computeIfPresent | 旧值不空则计算，新值不空加入 |

---
# add-ons
## Function
```java
/**
 * Represents a function that accepts one argument and produces a result.
 *
 * <p>This is a <a href="package-summary.html">functional interface</a>
 * whose functional method is {@link #apply(Object)}.
 *
 * @param <T> the type of the input to the function
 * @param <R> the type of the result of the function
 *
 * @since 1.8
 */
@FunctionalInterface
public interface Function<T, R> {

    /**
     * Applies this function to the given argument.
     *
     * @param t the function argument
     * @return the function result
     */
    R apply(T t);

    /**
     * Returns a composed function that first applies the {@code before}
     * function to its input, and then applies this function to the result.
     * If evaluation of either function throws an exception, it is relayed to
     * the caller of the composed function.
     *
     * @param <V> the type of input to the {@code before} function, and to the
     *           composed function
     * @param before the function to apply before this function is applied
     * @return a composed function that first applies the {@code before}
     * function and then applies this function
     * @throws NullPointerException if before is null
     *
     * @see #andThen(Function)
     */
    default <V> Function<V, R> compose(Function<? super V, ? extends T> before) {
        Objects.requireNonNull(before);
        return (V v) -> apply(before.apply(v));
    }

    /**
     * Returns a composed function that first applies this function to
     * its input, and then applies the {@code after} function to the result.
     * If evaluation of either function throws an exception, it is relayed to
     * the caller of the composed function.
     *
     * @param <V> the type of output of the {@code after} function, and of the
     *           composed function
     * @param after the function to apply after this function is applied
     * @return a composed function that first applies this function and then
     * applies the {@code after} function
     * @throws NullPointerException if after is null
     *
     * @see #compose(Function)
     */
    default <V> Function<T, V> andThen(Function<? super R, ? extends V> after) {
        Objects.requireNonNull(after);
        return (T t) -> after.apply(apply(t));
    }

    /**
     * Returns a function that always returns its input argument.
     *
     * @param <T> the type of the input and output objects to the function
     * @return a function that always returns its input argument
     */
    static <T> Function<T, T> identity() {
        return t -> t;
    }
}
```   
## BiFunction  
```java
/**
 * Represents a function that accepts two arguments and produces a result.
 * This is the two-arity specialization of {@link Function}.
 *
 * <p>This is a <a href="package-summary.html">functional interface</a>
 * whose functional method is {@link #apply(Object, Object)}.
 *
 * @param <T> the type of the first argument to the function
 * @param <U> the type of the second argument to the function
 * @param <R> the type of the result of the function
 *
 * @see Function
 * @since 1.8
 */
@FunctionalInterface
public interface BiFunction<T, U, R> {
  /**
    * Applies this function to the given arguments.
    *
    * @param t the first function argument
    * @param u the second function argument
    * @return the function result
    */
   R apply(T t, U u);

   /**
    * Returns a composed function that first applies this function to
    * its input, and then applies the {@code after} function to the result.
    * If evaluation of either function throws an exception, it is relayed to
    * the caller of the composed function.
    *
    * @param <V> the type of output of the {@code after} function, and of the
    *           composed function
    * @param after the function to apply after this function is applied
    * @return a composed function that first applies this function and then
    * applies the {@code after} function
    * @throws NullPointerException if after is null
    */
   default <V> BiFunction<T, U, V> andThen(Function<? super R, ? extends V> after) {
       Objects.requireNonNull(after);
       return (T t, U u) -> after.apply(apply(t, u));
   }
}
```
