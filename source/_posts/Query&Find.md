---
title: 查找算法
date: 2019-05-17 15:46:20
categories:
- DataStructure
- Find

tags:
- DataStructure
- Find

---
# 查找算法

## 二分查找

low, mid, high

数据：待查找数组array[N], 内节点N，外节点 N+1     

内路径和：I<sub>N</sub>  
外路径和：E<sub>N</sub>  


1--N  
1---- (1+N)/2  --- N

1--(..)---(1+N)/2-1 ...

**引理8.1**  如果算法B对N个记录的成功查找是等概率的，且不成功查找也是等概率的，则成功查找关键词的平均比较次数为SN =1+ I<sub>N</sub>/N，不成功查找关键词的平均比较次数为UN = E<sub>N</sub> / (N+1)，其中I<sub>N</sub>，E<sub>N</sub> 为T(1 , N) 的内、外。    

另外，由于：E<sub>N</sub> = I<sub>N</sub> + 2N<!-- 归纳演绎法, ok -->   
故而：`UN  = (SN-1) * N/(N+1)`

**引理8.2**  对半查找二叉判定树T(s , e) 的高度是 élog 2 (e – s + 2)ù  .

**引理8.3**  设T(1 , N) 是N个内结点的二叉查找判定树，不考虑外结点T(1 , N)之高度为k，T(1 , N)之外结点均属于k或k+1层.

**定理8.1**  在最坏情况下，算法B的关键词比较次数为élog 2 (N+1)ù，且其期望复杂性等于O(log 2 N) .


## 一致对半查找
>为了减少指针的使用量(low, mid, high) --&gt; 用两个指针取代 (i, δ)即: i-δ, i, i+δ   

i: N
δ: N/2

引入无穷小(或比待查找元素都小的元素) F<sub>0</sub>


## 斐波那契查找
>依照斐波那契数列进行查找

0, 1, 2, 3, 4, 5, 6, 7
0, 1, 1, 2, 3, 5, 8, 13

6阶斐波那契判定树f<sub>7</sub>-1=12

1---f<sub>7</sub>-1   
mid= f<sub>6</sub>=8

1---f<sub>6</sub>-1   
mid=f<sub>4</sub>

...

f<sub>6</sub>+1---f<sub>7</sub>-1  (元素数：f<sub>5</sub>-1)    
mid= f<sub>6</sub>+1 + (f<sub>4</sub>)-1= f<sub>6</sub>+f<sub>4</sub>

f<sub>6</sub>+1---f<sub>6</sub>+f<sub>4</sub>-1    
mid= f<sub>6</sub>+f<sub>3</sub>

f<sub>6</sub>+f<sub>4</sub>+1---f<sub>7</sub>-1   
mid= f<sub>6</sub>+f<sub>4</sub> + f<sub>2</sub>

f<sub>6</sub>+f<sub>4</sub>+1---f<sub>6</sub>+f<sub>4</sub>+f<sub>2</sub>-1   
mid=f<sub>6</sub>+f<sub>4</sub>+f<sub>1</sub> ... ...


f<sub>6</sub>+f<sub>4</sub>+f<sub>2</sub>+1---f<sub>7</sub>-1 (元素数：f<sub>1</sub>-1)  
mid= f<sub>6</sub>+f<sub>4</sub>+f<sub>2</sub>+f<sub>0</sub>



**引理8.4**  设m ≥ 3，Tm 是 m 阶Fibonacci树形，则Tm（内结点数为fm+1-1）的左子树形的高度等于其右子树形的高度加 1，且Tm 的高度为m - 1  .

**引理8.5**  设N = fm+1-1，则m阶Fibonacci树形的高度约等于1. 44 log 2 (N + 1) .

**定理8.2**  令N = fm+ 1 - 1，则算法Fibonacci在最坏情况下的时间复杂性阶为O(log2 N)，且期望复杂性阶亦为O(log2 N) .
