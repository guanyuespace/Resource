---
title: TreeMap(红黑树-平衡二叉查找树相关介绍)
date: 2019-05-15 08:49:45
categories:
- Java
- Introduction
- TreeMap
tags:
- Java
- Introduction
- TreeMap
---
# 红黑树
红黑树是一种近似平衡的二叉查找树，它能够确保任何一个节点的左右子树的高度差不会超过二者中较低那个的一陪。具体来说，红黑树是满足如下条件的二叉查找树（binary search tree）：   

1. 每个节点要么是红色，要么是黑色。
2. 根节点必须是黑色
3. 红色节点不能连续（也即是，红色节点的孩子和父亲都不能是红色）。<!-- 3. 每个红色节点的两个子节点都是黑色。(从每个叶子到根的所有路径上不能有两个连续的红色节点); -->    
4. **对于每个节点，从该点至null（树尾端）的任何路径，都含有相同个数的黑色节点。**<!-- 黑高：限制了任意一条路径不能比另一路径长2倍以上   例：{黑高=4 【最短：黑，黑，黑，黑】  【最长：黑，红，黑，红，黑，红，黑 】}-->   

![红黑树示例](TreeMap_base.png "红黑树示例")

<font size="-2">注：在树的结构发生改变时（插入或者删除操作），往往会破坏上述条件3或条件4，需要通过调整使得查找树重新满足红黑树的约束条件。</font>

<!-- more -->

## 预备知识
>**二叉查找树：左小右大，以此来对应左旋右旋后子节点的位置变化**

前文说到当查找树的结构发生改变时，红黑树的约束条件可能被破坏，需要通过调整使得查找树重新满足红黑树的约束条件。调整可以分为两类：一类是颜色调整，即改变某个节点的颜色；另一类是结构调整，集改变检索树的结构关系。结构调整过程包含两个基本操作：左旋（Rotate Left），右旋（RotateRight）。

### 节点定义
```java
private static final boolean RED   = false;
private static final boolean BLACK = true;

static final class Entry<K,V> implements Map.Entry<K,V> {
    K key;
    V value;
    Entry<K,V> left;
    Entry<K,V> right;
    Entry<K,V> parent;
    boolean color = BLACK;
}
```   

### 左旋
左旋的过程是将x的右子树绕x逆时针旋转，使得x的右子树成为x的父亲，同时修改相关节点的引用。旋转之后，二叉查找树的属性仍然满足。  

![左旋](TreeMap_rotateLeft.png "左旋")   

```java
/** From CLR */
private void rotateLeft(Entry<K,V> p) {
    if (p != null) {
        Entry<K,V> r = p.right;
        p.right = r.left;//构建父子关系:父-->子p.right-r.Left
        if (r.left != null)
            r.left.parent = p;//构建父子关系：子-->父

        r.parent = p.parent;//转让父节点
        if (p.parent == null)
            root = r;
        else if (p.parent.left == p)
            p.parent.left = r;
        else
            p.parent.right = r;

        r.left = p;//建立p与r的父子关系
        p.parent = r;
    }
}
```
![左旋代码示例](rotate_left.left "左旋代码示例")

### 右旋
右旋的过程是将x的左子树绕x顺时针旋转，使得x的左子树成为x的父亲，同时修改相关节点的引用。旋转之后，二叉查找树的属性仍然满足。  

![右旋](TreeMap_rotateRight.png "右旋")

```java
/** From CLR */
private void rotateRight(Entry<K,V> p) {
   if (p != null) {
       Entry<K,V> l = p.left;
       p.left = l.right;
       if (l.right != null) l.right.parent = p;
       l.parent = p.parent;
       if (p.parent == null)
           root = l;
       else if (p.parent.right == p)
           p.parent.right = l;
       else p.parent.left = l;
       l.right = p;
       p.parent = l;
   }
}
```

![右旋代码示例](rotate_right.jpg "右旋代码示例")   

### 寻找后继节点
>比当前节点大的最小元素

**查找思路：**
1. t的右子树不空，则t的后继是其右子树中最小的那个元素。<!-- 右子树中最左-->   
2. t的右孩子为空，则t的后继是其第一个向左走的祖先。<!-- 祖先节点的左。。-->

![寻找后继节点](TreeMap_successor.png "寻找后继节点")   

```java
/**
 * Returns the successor of the specified Entry, or null if no such.
 */
static <K,V> TreeMap.Entry<K,V> successor(Entry<K,V> t) {
    if (t == null)
        return null;
    else if (t.right != null) {
        Entry<K,V> p = t.right;
        while (p.left != null)
            p = p.left;
        return p;
    } else {
        Entry<K,V> p = t.parent;
        Entry<K,V> ch = t;
        while (p != null && ch == p.right) {//向左上走
            ch = p;
            p = p.parent;
        }
        return p;
    }
}
```

## 插入节点
>插入节点，重构二叉查找平衡树

```java
public V put(K key, V value) {
    Entry<K,V> t = root;
    if (t == null) {
        compare(key, key); // type (and possibly null) check

        root = new Entry<>(key, value, null);
        size = 1;
        modCount++;
        return null;
    }
    int cmp;
    Entry<K,V> parent;
    // split comparator and comparable paths
    Comparator<? super K> cpr = comparator;//自定义了比较方式：Comparator
    if (cpr != null) {
        do {
            parent = t;
            cmp = cpr.compare(key, t.key);
            if (cmp < 0)
                t = t.left;
            else if (cmp > 0)
                t = t.right;
            else
                return t.setValue(value);//有相同key则更新
        } while (t != null);
    }
    else {
        if (key == null)
            throw new NullPointerException();
        @SuppressWarnings("unchecked")
            Comparable<? super K> k = (Comparable<? super K>) key;
        do {
            parent = t;
            cmp = k.compareTo(t.key);
            if (cmp < 0)
                t = t.left;
            else if (cmp > 0)
                t = t.right;
            else
                return t.setValue(value);
        } while (t != null);
    }
    Entry<K,V> e = new Entry<>(key, value, parent);//新建节点以备插入
    if (cmp < 0)
        parent.left = e;
    else
        parent.right = e;
    fixAfterInsertion(e);//调整
    size++;
    modCount++;//ok... ... 结构变化
    return null;
}
```
插入or更新节点


### 颜色调整
>1. 每个节点要么是红色，要么是黑色。
>2. 根节点必须是黑色
>3. 每个红色节点的两个子节点都是黑色。(从每个叶子到根的所有路径上不能有两个连续的红色节点);红色节点不能连续（也即是，红色节点的孩子和父亲都不能是红色）。
>4. **对于每个节点，从该点至null（树尾端）的任何路径，都含有相同个数的黑色节点。**   
>   
>性质1规定红黑树节点的颜色要么是红色要么是黑色，那么在插入新节点时，这个节点应该是红色还是黑色呢？ **答案是红色**    
>如果插入的节点是黑色，那么这个节点所在路径比其他路径多出一个黑色节点，这个调整起来会比较麻烦（参考红黑树的删除操作，就知道为啥多一个或少一个黑色节点时，调整起来这么麻烦了）。如果插入的节点是红色，此时所有路径上的黑色节点数量不变，仅可能会出现两个连续的红色节点的情况。这种情况下，通过变色和旋转进行调整即可，比之前的简单多了。--摘自[红黑树详细分析，看了都说好](https://www.imooc.com/article/22894)


**设置颜色的意义：以此来作为左旋右旋的判断**   
```java
/** From CLR */
private void fixAfterInsertion(Entry<K,V> x) {
   x.color = RED;
   while (x != null && x != root && x.parent.color == RED) {//根节点黑色，红色父节点的子节点黑色
       if (parentOf(x) == leftOf(parentOf(parentOf(x)))) {//parentOf: return (p == null ? null: p.parent);
                                                          // leftOf: return (p == null) ? null: p.left;
           Entry<K,V> y = rightOf(parentOf(parentOf(x)));
           if (colorOf(y) == RED) {
               setColor(parentOf(x), BLACK);
               setColor(y, BLACK);
               setColor(parentOf(parentOf(x)), RED);
               x = parentOf(parentOf(x));
           } else {
               if (x == rightOf(parentOf(x))) {
                   x = parentOf(x);
                   rotateLeft(x);//
               }
               setColor(parentOf(x), BLACK);
               setColor(parentOf(parentOf(x)), RED);
               rotateRight(parentOf(parentOf(x)));
           }
       } else {
           Entry<K,V> y = leftOf(parentOf(parentOf(x)));
           if (colorOf(y) == RED) {
               setColor(parentOf(x), BLACK);
               setColor(y, BLACK);
               setColor(parentOf(parentOf(x)), RED);
               x = parentOf(parentOf(x));
           } else {
               if (x == leftOf(parentOf(x))) {
                   x = parentOf(x);
                   rotateRight(x);
               }
               setColor(parentOf(x), BLACK);
               setColor(parentOf(parentOf(x)), RED);
               rotateLeft(parentOf(parentOf(x)));
           }
       }
   }
   root.color = BLACK;
}
```
红黑树颜色调整


Later

## 删除节点
>左右子树
>1. 左右子树均不为空，左子节点ok，  右子节点后继节点左子节点
>...
>   
>相较于插入操作，红黑树的删除操作则要更为复杂一些。    
>删除操作首先要确定待删除节点有几个孩子，如果有两个孩子，不能直接删除该节点。   
>而是要先找到该节点的前驱（该节点左子树中最大的节点）或者后继（该节点右子树中最小的节点），   
>然后将前驱或者后继的值复制到要删除的节点中，最后再将前驱或后继删除。   
>由于前驱和后继至多只有一个孩子节点，这样我们就把原来要删除的节点有两个孩子的问题转化为只有一个孩子节点的问题，问题被简化了一些。   
>我们并不关心最终被删除的节点是否是我们开始想要删除的那个节点，只要节点里的值最终被删除就行了，至于树结构如何变化，这个并不重要。   



```java
/**
 * Delete node p, and then rebalance the tree.
 */
private void deleteEntry(Entry<K,V> p) {
    modCount++;
    size--;

    // If strictly internal, copy successor's element to p and then make p
    // point to successor.
    if (p.left != null && p.right != null) {
        Entry<K,V> s = successor(p);
        p.key = s.key;
        p.value = s.value;
        p = s;
    } // p has 2 children

    // Start fixup at replacement node, if it exists.
    Entry<K,V> replacement = (p.left != null ? p.left : p.right);

    if (replacement != null) {
        // Link replacement to parent
        replacement.parent = p.parent;
        if (p.parent == null)
            root = replacement;
        else if (p == p.parent.left)
            p.parent.left  = replacement;
        else
            p.parent.right = replacement;

        // Null out links so they are OK to use by fixAfterDeletion.
        p.left = p.right = p.parent = null;

        // Fix replacement
        if (p.color == BLACK)
            fixAfterDeletion(replacement);
    } else if (p.parent == null) { // return if we are the only node.
        root = null;
    } else { //  No children. Use self as phantom replacement and unlink.
        if (p.color == BLACK)
            fixAfterDeletion(p);

        if (p.parent != null) {
            if (p == p.parent.left)
                p.parent.left = null;
            else if (p == p.parent.right)
                p.parent.right = null;
            p.parent = null;
        }
    }
}
```

### 颜色调整

```java
/** From CLR */
private void fixAfterDeletion(Entry<K,V> x) {
    while (x != root && colorOf(x) == BLACK) {
        if (x == leftOf(parentOf(x))) {
            Entry<K,V> sib = rightOf(parentOf(x));

            if (colorOf(sib) == RED) {
                setColor(sib, BLACK);
                setColor(parentOf(x), RED);
                rotateLeft(parentOf(x));
                sib = rightOf(parentOf(x));
            }

            if (colorOf(leftOf(sib))  == BLACK &&
                colorOf(rightOf(sib)) == BLACK) {
                setColor(sib, RED);
                x = parentOf(x);
            } else {
                if (colorOf(rightOf(sib)) == BLACK) {
                    setColor(leftOf(sib), BLACK);
                    setColor(sib, RED);
                    rotateRight(sib);
                    sib = rightOf(parentOf(x));
                }
                setColor(sib, colorOf(parentOf(x)));
                setColor(parentOf(x), BLACK);
                setColor(rightOf(sib), BLACK);
                rotateLeft(parentOf(x));
                x = root;
            }
        } else { // symmetric
            Entry<K,V> sib = leftOf(parentOf(x));

            if (colorOf(sib) == RED) {
                setColor(sib, BLACK);
                setColor(parentOf(x), RED);
                rotateRight(parentOf(x));
                sib = leftOf(parentOf(x));
            }

            if (colorOf(rightOf(sib)) == BLACK &&
                colorOf(leftOf(sib)) == BLACK) {
                setColor(sib, RED);
                x = parentOf(x);
            } else {
                if (colorOf(leftOf(sib)) == BLACK) {
                    setColor(rightOf(sib), BLACK);
                    setColor(sib, RED);
                    rotateLeft(sib);
                    sib = leftOf(parentOf(x));
                }
                setColor(sib, colorOf(parentOf(x)));
                setColor(parentOf(x), BLACK);
                setColor(leftOf(sib), BLACK);
                rotateRight(parentOf(x));
                x = root;
            }
        }
    }

    setColor(x, BLACK);
}
```


---
# 拓展
## 平衡二叉树
>根据delta,调整高度   
