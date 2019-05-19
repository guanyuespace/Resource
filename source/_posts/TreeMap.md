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
3. 每个外节点（nil）是黑色的
4. 红色节点不能连续（也即是，红色节点的孩子和父亲都不能是红色）。<!-- 3. 每个红色节点的两个子节点都是黑色。(从每个叶子到根的所有路径上不能有两个连续的红色节点); -->    
5. **从某一结点到达其子孙外节点的每一条简单路径上包含相同个数的黑节点。**<!-- 黑高：限制了任意一条路径不能比另一路径长2倍以上   例：{黑高=4 【最短：黑，黑，黑，黑】  【最长：黑，红，黑，红，黑，红，黑，红】}  ，此表示为一般不含nil哨位外节点。   -->   

<!-- 《数据结构》中存在外界点nil且全部为黑色节点,bh(T*)：黑高--将从节点X出发 ~~（不包括该节点）~~ 到达一个外节点的任意路径上黑色节点的个数称为该节点的黑高，用bh(x)表示。根节点则为树黑高 -->

![红黑树示例，不含哨位节点nil](TreeMap_base.png "红黑树示例，不含哨位节点nil")
<font  size="-2">红黑树定义摘自《数据结构》第二版,刘大有</font>
<font size="-2">注：在树的结构发生改变时（插入或者删除操作），往往会破坏上述条件4或条件5，需要通过调整使得查找树重新满足红黑树的约束条件。</font>

<!-- more -->

## 预备知识
>**二叉查找树：左小右大，以此来对应左旋右旋后子节点的位置变化**

前文说到当查找树的结构发生改变时，红黑树的约束条件可能被破坏，需要通过调整使得查找树重新满足红黑树的约束条件。调整可以分为两类：一类是颜色调整，即改变某个节点的颜色；另一类是结构调整，集改变检索树的结构关系。结构调整过程包含两个基本操作：左旋（Rotate Left），右旋（RotateRight）。

### 节点定义

```java
class TreeMap{
  // ....
  private static final boolean RED   = false;
  private static final boolean BLACK = true;

  // ...
  static final class Entry<K,V> implements Map.Entry<K,V> {
    K key;
    V value;
    Entry<K,V> left;
    Entry<K,V> right;
    Entry<K,V> parent;
    boolean color = BLACK;
  }
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
![左旋代码示例](rotate_left.jpg "左旋代码示例")

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

分情况。。。

**设置颜色的意义：以此来作为左旋右旋的判断**   
```java
/** From CLR */
private void fixAfterInsertion(Entry<K,V> x) {
   x.color = RED;
   while (x != null && x != root && x.parent.color == RED) {//根节点黑色，红色父节点的子节点黑色
       if (parentOf(x) == leftOf(parentOf(parentOf(x)))) {//parentOf: return (p == null ? null: p.parent);
                                                          // leftOf: return (p == null) ? null: p.left;
           Entry<K,V> y = rightOf(parentOf(parentOf(x)));//叔叔节点
           if (colorOf(y) == RED) {//叔叔节点为红色
               setColor(parentOf(x), BLACK);//p(x),y黑色  p(p(x)) 红  
               setColor(y, BLACK);//情况，问题上移至p(p(x))
               setColor(parentOf(parentOf(x)), RED);
               x = parentOf(parentOf(x));
           } else {//叔叔节点为黑色
               if (x == rightOf(parentOf(x))) {//x为右节点 情况2-1-3
                   x = parentOf(x);
                   rotateLeft(x);//以p(x)为轴左旋
               }//变成情况2-1-2
               setColor(parentOf(x), BLACK);//p(x) 黑 p(p(x)) 红
               setColor(parentOf(parentOf(x)), RED);
               rotateRight(parentOf(parentOf(x)));//以p(p(x))为轴右旋
           }
       } else {//情况2-2
           Entry<K,V> y = leftOf(parentOf(parentOf(x)));
           if (colorOf(y) == RED) {//情况2-2-1 ...
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

<!-- Later -->

当插入节点为红色节点时则性质1， 3， 5永远成立，只有性质2：根节点黑色，性质4：红色节点不连续会被破坏。

待插入节点位节点Z，父亲节点p(Z)   

情况1. 如果Z为根节点，则原树为空，则性质2被破坏    
情况2. 如果p(Z)为红色，则性质4被破坏    

细分具体情况：
```   
情况1. Z为根节点，着色为红色   
情况2-1. p(Z)为红色，p(Z)是p(p(Z))的左孩子。   
  情况2-1-1. p(Z)为红色，p(Z)是p(p(Z))的左孩子，z的叔叔节点y为红色。    
  情况2-1-2. p(Z)为红色，p(Z)是p(p(Z))的左孩子，z的叔叔节点y为黑色而且z为左孩子。    
  情况2-1-3. p(Z)为红色，p(Z)是p(p(Z))的左孩子，z的叔叔节点y为黑色而且z为右孩子。  
情况2-2. p(Z)为红色，p(Z)是p(p(Z))的右孩子。   
  情况2-2-1. p(Z)为红色，p(Z)是p(p(Z))的右孩子，z的叔叔节点y为红色。    
  情况2-2-2. p(Z)为红色，p(Z)是p(p(Z))的右孩子，z的叔叔节点y为黑色而且z为左孩子。    
  情况2-2-3. p(Z)为红色，p(Z)是p(p(Z))的右孩子，z的叔叔节点y为黑色而且z为右孩子。  
```
具体情况具体调整：   
1. 情况1. 着色为黑色OK  
2.1. **情况2-1. p(Z)为红色，p(Z)是p(p(Z))的左孩子。**
2.1.1. ***情况2-1-1. p(Z)为红色，p(Z)是p(p(Z))的左孩子，z的叔叔节点y为红色。***   
**p(Z)，y 黑   p(p(Z)) 红**    
2.1.2. ***情况2-1-2. p(Z)为红色，p(Z)是p(p(Z))的左孩子，z的叔叔节点y为黑色而且z为左孩子。***      
**p(Z) 黑 p(p(Z)) 红 以p(p(Z))为轴右旋**     
2.1.3. ***情况2-1-3. p(Z)为红色，p(Z)是p(p(Z))的左孩子，z的叔叔节点y为黑色而且z为右孩子。***     
**以p(Z)为轴左旋,转换成情况2-1-2**   

![](insert.jpg "插入操作")
<!--  4节点(Z,p(Z),p(p(Z)),叔叔节点y)最终稳定形态 ①...②... -->

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
    if (p.left != null && p.right != null) {//当p左右孩子，节点p用后继节点代替
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
        if (p.color == BLACK)//删除黑色节点... ...
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
结构调整... ...

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

待删除节点y

1. 删除节点为红色(ok)


2. 删除节点为黑色(黑高-1，红色节点变连续)
2.1. y为根节点
2.2. p(y)与child(y)为红色... ...


---
<!-- 以下内容部分摘自《数据结构》刘大有... -->
# 补充
## 二叉查找树
中序遍历下获取数据的增序序列

### 前驱结点
>比当前节点小的最大节点

**右父节点树左父节点&lt;左子树&lt;当前节点&lt;右子树&lt;右父节点树&lt;右父节点树右子节点**    
**左父节点树左子节点&lt;左父节点树&lt;左子树&lt;当前节点&lt;右子树&lt;左父节点树右父节点**   

前驱结点：左子树，左父节点

<!-- 二叉查找树，左子树，左父树-->

1. 若左子树存在，左子树中最右
2. 若左子树不存在，左父节点“树”：最大，左父节点or右父节点树的左子节点... ...



### 后继节点
>比当前节点大的最小节点

**右父节点树右子节点&gt;右父节点树&gt;右子树&gt;当前节点&gt;左子树&gt;右父节点树左父节点**    
**左父节点树右父节点&gt;右子树&gt;当前节点&gt;左子树&gt;左父节点树&gt;左父节点树左子节点**

1. 若右子树存在，右子树最左
2. 若右子树不存在，右父节点“树”：最小，右父节点or左父节点树右父节点... ...



### 删除节点

<!-- 相对待删除节点q -->
#### 右子节点为空
左子节点直接替换ok   
![](http://cc.jlu.edu.cn/G2S/eWebEditor/uploadfile/20121217103146001.png)  

#### 右子节点的左子节点为空
左子节点作为右子节点的左子节点   
![](http://cc.jlu.edu.cn/G2S/eWebEditor/uploadfile/20121217103146002.png)![](http://cc.jlu.edu.cn/G2S/eWebEditor/uploadfile/20121217103146003.png)   

#### 右子节点的左子节点不空
最左左子节点（即待删除节点的后继节点）替换待删除节点
最左左子节点的右子节点替换最左左子节点的原位
![](http://cc.jlu.edu.cn/G2S/eWebEditor/uploadfile/20121217103147004.png)  

---
# 拓展
## 平衡二叉树
> 根据delta 平衡系数,调整高度      
> 每个节点自带平衡系数

### 高度平衡二叉树
平衡系数 = h(RightTree) - h(LeftTree)
```
+.- ： +1 0 -1
```

#### 左旋
#### 右旋


### 重量平衡二叉树

平衡系数β =
1. 树为空，1/2
2. 树不为空， 外节点数目（左子树）/总外节点数

相对于α的重量平衡二叉树   

任意子树平衡系数 = β（T*）    
```
α<=  β（T*）  < 1-α  
```


## 斐波那契树
![斐波那契树](fabonacci.jpg "斐波那契树")
