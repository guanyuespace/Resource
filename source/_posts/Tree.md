---
title: 数据结构-树
date: 2019-05-16 10:38:31
categories:
- DataStructure
- Tree
tags:
- DataStructure
- Tree
---
# 数据结构-树

## 红黑树
>参考[TreeMap](https://guanyuespace.github.io/2019/05/15/TreeMap)

### 定义
1. 每个节点要么是红色，要么是黑色。
2. 根节点必须是黑色
3. 红色节点不能连续（也即是，红色节点的孩子和父亲都不能是红色）。
4. 外界点必须为黑色，称之为nil节点
5. 从某一节点到达任意外界点所经过的黑色节点数目相同

### 高度
>满二叉树的性质

h &gt;= 2log<sub>2</sub>(N+1)  --《数据结构8.6 红黑树Page300》

<!-- more -->

## 平衡树

## 斐波那契判定树 <!-- 斐波那契查找 -->

---
### 斐波那契查找
在介绍斐波那契查找算法之前，我们先介绍一下很它紧密相连并且大家都熟知的一个概念——黄金分割。

黄金比例又称黄金分割，是指事物各部分间一定的数学比例关系，即将整体一分为二，较大部分与较小部分之比等于整体与较大部分之比，其比值约为1:0.618或1.618:1。

0.618被公认为最具有审美意义的比例数字，这个数值的作用不仅仅体现在诸如绘画、雕塑、音乐、建筑等艺术领域，而且在管理、工程设计等方面也有着不可忽视的作用。因此被称为黄金分割。

斐波那契序列的定义是： ![](http://cc.jlu.edu.cn/G2S/eWebEditor/uploadfile/20121217102641001.png)

大家记不记得斐波那契数列：1, 1, 2, 3, 5, 8, 13, 21, 34, 55, 89…….（从第三个数开始，后边每一个数都是前两个数的和）。然后我们会发现，随着斐波那契数列的递增，前后两个数的比值会越来越接近0.618，利用这个特性，我们就可以将黄金比例运用到查找技术中。<!-- 斐波那契数列又称为黄金分割数列-->


#### 基本思想
>也是二分查找的一种提升算法，通过运用黄金比例的概念在数列中选择查找点进行查找，提高查找效率。同样地，斐波那契查找也属于一种有序查找算法。

相对于折半查找，一般将待比较的key值与第mid=（low+high）/2位置的元素比较，比较结果分三种情况：
1. 相等，mid位置的元素即为所求
2. &gt;，low=mid+1;
3. &lt;，high=mid-1。

斐波那契查找与折半查找很相似，他是根据斐波那契序列的特点对有序表进行分割的。他要求开始表中记录的个数为某个斐波那契数小1，及n=F(k)-1;

开始将k值与第F(k-1)位置的记录进行比较(及mid=low+F(k-1)-1),比较结果也分为三种
1. 相等，mid位置的元素即为所求
2. &gt;，low=mid+1,**k-=2;**            
<font size="-1" color="red">说明：low=mid+1说明待查找的元素在[mid+1,high]范围内，k-=2 说明范围[mid+1,high]内的元素个数为n-(F(k-1))= Fk-1-F(k-1)=Fk-F(k-1)-1=F(k-2)-1个，所以可以递归的应用斐波那契查找。</font>
3. &lt;，high=mid-1,k-=1。
<font size="-1" color="red">说明：low=mid+1说明待查找的元素在[low,mid-1]范围内，k-=1 说明范围[low,mid-1]内的元素个数为F(k-1)-1个，所以可以递归 的应用斐波那契查找。</font>

复杂度分析：最坏情况下，时间复杂度为O(log2n)，且其期望复杂度也为O(log2n)。

---
#### 斐波那契（Fibonacci）查找算法思想

假设有一个长度为 f<sub>k</sub>-1 的文件，其记录被整序，可用记录下标表为 [1，f<sub>k</sub>-1]。  
记录 f<sub>k</sub>-1 将文件分为三个部分：   

a. 左子文件 [1，f<sub>k-1</sub>-1]；   
b. f<sub>k-1</sub>；    
c. 右子文件 [f<sub>k-1</sub>+1，f<sub>k</sub>-1]，   

其中，左、右子文件的大小分别为 f<sub>k-1</sub>-1，f<sub>k-2</sub>-1（由f<sub>k</sub>=f<sub>k-1</sub>+f<sub>k-2</sub> 故而 f<sub>k</sub>-1-f<sub>k-1</sub>=f<sub>k-2</sub>-1），故左、右子文件还可继续进行上面的划分（黄金分割）过程。


#### Fibonacci 查找的算法分析

**引理 8.4**  设m ≥ 3，T<sub>m</sub>是 m 阶 Fibonacci 树形，则 T<sub>m</sub>（内结点数为 f<sub>m+1</sub>-1）的左子树形的高度等于其右子树形的高度加 1，且 T<sub>m</sub>的高度为m - 1.

**引理 8.5**  设N = f<sub>m+1</sub>-1，则m 阶 Fibonacci 树形的高度约等于 1**.** 44 log <sub>2</sub> (N + 1) **.**

**定理 8.2**  令N = f<sub>m+</sub> <sub>1</sub> - 1，则算法 Fibonacci 在最坏情况下的时间复杂性阶为O(log<sub>2</sub> N)，且期望复杂性阶亦为O(log<sub>2</sub> N) **.**


```c++
const int MAX_SIZE = 20;
int a[] = { 1, 5, 15, 22, 25, 31, 39, 42, 47, 49, 59, 68, 88 , 167 };

int FibonacciSearch(int value)
{
	int F[MAX_SIZE];
	Fibonacci(F);//生成斐波那契序列
	int n = sizeof(a) / sizeof(int);//length

	int k = 0;
	while (n > F[k] - 1)
		k++;//第K项
  //F[K]-1: 0~F[k-1]-1   F[K-1]    F[k-1]+1,F[k]-1
  //        F[k-1]个     1         F[k-2]-1

  //temp：斐波那契查找树
	vector<int> temp;
	temp.assign(a, a + n);//拷贝
  //构造斐波那契查找树
	for (size_t i = n; i < F[k] - 1; i++)
		temp.push_back(a[n - 1]);//insert,补齐F[k]-1元素

	int l = 0, r = n - 1;
	while (l <= r)
	{
		int mid = l + F[k - 1] - 1;
    cout<<"k="<<k<<"\t currnet mid="<<mid<<"\t temp[mid]="<<temp[mid]<<endl;
		if (value > temp[mid]){
			l = mid + 1;
			k = k - 2;//右半部分
		}	else if (value < temp[mid]){
			r = mid - 1;
			k = k - 1;//左半部分
		}	else{
			if (mid < n)
				return mid;
			else
				return n - 1;//若mid>n-1则说明数据位于填充段，即返回序列n-1最后一元
		}
	}
	return -1;
}

int main()
{
  int n = sizeof(a) / sizeof(int);
  for(int i=0; i < n; i++)
      cout<< a[i]<<"\t";
  cout<<endl;

	int index = FibonacciSearch(22);
	cout << "index=" << index << "\t value=" << a[index]<<endl;
}
```

Result
```sh
maopengbo@3hw:~/code/C$ ./a.out
1       5       15      22      25      31      39      42      47      49      59      68      88      167
k=8      currnet mid=12  temp[mid]=88
k=7      currnet mid=7   temp[mid]=42
k=6      currnet mid=4   temp[mid]=25
k=5      currnet mid=2   temp[mid]=15
k=3      currnet mid=3   temp[mid]=22
index=3   value=22
```
