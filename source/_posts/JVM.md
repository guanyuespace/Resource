---
title: JVM
date: 2019-02-28 15:04:09
categories:
- Java
- JVM
tags:
- Java
- JVM
---
<!-- TOC -->

- [JVM](#jvm)
- [了解Java内存](#%E4%BA%86%E8%A7%A3java%E5%86%85%E5%AD%98)
- [虚拟机做了哪些工作?](#%E8%99%9A%E6%8B%9F%E6%9C%BA%E5%81%9A%E4%BA%86%E5%93%AA%E4%BA%9B%E5%B7%A5%E4%BD%9C)
  - [什么时候引发GC](#%E4%BB%80%E4%B9%88%E6%97%B6%E5%80%99%E5%BC%95%E5%8F%91gc)
  - [判断对象是否死亡: 可达性分析算法和引用计数法](#%E5%88%A4%E6%96%AD%E5%AF%B9%E8%B1%A1%E6%98%AF%E5%90%A6%E6%AD%BB%E4%BA%A1-%E5%8F%AF%E8%BE%BE%E6%80%A7%E5%88%86%E6%9E%90%E7%AE%97%E6%B3%95%E5%92%8C%E5%BC%95%E7%94%A8%E8%AE%A1%E6%95%B0%E6%B3%95)
  - [方法区的回收](#%E6%96%B9%E6%B3%95%E5%8C%BA%E7%9A%84%E5%9B%9E%E6%94%B6)
  - [垃圾收集算法](#%E5%9E%83%E5%9C%BE%E6%94%B6%E9%9B%86%E7%AE%97%E6%B3%95)
    - [算法分类:](#%E7%AE%97%E6%B3%95%E5%88%86%E7%B1%BB)
      - [标记-清除算法](#%E6%A0%87%E8%AE%B0-%E6%B8%85%E9%99%A4%E7%AE%97%E6%B3%95)
      - [复制算法](#%E5%A4%8D%E5%88%B6%E7%AE%97%E6%B3%95)
      - [标记-整理算法](#%E6%A0%87%E8%AE%B0-%E6%95%B4%E7%90%86%E7%AE%97%E6%B3%95)
      - [分代收集算法](#%E5%88%86%E4%BB%A3%E6%94%B6%E9%9B%86%E7%AE%97%E6%B3%95)
      - [分区收集:](#%E5%88%86%E5%8C%BA%E6%94%B6%E9%9B%86)
    - [垃圾收集器](#%E5%9E%83%E5%9C%BE%E6%94%B6%E9%9B%86%E5%99%A8)
    - [STOP THE WORLD](#stop-the-world)
  - [虚拟机类加载机制](#%E8%99%9A%E6%8B%9F%E6%9C%BA%E7%B1%BB%E5%8A%A0%E8%BD%BD%E6%9C%BA%E5%88%B6)
    - [Java类和方法如何加载](#java%E7%B1%BB%E5%92%8C%E6%96%B9%E6%B3%95%E5%A6%82%E4%BD%95%E5%8A%A0%E8%BD%BD)
      - [Class类文件结构](#class%E7%B1%BB%E6%96%87%E4%BB%B6%E7%BB%93%E6%9E%84)

<!-- /TOC -->
# JVM
>参考[Java虚拟机（JVM）](https://blog.csdn.net/qq_41701956/article/details/81664921)       
[java原理实现: 从虚拟机到源码](https://blog.csdn.net/secret_breathe/article/details/80886851)   
[深入理解java虚拟机](https://www.cnblogs.com/prayers/p/5515245.html)    

<!-- more -->

1. Java 内存区域与内存溢出异常  
  1.1 运行时数据区域   
  1.2 **HotSpot 虚拟机对象探秘**  
    - 对象的创建
    - 对象的内存布局
    >在 HotSpot 虚拟机中，分为 3 块区域：对象头(Header)、实例数据(Instance Data)和对齐填充(Padding)

    -  对象的访问定位
2. 垃圾回收器与内存分配策略  
3. **Java 内存模型与线程**
  3.1 Java内存模型



![对象的访问定位-通过句柄访问](https://user-gold-cdn.xitu.io/2017/9/4/ebf00ed26c35aefd93d5a3a36b3b1613 "对象的访问定位-通过句柄访问")    
![对象的访问定位-通过句柄访问](https://user-gold-cdn.xitu.io/2017/9/4/de6924b6e9d576105ba24700f1f357f4 "对象的访问定位-通过句柄访问")   
比较：使用句柄的最大好处是 `reference` 中存储的是稳定的句柄地址，在对象移动(GC)时只改变实例数据指针地址，`reference` 自身不需要修改。直接指针访问的最大好处是速度快，节省了一次指针定位的时间开销。如果是对象频繁 `GC` 那么句柄方法好，如果是对象频繁访问则直接指针访问好。

....

---
# 了解Java内存
​ Java内存是由虚拟机自动管理的, 当时也会发生内存溢出异常. 为了在发生内存溢出时不至于束手无策, 还是需要了解 Java 怎么管理内存的.

1. Java内存划分  
​ Java内存划分按照各自的用途以及创建和销毁的时间. 有的区域随着虚拟机进程的启动而存在, 有的依赖用户线程的启动和结束而建立和销毁.

2. 跟随线程的区域
  - **程序计数器**  
  ​程序计数器可以看做是当前线程执行的字节码的行号指示器. 它是**通过改变计数器的值来选择下一条需要执行的字节码指令**. 是程序能够按照逻辑执行各种操作的关键.   
  ​每个线程都有自己的程序计数器, 同一时刻一个处理器只会执行一个独立的程序计数器. 各条线程之间的计数器互不影响, 独立存储. 可以看做是 "线程私有" 的内存.
  - **Java虚拟机栈**    
  ​每个线程拥有独立的Java虚拟机栈. 它的生命周期与线程相同.       
  虚拟机栈描述的是Java方法执行的内存模型. _每个方法在执行的时候都会创建一个栈帧( Stack Frame 栈框架), 存储了表: 局部变量表 (详见书籍8.2章), 操作数栈(记录操作次数), 动态链接(指向运行时常量池, 取出该栈帧所属方法的部分常量), 方法出口等信息_. 在虚拟机运行的时候用到.    
  - **局部变量表**   
  存放了在编译期可知的各种基本数据类型(虚拟机在编译代码时就确定了方法变量的数量和类型等), 包括**8种基本数据类型**和**对象引用**( Reference 类型, 可能是一个指针,指向对象起始地址, 或是指向一个代表对象的句柄或其他与此对象相关的地址位置) 和** returnAddress 类型**(指向一条字节码指令的地址).   
  ​64位的 long 和 double 类型的数据占用了2个局部变量空间(Slot, 可看作空间单位), 其他类型占1个Slot     
  - **所需的空间在编译期分配完成**   
  ***当进入一个方法时, 这个方法需要在帧中分配多大的局部变量空间是完全确定的. 运行期间也不会改变. 因为这里 跟方法是否执行无关, 仅记录了方法的特征. 每当方法需要的时候就从局部变量表中读取.***   
  编译期发生了哪些事(后面会有详细解释): Java虚拟机按照规范, 把代码按照固定的 Class 文件格式解析成二进制文件. Class文件中描述了类, 方法, 属性, 接口等各种信息. 需要提前加载的常量会提前加载, 整个过程包含了验证, 分配内存等操作.      
  栈深: 如果创建时线程请求的栈深度大于虚拟机允许的深度, 将抛出 StackOverflowError 异常; 大部分虚拟机允许动态扩展, 扩展时如果无法申请到足够的内存, 就会抛出 OutOfMemoryError 异常.     
  - **本地方法栈**     
  与虚拟机栈非常的相似, 是虚拟机的一部分. 只不过是为虚拟机的Native方法服务. 虚拟机栈是为了执行Java方法(已翻译成字节码)服务.   
  同样会抛出 StackOverflowError 和 OutOfMemoryError 异常.   
  HotSpot直接把本地方法栈和虚拟机栈合二为一.  


3. 随虚拟机启动的区域    

  - **Java堆**     
  只用来存放对象实例. 因此一般是所有内存中最大的一块. 它被所有线程共享. 几乎所有的对象实例都要在这里分配内存. 垃圾收集器也主要管理这一块. 因此也成 "GC堆" (Garbage Collected Heap). 还可细分为新生代和老年代. 具体在后面解释   
  - **内存分配**   
  优化内存分配的目的是为了更好的回收内存, 或者更快的分配内存.     
  Java堆可以是不连续的, 容量对大部分虚拟机来说都是按照可动态扩展来实现. 当堆中没有足够内存完成实例分配, 并且也无法再扩展时, 将会抛出 OutOfMemoryError 异常.   
  - **方法区(非堆)**  
  _用于存储已被虚拟机加载的类的信息, 常量, 静态变量, 即时编译器编译后的代码等数据._ 虽然Java虚拟机规范描述为属于堆的一个逻辑部分, 但是有个别名叫 Non-Heap (非堆).   
  在HotSpot中, 使用永久代的GC方法管理方法区. 实际并不等价于永久代. 而且较其他虚拟机更容易产生内存溢出问题.     
  _这个区域的垃圾回收主要针对常量池的回收和类型的卸载. 回收效果难以令人满意, 尤其是类型卸载的条件相当苛刻._   
  - **运行时常量池**     
  **方法区的一部分.** 编译成的Class文件中有一项信息就是常量池 (Constant Pool Table). 用于存放编译期生成的各种字面量和符号引用. 这部分内容在类加载后会进入方法区的运行时常量池中存放.   
  Java并不要求常量一定只有编译期才能产生(Class文件常量池的内容), **运行期间也可能将新的常量放入常量池中. 比如 Java 代码中字符串的连接.**        
  当常量池无法再申请到内存时会抛出 OutOfMemoryError 异常.     


4. 直接内存  

  **直接内存 (Direct Memory) 不属于虚拟机运行时数据区的一部分**, 也不是Java虚拟机规范中定义的内存区域. 但是也被频繁使用, 同样会导致 OutOfMemoryError 异常.     
  JDK1.4加入了NIO (New Input/Output) 操作, 引入了一种基于通道 (Channel) 与缓冲区 (Buffer) 的 IO方式. NIO操作需要用到Direct Memory内存, 通过存储在Java堆中的DirectByteBuffer对象作为这块内存的引用进行操作. 避免了在 Java堆 和 Native堆中来回复制数据.     
  因为用到内存, 必然受本机内存影响. 如果把内存都分配给虚拟机, 用到直接内存时就会抛出 OutOfMemoryError 异常. 尤其是大量使用 NIO的程序.    

# 虚拟机做了哪些工作?   
- **编译**   
虚拟机编译器把写好的代码编译为存储字节码的Class文件.     
​虚拟机不和任何语言绑定, 它只与"Class" 文件 <一种特定的二进制文件格式> 关联. Class文件中包含Java虚拟机的指令集和符号表以及其他辅助信息. 代码中的各种变量,关键字,运算符的语义最终都是由多条字节码命令组合而成. 所以一些Java本身无法有效支持的语言特性不代表字节码本身无法有效支持.  
​`释疑: 什么是Java编译, 编译期, 编译时都做了哪些事情?`

- **执行**  
​虚拟机根据Class文件执行各种指令, 完成所有操作.

- **new 一个对象**  
分配内存, 通过指针碰撞, 或者空闲列表. 考虑到线程安全,    
>***一种是对分配内存空间的操作进行同步处理--实际上虚拟机采用CAS(Compare And Swap 比较并操作)和失败重试的方式保证更新的原子性,***     
>***另一种是每个线程在java堆中预先分配一小块内存空间, 称为本地内存分配缓冲(Thread Local Allocation Buffer, TLAB). 某个线程需要内存, 就从所属的TLAB上分配. 只有TLAB用完, 并分配新的TLAB时, 才需要同步锁定.***    

 分配完成后, 虚拟机要将分配到的内存空间都初始化为零值(不包括对象头. 如果使用TLAB, 则在TLAB分配时进行). 这一步操作保证了对象的实例字段在java代码中不需要赋初值就直接使用. 程序能访问这些字段的数据类型所对应的零值.     
​ 接下来, 虚拟机要对对象进行必要的设置, 这些信息存放在对象头(Object Header)之中. 根据虚拟机当前的状态不同, 对象头会有不同的设置方式.    
​ 此时, 对象的创建对虚拟机来说已经完成了. 但是对程序来说所有的字段都还为零, 还要执行<init>方法. 一般来说, 执行new指令之后都会紧接着执行<init>方法(详见.class文件的方法表), 把对象初始化, 这时一个真正的对象才被new出来.    
- **内存管理**  
虚拟机在运行的同时进行内存管理. 当内存已满, 就会进行GC(Garbage Collection).    
- **虚拟机如何管理内存**  
java内存运行时, 程序计数器, 虚拟机栈, 本地方法栈3个区域随线程而生, 随线程而灭. 每一个栈帧中分配多少内存基本上是在类结构确定下来的时候就是已知的. 因此这几个区域的内存分配和回收都具备确定性. 随着方法结束或者线程结束时, 内存自认就跟着回收了. 而Java堆和方法区则不一样. 一个接口的多个实现类需要的内存可能不一样, 一个方法的多个分支需要的内存也不一样. 只有在程序运行期间才能知道会创建哪些对象. 这部分内存的分配和回收都是动态的. GC也主要关注这部分内存.   

## 什么时候引发GC
新生代(Eden)没有足够的空间分配, 虚拟机将发起一次Minor GC(新生代GC, 特性: 非常频繁, 速度快).       
大对象(典型: 很长的字符串,数组)的问题: 经常出现大对象容易导致内存还有不少空间时就提前触发GC以获取足够的连续空间. 可以设置大于某个值的对象直接进入老年代, 避免在Eden区和两个Survivor区来回的复制(复制算法).      
晋升老年代: 活过多次GC, 或相同年龄的所有对象占用空间很大, 大于等于这个年龄的对象直接进入老年代, 或者大对象.   
- 空间分配担保      
在发起Minor GC之前, 虚拟机会先检查老年代的最大可用的连续空间是否大于新生代的所有对象总空间. 如果成立, 则Minor GC是安全的, 不成立, 则带有风险. 如果允许担保失败, 就会进行尝试一次Minor GC, 否则改为Full GC(老年代GC 比Minor GC慢10倍以上).
- 分配担保: 新生代内存已满, 需要老年代进行分配担保, 把之前每一次回收晋升到老年代对象容量的平均大小值作为经验值, 与老年代的剩余空间进行比较, 决定是否进行Full GC来让老年代腾出更多空间. 如果担保失败, 只能在失败后发起一次Full GC.


## 判断对象是否死亡: 可达性分析算法和引用计数法
1. 引用计数法   
原理: 给对象添加一个引用计数器, 每当有一个地方引用它时, 计数器加一, 当引用失效时, 引用就减一. 任何时刻计数器都为0的对象就是不可使用的.         
优点: **实现简单, 判定效率高.**   
缺点: **很难解决对象之间的循环引用的问题.(两个对象相互引用)**    
![两个对象相互引用](https://user-gold-cdn.xitu.io/2017/9/4/4c289a224cb4944e499fb5bfd33e592f "两个对象相互引用")    
从图中可以看出，如果不下小心直接把 Obj1-reference 和 Obj2-reference 置 null。则在 Java 堆当中的两块内存依然保持着互相引用无法回收。
2. 可达性分析算法    
原理: 通过一系列的成为"GC Roots"的对象作为起始点, 从这些节点开始向下搜索, 搜索所走过的路径称为引用链(Reference Chain), 当一个对象到GC Roots没有任何引用链(图论中称为从GC Roots到这个对象不可达)时, 证明此对象是不可达的.    
![](https://user-gold-cdn.xitu.io/2017/9/4/58bfac15ca6d3076def5174ed5ca5a99 "可达性分析法")        
可作为 GC Roots 的对象：   
  - 虚拟机栈(栈帧中的本地变量表)中引用的对象
  - 方法区中类静态属性引用的对象
  - 方法区中常量引用的对象
  - 本地方法栈中 JNI(即一般说的 Native 方法) 引用的对象    

 引用也分为强引用, 软引用, 弱引用, 虚引用, 用于内存充足时保留部分某些对象
 - 强引用  
 类似于 `Object obj = new Object();` 创建的，只要强引用在就不回收。    
 - 软引用  
 `SoftReference` 类实现软引用。在系统要发生内存溢出异常之前，将会把这些对象列进回收范围之中进行二次回收。   
 - 弱引用   
 `WeakReference` 类实现弱引用。对象只能生存到下一次垃圾收集之前。在垃圾收集器工作时，无论内存是否足够都会回收掉只被弱引用关联的对象。
 - 虚引用
 `PhantomReference` 类实现虚引用。无法通过虚引用获取一个对象的实例，为一个对象设置虚引用关联的唯一目的就是能在这个对象被收集器回收时收到一个系统通知。  


 3. 生存还是死亡     
 即使在可达性分析算法中不可达的对象，也并非是“facebook”(非死不可)的，这时候它们暂时出于“缓刑”阶段，一个对象的真正死亡至少要经历两次标记过程：如果对象在进行中可达性分析后发现没有与 GC Roots 相连接的引用链，那他将会被第一次标记并且进行一次筛选，筛选条件是此对象是否有必要执行 finalize() 方法。当对象没有覆盖 finalize() 方法，或者 finalize() 方法已经被虚拟机调用过，虚拟机将这两种情况都视为“没有必要执行”。       
 如果这个对象被判定为有必要执行 finalize() 方法，那么这个对象竟会放置在一个叫做 F-Queue 的队列中，并在稍后由一个由虚拟机自动建立的、低优先级的 Finalizer 线程去执行它。这里所谓的“执行”是指虚拟机会出发这个方法，并不承诺或等待他运行结束。finalize() 方法是对象逃脱死亡命运的最后一次机会，稍后 GC 将对 F-Queue 中的对象进行第二次小规模的标记，如果对象要在 finalize() 中成功拯救自己 —— 只要重新与引用链上的任何一个对象简历关联即可。        
 finalize() 方法只会被系统自动调用一次。

 4. finalize方法   
 **任何一个对象的finalize()方法都只会被系统自动调用一次，如果对象面临下一次回收，它的finalize()方法不会被再次执行，因此第二段代码的自救行动失败了**  

## 方法区的回收
方法区(或者HotSpot虚拟机中的永久代) 中进行垃圾回收性价比很低. 在堆中, 尤其是新生代, 常规一次GC可以回收75% ~ 95%的空间. 永久代的效率远低于此.       
永久代的GC主要回收: **废弃常量** 和 **无用的类**     

判断一个常量是否是废弃常量比较简单, 但是判定类是否无用就苛刻的多. 需要满足以下3个条件:
- 该类所有的实例对象都被回收   
- 加载该类的类加载器(ClassLoader) 已经被回收   
- 该类对应的**java.lang.Class对象没有在任何地方被引用**, 无法在任何地方通过反射访问该类的方法.在大量使用反射, 动态代理, CGLib等ByteCode框架, 动态生成JSP页面以及OSGi这类需要频繁自定义ClassLoader的场景都需要虚拟机具备类卸载的功能, 以保证永久代不会溢出

##  垃圾收集算法
### 算法分类:
#### 标记-清除算法
分为 "标记" 和 "清除" 两个阶段. 首先标记所有需要回收的对象, 在标记后统一回收.           
不足:   
  - 效率问题: 标记和清除的效率都不高    
  - 空间问题: 清除之后会产生大量不连续的内存碎片. 空间碎片太多可能会导致以后分配较大对象时没有足够的连续空间, 而不得不提前触发另一次GC.

#### 复制算法   
他将可用内存按照容量划分为大小相等的两块，每次只使用其中的一块。当这块的内存用完了，就将还存活着的对象复制到另外一块上面，然后再把已使用过的内存空间一次清理掉。这样使得每次都是对整个半区进行内存回收，内存分配时也就不用考虑内存碎片等复杂情况，只要移动堆顶指针，按顺序分配内存即可     
不足：将内存缩小为了原来的一半   
实际中我们并不需要按照1:1比例来划分内存空间，而是将内存分为一块较大的Eden空间和两块较小的Survivor空间，每次使用Eden和其中一块Survivor

当另一个Survivor空间没有足够空间存放上一次新生代收集下来的存活对象时，这些对象将直接通过分配担保机制进入老年代
> 新生代中, 98%的对象是朝生夕死的, 这个区域采用的算法是将存活的对象一次复制到另一块单独的内存中(称为Survivor), 之后清空其余内存. 再一次GC时, 同样复制到另一块Survivor中, 之后清理掉Eden和用过的Survivor. 考虑到存活的对象很少, 并且为了减少内存的浪费, HopSpot默认的内存分配比例是8 : 1 : 1, 这样只有10%的内存没有使用.            
 当Survivor内存不够用时, 需要依赖其他内存(指老年代)进行分配担保(Handle Promotion)    

#### 标记-整理算法
让所有存活的对象都向一端移动，然后直接清理掉端边界以外的内存
>​ 在老年代中, 在回收的对象存活率较高时就要进行较多的复制操作, 效率低. 更关键的是如果不想浪费大量的内存空间, 就要有额外的内存空间进行分配担保, 以应对内存中100%的对象都生存的极端情况. 因此一般采用标记-整理算法. 先进行标记, 之后把所有存活的对象往内存的一端移动, 最后清理边界外的空间.


#### 分代收集算法     
只是根据对象存活周期的不同将内存划分为几块。一般是把java堆分为新生代和老年代，这样就可以根据各个年代的特点采用最适当的收集算法。在新生代中，每次垃圾收集时都发现有大批对象死去，只有少量存活，那就选用复制算法，只需要付出少量存活对象的复制成本就可以完成收集。而老年代中因为对象存活率高、没有额外空间对它进行分配担保，就必须使用标记清理或者标记整理算法来进行回收       
> 分代收集算法为了在不同的内存区域采用不同的内存回收策略. 根据对象的生存周期, 把内存分为几块. 一般把Java堆分为新生代和老年代. 新生代中采用复制算法, 老年代中采用 标记-清理 或 标记-整理 算法.  

#### 分区收集:
将内存分为多个大小相等的独立区域(Region), 新生代和老年代都是一部分Region(不需要连续)的集合, 根据垃圾回收价值, 维护一个优先列表, 优先回收价值最大的区域(Garbage-First).          
程序对Reference类型的数据进行写操作时, 会检查引用的对象是否处于不同的Region之中, 是则把相关引用信息记录到被引用对象所属的Region的Remembered Set, GC根节点的枚举范围加入Remembered Set即可保证不会对全堆扫描也不会有遗漏的引用.

### 垃圾收集器
....

### STOP THE WORLD
....

## 虚拟机类加载机制  
参考[ClassLoader](/2019/02/15/ClassLoader/)    

.....

### Java类和方法如何加载
#### Class类文件结构
Class文件是一组以8字节为单位的二进制流. 各个数据项目严格按照顺序紧凑的排列在Class文件中, 中间没有任何分隔符.    
Class文件格式采用类似C语言结构的伪结构来存储数据. 这种伪结构只有两种数据类型: **无符号数**和**表**           
**无符号数**属于基本的数据类型. 以u1,u2,u4,u8分别代表1个字节, 2个字节, 4个字节,8个字节的无符号数.无符号数可以用来描述数字, 索引引用, 数量值或者按照UTF-8编码构成的字符串.      
**表**是由多个无符号数或其他表作为数据项构成的复合数据类型, 所有表都习惯的以"_info"结尾.表用于描述有层次关系的复合结构的数据. 整个Class文件其实就是一张表.      
#### Class文件怎么描述数量不定的数据   
由于Class文件没有任何分隔符号, 因此无论是顺序和数量, 甚至于数据存储的字节序都是被严格限定的. 每一个字节代表的含义, 长度多少, 先后顺序都不允许改变.    
无论是符号还是表, 当需要描述同一类型但数量不定的多个数据时, 经常会使用一个前置的容量计数器加若干个连续的数据项的形式. 这时称这一系列连续的某一类型的数据为某一类型的集合.        
#### Class文件的格式
1. 每个Class文件的头4个字节成为魔数, 0xCAFEBABE, 它的唯一作用是标识这是一个Class文件.
2. 紧跟着魔数的第5, 第6个字节是次版本号, 第7,8个字节是主版本号. 高版本的 JDK 能向下兼容低版本的版本的Class文件, 但不能运行以后版本的Class文件, 即使文件格式未发生任何变化.

.....

## 模块化热部署OSGi的类加载器结构
- 虚拟机字节码执行引擎   
- 什么是执行引擎   
- 运行时栈帧结构   
 栈帧 (Stack Frame) 是用于支持虚拟机进行方法调用和方法执行的数据结构. 它是虚拟机运行时数据区中的虚拟机栈(Virtual Machine Stack)的栈元素. 栈帧存储了方法的局部变量表, 操作数栈, 动态连接, 方法返回地址和一些额外的附加信息.      

 在编译程序代码的时候, 栈帧中需要多大的局部变量表, 需要多深的操作数栈就已经确定了. 并且写入到方法表的Code属性之中. 因此, 一个栈帧需要分配多少内存不会受到程序运行期变量数据的影响, 而仅仅取决于具体的虚拟机实现.     

 一个线程中的方法调用链可能很长, 很多方法都可能同时处于执行状态, 对于执行引擎来说, 在活动线程中, 只有位于栈顶的栈帧才是有效的, 称为当前栈帧 (Current Stack Frame), 与这个栈帧相关联的方法称为当前方法 (Current Method), 执行引擎的所有字节码指令都只针对当前栈帧进行操作.      
- 局部变量表   
 局部变量表(Local Variable) 是一组变量值存储空间. 用于存放方法参数和方法内部定义的局部变量.

# Java内存模型与线程

... ...


---
