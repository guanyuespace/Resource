---
title: Test
date: 2019-02-01 11:31:46
categories:
- Android
- Log
- ANR
tags:
- android
- anr_log
---
# Android ANR分析

> 前言
> ==
>
> ANR即Application Not Responding，顾名思义就是应用程序无响应。在Android中，一般情况下，四大组件均是工作在主线程中的，Android中的Activity Manager和Window Manager会随时监控应用程序的响应情况，如果因为一些耗时操作（网络请求或者IO操作）造成主线程阻塞一定时间（例如造成5s内不能响应用户事件或者BroadcastReceiver的onReceive方法执行时间超过10s），那么系统就会显示ANR对话框提示用户对应的应用处于无响应状态。
>
> 虽然每个程序员都不想ANR发生在自己的头上，因此，你需要严格遵守Google提供的一系列建议（看[这里](http://developer.android.com/training/articles/perf-anr.html)），简单总结就是以下两点：
>
>     1. 不要让主线程干耗时的工作
>     2. 不要让其他线程阻塞主线程的执行
>     
>
> 因此，要尽量保证主线程执行工作干净利落，一个消息循环执行时间最好不超过100ms到200ms，对于一些脏活累活可以交给AsyncTask、HandlerThread、IntentService或者另外起的新线程来完成，这样应用程序就能够及时响应用户的操作而不会给用户带来卡顿的感觉。

<!-- more -->
## ANR分析

一般应用程序在发布之前最好对新增的功能通过Systrace+TraceView进行性能测试，这样能够及时发现程序当中的耗时操作，对于一些可能引起ANR的风险做到提前规避。如果你是一个完美主义者，你也可以使用StrictMode来发现并干掉在你主线程中存在的一些磁盘IO、网络操作或者数据库读写等耗时代码，但是我个人觉得要完全避免在主线程进行这些操作还是不太现实，即使做到了也可能会造成应用程序的代码结构比较糟糕。    
即使程序员在自己的环境中对代码进过了一系列的性能测试，到用户手中还是有中招的风险，毕竟再高效率的代码还是要时间来执行，而且不同手机性能差距还是很明显。如果你不幸中招，那么可以采用以下的方法来进行简单的分析。  

## 导出trace文件


如果ANR发生，对应的应用会收到SIGQUIT异常终止信号，dalvik虚拟机就会自动在/data/anr/目录下生成trace.txt文件，这个文件记录了在发生ANR时刻系统各个线程的执行状态，获取这个文件是不需要root权限的，因此首先需要做的就是通过adb pull命令将这个文件导出并等待分析。

## trace文件格式解析

导出trace文件后，可以看到类似于如下的文件内容：  
```java
----- pid 901 at 2015-11-28 14:38:34 -----
Cmd line: system_server

JNI: CheckJNI is off; workarounds are off; pins=6; globals=2154 (plus 409 weak)

DALVIK THREADS:
(mutexes: tll=0 tsl=0 tscl=0 ghl=0)

"main" prio=5 tid=1 NATIVE
  | group="main" sCount=1 dsCount=0 obj=0x415a4e88 self=0x414c48d8
  | sysTid=901 nice=-2 sched=0/0 cgrp=apps handle=1073926484
  | state=S schedstat=( 303590361913 618664734427 651535 ) utm=19466 stm=10893 core=0
  #00  pc 00021914  /system/lib/libc.so (epoll_wait+12)
  #01  pc 0001065f  /system/lib/libutils.so (android::Looper::pollInner(int)+98)
  #02  pc 00010889  /system/lib/libutils.so (android::Looper::pollOnce(int, int*, int*, void**)+92)
  #03  pc 0006b771  /system/lib/libandroid_runtime.so (android::NativeMessageQueue::pollOnce(_JNIEnv*, int)+22)
  #04  pc 0002034c  /system/lib/libdvm.so (dvmPlatformInvoke+112)
  #05  pc 00050fcf  /system/lib/libdvm.so (dvmCallJNIMethod(unsigned int const*, JValue*, Method const*, Thread*)+398)
  #06  pc 00000214  /dev/ashmem/dalvik-jit-code-cache (deleted)
  at android.os.MessageQueue.nativePollOnce(Native Method)
  at android.os.MessageQueue.next(MessageQueue.java:138)
  at android.os.Looper.loop(Looper.java:196)
  at com.android.server.ServerThread.initAndLoop(SystemServer.java:1174)
  at com.android.server.SystemServer.main(SystemServer.java:1271)
  at java.lang.reflect.Method.invokeNative(Native Method)
  at java.lang.reflect.Method.invoke(Method.java:515)
  at com.android.internal.os.ZygoteInit$MethodAndArgsCaller.run(ZygoteInit.java:878)
  at com.android.internal.os.ZygoteInit.main(ZygoteInit.java:694)
  at dalvik.system.NativeStart.main(Native Method)
```

首先需要了解这些参数表示的意义，我们挑其中关键的几个说明：  

- 第一二行
```java    
----- pid 901 at 2015-11-28 14:38:34 -----
Cmd line: system_server
```       
说明了发生ANR的进程id、时间和进程名称。

- 后面三行是线程的基本信息  
```
JNI: CheckJNI is off; workarounds are off; pins=6; globals=2154 (plus 409 weak)
DALVIK THREADS:
(mutexes: tll=0 tsl=0 tscl=0 ghl=0)
```
其中tll、tsl、tscl、ghl、hwl、hwll分别对应：thread list lock, thread suspend lock, thread suspend count lock, gc heap lock, heap worker lock和heap worker list lock。
- later ...
```java
main prio=5 tid=1 NATIVE
```
说明了线程名称、线程的优先级、线程锁id和线程状态。线程名称是启动线程的时候手动指明的，这里的main标识是主线程，是Android自动设定的一个线程名称，如果是自己手动创建的线程，一般会被命名成“Thread-xx”的格式，其中xx是线程id，它只增不减不会被复用；注意这其中的tid不是线程的id,它是一个在Java虚拟机中用来实现线程锁的变量，随着线程的增减，这个变量的值是可能被复用的；最后线程的状态还分为如下几种  
<table>
<thead>
<tr>
  <th>状态</th>
  <th align="center">值</th>
  <th>说明</th>
</tr>
</thead>
<tbody><tr>
  <td>THREAD_ZOMBIE</td>
  <td align="center">0</td>
  <td>TERMINATED</td>
</tr>
<tr>
  <td>THREAD_RUNNING</td>
  <td align="center">1</td>
  <td>RUNNABLE or running now</td>
</tr>
<tr>
  <td>THREAD_TIMED_WAIT</td>
  <td align="center">2</td>
  <td>TIMED_WAITING in Object.wait()</td>
</tr>
<tr>
  <td>THREAD_MONITOR</td>
  <td align="center">3</td>
  <td>BLOCKED on a monitor</td>
</tr>
<tr>
  <td>THREAD_INITIALIZING</td>
  <td align="center">5</td>
  <td>allocated not yet running</td>
</tr>
<tr>
  <td>THREAD_STARTING</td>
  <td align="center">6</td>
  <td>started not yet on thread list</td>
</tr>
<tr>
  <td>THREAD_NATIVE</td>
  <td align="center">7</td>
  <td>off in a JNI native method</td>
</tr>
<tr>
  <td>THREAD_VMWAIT</td>
  <td align="center">8</td>
  <td>waiting on a VM resource</td>
</tr>
<tr>
  <td>THREAD_SUSPENDED</td>
  <td align="center">9</td>
  <td>suspended usually by GC or debugger</td>
</tr>
</tbody>
</table>


<font color="red" size="+1">&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;特别说明一下MONITOR状态和SUSPEND状态，MONITOR状态一般是类的同步块或者同步方法造成的，SUSPENDED状态在debugger的时候会出现，可以用来区别是不是真的是用户正常操作跑出了ANR。</font><br/>

- 后面一行
```java
| group="main" sCount=1 dsCount=0 obj=0x415a4e88 self=0x414c48d8
```
group是线程组名称。sCount是此线程被挂起的次数，dsCount是线程被调试器挂起的次数，当一个进程被调试后，sCount会重置为0，调试完毕后sCount会根据是否被正常挂起增长，但是dsCount不会被重置为0，所以dsCount也可以用来判断这个线程是否被调试过。obj表示这个线程的Java对象的地址，self表示这个线程本身的地址。

- 此后是线程的调度信息
```java   
sysTid=901 nice=-2 sched=0/0 cgrp=apps handle=1073926484
```
sysTid是Linux下的内核线程id，nice是线程的调度优先级，sched分别标志了线程的调度策略和优先级，cgrp是调度属组，handle是线程的处理函数地址。

- 然后是线程当前上下文信息
```
state=S schedstat=( 303590361913 618664734427 651535 ) utm=19466 stm=10893 core=0
```
state是调度状态；schedstat从 /proc/\[pid\]/task/\[tid\]/schedstat读出，三个值分别表示线程在cpu上执行的时间、线程的等待时间和线程执行的时间片长度，有的android内核版本不支持这项信息，得到的三个值都是0；utm是线程用户态下使用的时间值(单位是jiffies）;stm是内核态下的调度时间值；core是最后执行这个线程的cpu核的序号。

- 最后就是这个线程的调用栈信息。

 通过分析trace文件得到ANR信息
--

---

通过上面分析，可以看到trace文件的头部就包含了很多与该线程相关的信息，但是并不是每个信息我们都必须弄懂，排查ANR的时候只需要找到其中关键的几个信息即可。一般可以通过以下几个简单的方法来判断。

## trace文件顶部的线程一般是ANR的元凶

这是一个简单的方法，但是大部分情况下都适用，可以通过这个方法来快速判断是否是自己的应用造成了本次ANR。说明以下，并不是trace文件包含的应用就一定是造成ANR的帮凶，应用出现在trace文件中，只能说明出现ANR的时候这个应用进程还活着，trace文件的顶部则是触发ANR的应用信息。因此，如果你的应用出现在了trace文件的顶部，那么很有可能是因为你的应用造成了ANR，否则是你的应用造成ANR的可能性不大，但是具体是不是还需要进一步分析。例如：
```java
----- pid 1182 at 2015-11-26 01:53:34 -----
Cmd line: system_server

JNI: CheckJNI is off; workarounds are off; pins=5; globals=2982 (plus 135 weak)

DALVIK THREADS:
(mutexes: tll=0 tsl=0 tscl=0 ghl=0)

"main" prio=5 tid=1 NATIVE
| group="main" sCount=1 dsCount=0 obj=0x420a0e58 self=0x4208f918
| sysTid=1182 nice=-2 sched=0/0 cgrp=apps handle=1074594132
| state=S schedstat=( 211672310629 149959255867 472114 ) utm=13047 stm=8120 core=1
#00  pc 000218b8  /system/lib/libc.so (epoll_wait+12)
...
at android.os.MessageQueue.nativePollOnce(Native Method)
at android.os.MessageQueue.next(MessageQueue.java:138)
at android.os.Looper.loop(Looper.java:123)
at com.android.server.ServerThread.initAndLoop(SystemServer.java:1213)
at com.android.server.SystemServer.main(SystemServer.java:1317)
at java.lang.reflect.Method.invokeNative(Native Method)
at java.lang.reflect.Method.invoke(Method.java:515)
at com.android.internal.os.ZygoteInit$MethodAndArgsCaller.run(ZygoteInit.java:793)
at com.android.internal.os.ZygoteInit.main(ZygoteInit.java:609)
at dalvik.system.NativeStart.main(Native Method)

...

----- end 1182 -----

----- pid 18927 at 2015-11-26 01:53:34 -----
Cmd line: com.android.example

JNI: CheckJNI is off; workarounds are off; pins=0; globals=465 (plus 984 weak)

DALVIK THREADS:
(mutexes: tll=0 tsl=0 tscl=0 ghl=0)

"main" prio=5 tid=1 NATIVE
| group="main" sCount=1 dsCount=0 obj=0x420a0e58 self=0x4208f918
| sysTid=18927 nice=-6 sched=0/0 cgrp=apps handle=1074594132
| state=S schedstat=( 7748840431407 1615931922290 9994018 ) utm=712375 stm=62509 core=1
#00  pc 00020704  /system/lib/libc.so (__ioctl+8)
#01  pc 0002cfa3  /system/lib/libc.so (ioctl+14)
#02  pc 0001d3ed  /system/lib/libbinder.so (android::IPCThreadState::talkWithDriver(bool)+140)
#03  pc 0001d8d7  /system/lib/libbinder.so (android::IPCThreadState::waitForResponse(android::Parcel*, int*)+42)
#04  pc 0001dadf  /system/lib/libbinder.so (android::IPCThreadState::transact(int, unsigned int, android::Parcel const&, android::Parcel*, unsigned int)+118)
#05  pc 00019791  /system/lib/libbinder.so (android::BpBinder::transact(unsigned int, android::Parcel const&, android::Parcel*, unsigned int)+30)
...
#23  pc 00000d80  /system/bin/app_process
at android.os.BinderProxy.transact(Native Method)
at android.app.IAlarmManager$Stub$Proxy.set(IAlarmManager.java:154)
at android.app.AlarmManager.setImpl(AlarmManager.java:369)
at android.app.AlarmManager.setRepeating(AlarmManager.java:258)
at android.os.Handler.handleCallback(Handler.java:733)
at android.os.Handler.dispatchMessage(Handler.java:95)
at android.os.Looper.loop(Looper.java:136)
at android.app.ActivityThread.main(ActivityThread.java:5072)
at java.lang.reflect.Method.invokeNative(Native Method)
at java.lang.reflect.Method.invoke(Method.java:515)
at com.android.internal.os.ZygoteInit$MethodAndArgsCaller.run(ZygoteInit.java:793)
at com.android.internal.os.ZygoteInit.main(ZygoteInit.java:609)
at dalvik.system.NativeStart.main(Native Method)
```
虽然应用com.android.example出现在了trace文件中，但是在ANR的时候它在通过IPCThread在进行进程间通信，而此次ANR发生于system_server获取用户事件的native方法里面，并不是我们的应用造成了ANR。又例如下面的trace文件顶部内容为：
```java
----- pid 13406 at 2015-11-27 11:46:14 -----
Cmd line: com.android.example

JNI: CheckJNI is off; workarounds are off; pins=0; globals=536 (plus 102 weak)

DALVIK THREADS:
(mutexes: tll=0 tsl=0 tscl=0 ghl=0)

"main" prio=5 tid=1 SUSPENDED
  | group="main" sCount=1 dsCount=0 obj=0x41795e58 self=0x416b58b0
  | sysTid=13406 nice=-6 sched=0/0 cgrp=apps handle=1074557268
  | state=S schedstat=( 2352435524847 736727917292 2633566 ) utm=213075 stm=22168 core=1
  at java.lang.String.<init>(String.java:~261)
  at java.util.zip.ZipEntry.<init>(ZipEntry.java:392)
  at java.util.zip.ZipFile.readCentralDir(ZipFile.java:414)
  at java.util.zip.ZipFile.<init>(ZipFile.java:151)
  at java.util.zip.ZipFile.<init>(ZipFile.java:123)
  at com.android.example.Utility.isValideFile(Utility.java:2700)
  at com.android.example.Test.getPath(Test.java:243)
  at com.android.example.Test$1.run(Test.java:531)
  at android.os.Handler.handleCallback(Handler.java:733)
  at android.os.Handler.dispatchMessage(Handler.java:95)
  at android.os.Looper.loop(Looper.java:136)
  at android.app.ActivityThread.main(ActivityThread.java:5050)
  at java.lang.reflect.Method.invokeNative(Native Method)
  at java.lang.reflect.Method.invoke(Method.java:515)
  at com.android.internal.os.ZygoteInit$MethodAndArgsCaller.run(ZygoteInit.java:807)
  at com.android.internal.os.ZygoteInit.main(ZygoteInit.java:623)
  at dalvik.system.NativeStart.main(Native Method)
  ...
```
这种情况说明ANR发生于com.android.example应用中，而且指明了ANR发生时代码的执行位置，这种情况十有八九就是我们应用程序的问题，之后就需要通过这个trace文件指明的路径来对代码进行排查。

### 注意死锁和等待

虽然说ANR一般情况是由于让主线程做了很多耗时的操作，但是死锁或者主线程等待也是ANR高发的原因，例如如下的trace:
```java
----- pid 9436 at 2015-11-28 21:30:41 -----
Cmd line: com.example.yxz.myapplication

JNI: CheckJNI is off; workarounds are off; pins=0; globals=277

DALVIK THREADS:
(mutexes: tll=0 tsl=0 tscl=0 ghl=0)

"main" prio=5 tid=1 MONITOR
  | group="main" sCount=1 dsCount=0 obj=0x415a4e88 self=0x414c48d8
  | sysTid=9436 nice=0 sched=0/0 cgrp=apps handle=1073926484
  | state=S schedstat=( 671264662 337280259 1005 ) utm=53 stm=14 core=0
  at com.example.yxz.myapplication.performancetest.WaitANR$InnerMonitorClass.TimeConsumeFunc(WaitANR.java:~48)
  - waiting to lock <0x447a5670>  held by tid=11 (Thread-14208)
  at com.example.yxz.myapplication.performancetest.WaitANR$2.run(WaitANR.java:32)
  at android.os.Handler.handleCallback(Handler.java:733)
  at android.os.Handler.dispatchMessage(Handler.java:95)
  at android.os.Looper.loop(Looper.java:212)
  at android.app.ActivityThread.main(ActivityThread.java:5135)
  at java.lang.reflect.Method.invokeNative(Native Method)
  at java.lang.reflect.Method.invoke(Method.java:515)
  at com.android.internal.os.ZygoteInit$MethodAndArgsCaller.run(ZygoteInit.java:878)
  at com.android.internal.os.ZygoteInit.main(ZygoteInit.java:694)
  at dalvik.system.NativeStart.main(Native Method)

....

"Thread-14208" prio=10 tid=11 TIMED_WAIT
  | group="main" sCount=1 dsCount=0 obj=0x447a4b98 self=0x78296bb8
  | sysTid=9955 nice=-8 sched=0/0 cgrp=apps handle=2015978016
  | state=S schedstat=( 946045 640869 1 ) utm=0 stm=0 core=2
  at java.lang.VMThread.sleep(Native Method)
  at java.lang.Thread.sleep(Thread.java:1013)
  at java.util.concurrent.TimeUnit.sleep(TimeUnit.java:331)
  at com.example.yxz.myapplication.performancetest.WaitANR$InnerMonitorClass.TimeConsumeFunc(WaitANR.java:48)
  at com.example.yxz.myapplication.performancetest.WaitANR$1.run(WaitANR.java:20)
  at java.lang.Thread.run(Thread.java:841)
```
从trace文件可以看出，发生ANR的主线程正处于monitor状态，也就是它在等待一个synchronized块或者方法，但是目前这个monitor正在被tid=11的线程持有，所以造成了主线程被阻塞，从而发生了ANR。死锁的分析也是类似，发生死锁的线程一般处于MONITOR状态或者WAIT状态，等待其他进程的锁或者monitor，而其他进程又在等待另外线程的锁或者monitor，一直这样依赖下去，直到形成一个环。  

结束
==

[原文：Android ANR分析](https://blog.csdn.net/yxz329130952/article/details/50087731)
参考：
===

[Android 信号处理面面观 之 trace 文件含义](http://blog.csdn.net/rambo2188/article/details/7017241)  
[How to read Dalvik SIGNQUIT output](http://elliotth.blogspot.sg/2012/08/how-to-read-dalvik-sigquit-output.html)
