---
title: 信号机制
date: 2019-04-30 10:28:16
---
<!-- TOC -->

- [Signal](#signal)
  - [信号本质](#%E4%BF%A1%E5%8F%B7%E6%9C%AC%E8%B4%A8)
  - [信号的分类](#%E4%BF%A1%E5%8F%B7%E7%9A%84%E5%88%86%E7%B1%BB)
    - [可靠信号与不可靠信号](#%E5%8F%AF%E9%9D%A0%E4%BF%A1%E5%8F%B7%E4%B8%8E%E4%B8%8D%E5%8F%AF%E9%9D%A0%E4%BF%A1%E5%8F%B7)
    - [实时信号与非实时信号](#%E5%AE%9E%E6%97%B6%E4%BF%A1%E5%8F%B7%E4%B8%8E%E9%9D%9E%E5%AE%9E%E6%97%B6%E4%BF%A1%E5%8F%B7)
  - [信号的处理流程](#%E4%BF%A1%E5%8F%B7%E7%9A%84%E5%A4%84%E7%90%86%E6%B5%81%E7%A8%8B)
    - [信号的诞生](#%E4%BF%A1%E5%8F%B7%E7%9A%84%E8%AF%9E%E7%94%9F)
    - [**信号在目标进程中注册**](#%E4%BF%A1%E5%8F%B7%E5%9C%A8%E7%9B%AE%E6%A0%87%E8%BF%9B%E7%A8%8B%E4%B8%AD%E6%B3%A8%E5%86%8C)
    - [信号的执行与注销](#%E4%BF%A1%E5%8F%B7%E7%9A%84%E6%89%A7%E8%A1%8C%E4%B8%8E%E6%B3%A8%E9%94%80)
  - [信号的装载](#%E4%BF%A1%E5%8F%B7%E7%9A%84%E8%A3%85%E8%BD%BD)
    - [signal](#signal)
    - [sigaction](#sigaction)
  - [信号的发送](#%E4%BF%A1%E5%8F%B7%E7%9A%84%E5%8F%91%E9%80%81)
    - [kill](#kill)
    - [sigqueue](#sigqueue)
    - [alarm](#alarm)
    - [abort](#abort)
    - [setitimer](#setitimer)
    - [raise](#raise)
  - [信号集及信号集操作函数：](#%E4%BF%A1%E5%8F%B7%E9%9B%86%E5%8F%8A%E4%BF%A1%E5%8F%B7%E9%9B%86%E6%93%8D%E4%BD%9C%E5%87%BD%E6%95%B0)
  - [信号阻塞与信号未决:](#%E4%BF%A1%E5%8F%B7%E9%98%BB%E5%A1%9E%E4%B8%8E%E4%BF%A1%E5%8F%B7%E6%9C%AA%E5%86%B3)
    - [Test](#test)

<!-- /TOC -->

# Signal
>参考:[Linux 信号（signal) 机制分析](https://www.cnblogs.com/subo_peng/p/5325326.html)

## 信号本质
软中断信号（signal，又简称为信号）用来通知进程发生了异步事件。在软件层次上是对中断机制的一种模拟， **在原理上，一个进程收到一个信号与处理器收到一个中断请求可以说是一样的。信号是进程间通信机制中唯一的异步通信机制.** 进程之间可以互相通过系统调用 kill 发送软中断信号。内核也可以因为内部事件而给进程发送信号，通知进程发生了某个事件。信号机制除了基本通知功能外，还可以传递附加信息。   

收到信号的进程对各种信号有不同的处理方法。处理方法可以分为三类：   
- 第一种是类似中断的处理程序，对于需要处理的信号，进程可以 *指定处理函数，* 由该函数来处理。
- 第二种方法是，*忽略某个信号，* 对该信号不做任何处理，就象未发生过一样。
- 第三种方法是，*对该信号的处理保留系统的默认值，* 这种缺省操作，对大部分的信号的缺省操作是使得进程终止。进程通过系统调用 signal 来指定进程对某个信号的处理行为。


## 信号的分类
可以从两个不同的分类角度对信号进行分类：   
- 可靠性方面：可靠信号与不可靠信号；  
- 与时间的关系上：实时信号与非实时信号。  


### 可靠信号与不可靠信号
Linux 信号机制基本上是从 Unix 系统中继承过来的。早期 Unix 系统中的信号机制比较简单和原始，信号值小于 SIGRTMIN 的信号都是不可靠信号。这就是 ***"不可靠信号"*** 的来源。 ***它的主要问题是信号可能丢失。***   
由于原来定义的信号已有许多应用，不好再做改动，最终只好又新增加了一些信号，并在一开始就把它们定义为可靠信号，这些信号 ***支持排队，不会丢失。***   
信号值位于 SIGRTMIN 和 SIGRTMAX 之间的信号都是可靠信号，可靠信号克服了信号可能丢失的问题。Linux 在 支持新版本的信号安装函数 sigation() 以及信号发送函数 sigqueue() 的同时，仍然支持早期的 signal() 信号安装函数，支持信号发送 函数 kill()。     
***信号的可靠与不可靠只与信号值有关，与信号的发送及安装函数无关。*** *目前 linux 中的 signal() 是通过 sigation() 函数实现的，因此，即使通过 signal() 安装的信号，在信号处理函数的结尾也不必再调用一次信号安装函数。* 同时，由 signal() 安装的实时信号支持排队，同样不会丢失。   
对于目前 linux 的两个信号安装函数：`signal()` 及 `sigaction()` 来说，它们都不能把 SIGRTMIN 以前的信号变成可靠信号（都不支持排队，仍有可能丢失，仍然是不可靠信号），而且对 SIGRTMIN 以后的信号都支持排队。 **这两个函数的最大区别在于，经过 sigaction 安装的信号都能传递信息给信号处理函数，而经过 signal 安装的信号不能向信号处理函数传递信息。对于信号发送函数来说也是一样的。**

### 实时信号与非实时信号
<!-- 历史遗留问题，Linux(Unix)信号机制前32种信号 -->
非实时信号都不支持排队，都是不可靠信号；实时信号都支持排队，都是可靠信号。

## 信号的处理流程
对于一个完整的信号生命周期 (从信号发送到相应的处理函数执行完毕) 来说，可以分为三个阶段：
- 信号诞生
- 信号在进程中注册
- 信号的执行和注销


### 信号的诞生
信号事件的发生有两个来源：硬件来源 (比如我们按下了键盘或者其它硬件故障)；软件来源，最常用发送信号的系统函数是 kill, raise, alarm 和 setitimer 以及 sigqueue 函数，软件来源还包括一些非法运算等操作。   
[【信号的相关介绍#3.1信号诞生】](https://www.cnblogs.com/subo_peng/p/5325326.html)... ...

### **信号在目标进程中注册**   
**在进程表的表项中有一个软中断信号域，该域中每一位对应一个信号。内核给一个进程发送软中断信号的方法，是在进程所在的进程表项的信号域设置对应于该信号的位。如果信号发送给一个正在睡眠的进程，如果进程睡眠在可被中断的优先级上，则唤醒进程；否则仅设置进程表中信号域相应的位，而不唤醒进程。如果发送给一个处于可运行状态的进程，则只置相应的位即可。**      
进程的 `task_struct` 结构中有关于本进程中未决信号的数据成员： struct sigpending pending：
```c
struct sigpending{
        struct sigqueue \*head, \*tail;
        sigset_t signal;

};
```
第三个成员是进程中所有未决信号集，第一、第二个成员分别指向一个 `sigqueue` 类型的结构链（称之为 "未决信号信息链"）的首尾，信息链中的每个 `sigqueue` 结构刻画一个特定信号所携带的信息，并指向下一个 `sigqueue` 结构:
```c
struct sigqueue{
        struct sigqueue \*next;
        siginfo_t info;
}
```
信号在进程中注册指的就是信号值加入到进程的未决信号集 `sigset_t signal`（每个信号占用一位）中，并且信号所携带的信息被保留到未决信号信息链的某个 `sigqueue` 结构中。只要信号在进程的未决信号集中，表明进程已经知道这些信号的存在，但还没来得及处理，或者该信号被进程阻塞。   

当一个实时信号发送给一个进程时，不管该信号是否已经在进程中注册，都会被再注册一次，因此，信号不会丢失，因此，实时信号又叫做 "可靠信号"。这意味着同一个实时信号可以在同一个进程的未决信号信息链中占有多个 `sigqueue` 结构（进程每收到一个实时信号，都会为它分配 一个结构来登记该信号信息，并把该结构添加在未决信号链尾，即所有诞生的实时信号都会在目标进程中注册）。   

当一个非实时信号发送给一个进程时，如果该信号已经在进程中注册（通过 `sigset_t signal` 指示），则该信号将被丢弃，造成信号丢失。因此，非实时信号又叫做 "不可靠信号"。这意味着同一个非实时信号在进程的未决信号信息链中，至多占有一个 `sigqueue` 结构。    

总之信号注册与否，与发送信号的函数（如 `kill()` 或 `sigqueue()` 等）以及信号安装函数（`signal()` 及 `sigaction()`）无关，只与信号值有关（信号值小于 SIGRTMIN 的信号最多只注册一次，信号值在 SIGRTMIN 及 SIGRTMAX 之间的信号，只要被进程接收到就被注册）     

### 信号的执行与注销
<!--操作系统：进程调度 -->
内核处理一个进程收到的软中断信号是在该进程的上下文中，因此，进程必须处于运行状态。当其由于被信号唤醒或者正常调度重新获得 CPU 时，在其从内核空间返回到用户空间时会检测是否有信号等待处理。如果存在未决信号等待处理且该信号没有被进程阻塞，则在运行相应的信号处理函数前，进程会把信号在未决信号链中占有的结构卸掉。

对于非实时信号来说，由于在未决信号信息链中最多只占用一个 `sigqueue` 结构，因此该结构被释放后，应该把信号在进程 未决信号集中删除（信号注销完毕）；而对于实时信号来说，可能在未决信号信息链中占用多个 `sigqueue` 结构，因此应该针对占用 `sigqueue` 结构的 数目区别对待：如果只占用一个 `sigqueue` 结构（进程只收到该信号一次），则执行完相应的处理函数后应该把信号在进程的未决信号集中删除（信号注销完 毕）。否则待该信号的所有 `sigqueue` 处理完毕后再在进程的未决信号集中删除该信号。   

当所有未被屏蔽的信号都处理完毕后，即可返回用户空间。对于被屏蔽的信号，当取消屏蔽后，在返回到用户空间时会再次执行上述检查处理的一套流程。   

内核处理一个进程收到的信号的时机是在一个进程从内核态返回用户态时。所以，当一个进程在内核态下运行时，软中断信号并不立即起作用，要等到将返回用户态时才处理。进程只有处理完信号才会返回用户态，进程在用户态下不会有未处理完的信号。   

处理信号有三种类型：进程接收到信号后退出；进程忽略该信号；进程收到信号后执行用户设定用系统调用 `signal` 的函数。当进程接收到一个它忽略的信号时，进程丢弃该信号，就象没有收到该信号似的继续运行。如果进程收到一个要捕捉的信号，那么进程从内核态返回用户态时执行用户定义的函数。而且执行用户定义的函数的方法很巧妙，内核是在用户栈上创建一个新的层，该层中将返回地址的值设置成用户定义的处理函数的地址，这样进程从内核返回弹出栈顶时就返回到用户定义的函数处，从函数返回再弹出栈顶时，才返回原先进入内核的地方。这样做的原因是用户定义的处理函数不能且不允许在内核态下执行（如果用户定义的函数在内核态下运行的话，用户就可以获得任何权限）。 <!-- meaning!!! -->   


## 信号的装载
### signal
### sigaction
## 信号的发送
### kill
>向任何进程或进程组发送任何信号。参数 pid 的值为信号的接收进程  
该调用执行成功时，返回值为 0；错误时，返回 - 1，并设置相应的错误代码 errno。  

```c
#include <sys/types.h>
#include <signal.h>
int kill(pid_t pid,int signo)
```

该系统调用可以用来向任何进程或进程组发送任何信号。参数 pid 的值为信号的接收进程     
- pid>0 进程 ID 为 pid 的进程  
- pid=0 同一个进程组的进程   
- pid<0 pid!=-1 进程组 ID 为 -pid 的所有进程    
- pid=-1 除发送进程自身外，所有进程 ID 大于 1 的进程  

Sinno 是信号值，**当为 0 时（即空信号），实际不发送任何信号，但照常进行错误检查，因此，可用于检查目标进程是否存在，以及当前进程是否具有向目标发送信号的权限**（root 权限的进程可以向任何进程发送信号，非 root 权限的进程只能向属于同一个 session 或者同一个用户的进程发送信号）。
### sigqueue
```c
#include <sys/types.h>
#include <signal.h>
int sigqueue(pid_t pid, int sig, const union sigval val)
```

### alarm
```c
#include <unistd.h>
unsigned int alarm(unsigned int seconds)
```

系统调用 alarm 安排内核为调用进程在指定的 seconds 秒后发出一个 SIGALRM 的信号。如果指定的参数 seconds 为 0，则不再发送 SIGALRM 信号。后一次设定将取消前一次的设定。该调用返回值为上次定时调用到发送之间剩余的时间，或者因为没有前一次定时调用而返回 0。   

注意，在使用时，alarm 只设定为发送一次信号，如果要多次发送，就要多次使用 alarm 调用。
### abort
```c
#include <stdlib.h>
void abort(void);
```
向进程发送 SIGABORT 信号，默认情况下进程会异常退出，当然可定义自己的信号处理函数。即使 SIGABORT 被进程设置为阻塞信号，调用 abort() 后，SIGABORT 仍然能被进程接收。该函数无返回值。
### setitimer   
....

### raise
```c
#include <signal.h>
int raise(int signo)
```
向进程本身发送信号，参数为即将发送的信号值。调用成功返回 0；否则，返回 -1。

## 信号集及信号集操作函数：

信号集被定义为一种数据类型：
```c
typedef struct {
       unsigned long sig[\_NSIG\_WORDS];
} sigset\_t
```

信号集用来描述信号的集合，每个信号占用一位。Linux 所支持的所有信号可以全部或部分的出现在信号集中，主要与信号阻塞相关函数配合使用。下面是为信号集操作定义的相关函数：  

```c
#include <signal.h>
int sigemptyset(sigset_t \*set)；
int sigfillset(sigset_t \*set)；
int sigaddset(sigset_t \*set, int signum)
int sigdelset(sigset_t \*set, int signum)；
int sigismember(const sigset_t \*set, int signum)；
```

sigemptyset(sigset_t \*set) 初始化由 set 指定的信号集，信号集里面的所有信号被清空；

sigfillset(sigset_t \*set) 调用该函数后，set 指向的信号集中将包含 linux 支持的 64 种信号；

sigaddset(sigset_t \*set, int signum) 在 set 指向的信号集中加入 signum 信号；

sigdelset(sigset_t \*set, int signum) 在 set 指向的信号集中删除 signum 信号；

sigismember(const sigset_t \*set, int signum) 判定信号 signum 是否在 set 指向的信号集中。

## 信号阻塞与信号未决:

每个进程都有一个用来描述哪些 **信号递送到进程时将被阻塞的信号集，该信号集中的所有信号在递送到进程后都将被阻塞。** 下面是与信号阻塞相关的几个函数：
```c
#include <signal.h>
int  sigprocmask(int  how,  const  sigset_t \*set, sigset_t \*oldset))；
int sigpending(sigset_t \*set));
int sigsuspend(const sigset_t \*mask))；
```
sigprocmask() 函数能够根据参数 `how` 来实现对信号集的操作，操作主要有三种：

- SIG_BLOCK 在进程当前阻塞信号集中添加 set 指向信号集中的信号
- SIG_UNBLOCK 如果进程阻塞信号集中包含 set 指向信号集中的信号，则解除对该信号的阻塞
- SIG_SETMASK 更新进程阻塞信号集为 set 指向的信号集

sigpending(sigset_t \*set)) 获得当前已递送到进程，却被阻塞的所有信号，在 set 指向的信号集中返回结果。

sigsuspend(const sigset_t \*mask)) 用于在接收到某个信号之前, 临时用 mask 替换进程的信号掩码, 并暂停进程执行，直到收到信号为止。sigsuspend 返回后将恢复调用之前的信号掩码。信号处理函数完成后，进程将继续执行。该系统调用始终返回 - 1，并将 errno 设置为 EINTR。   

### Test
```c
#include "signal.h"
#include "unistd.h"

static void my_op(int);
int main()
{
        sigset_t new_mask,old_mask,pending_mask;
        struct sigaction act;
        sigemptyset(&act.sa_mask);//初始化清空
        act.sa_flags=SA_SIGINFO;//
        act.sa_sigaction=(void*)my_op;//处理

        if(sigaction(SIGRTMIN+10,&act,NULL))
                printf("install signal SIGRTMIN+10 error\n");

        sigemptyset(&new_mask);//
        sigaddset(&new_mask,SIGRTMIN+10);//ok，添加信号

        if(sigprocmask(SIG_BLOCK, &new_mask,&old_mask))//阻塞该信号
                printf("block signal SIGRTMIN+10 error\n");

        sleep(10);
        printf("now begin to get pending mask and unblock SIGRTMIN+10\n");

        if(sigpending(&pending_mask)<0)//获取被阻塞的信号
                printf("get pending mask error\n");

        if(sigismember(&pending_mask,SIGRTMIN+10))//判断信号【SIGRTMIN+10】是否在获取的阻塞集中
                printf("signal SIGRTMIN+10 is pending\n");

        if(sigprocmask(SIG_SETMASK,&old_mask,NULL)<0)//更新阻塞集
                printf("unblock signal error\n");

        printf("signal unblocked\n");
        sleep(10);
}

static void my_op(int signum)
{
        printf("receive signal %d \n",signum);
}
```

编译该程序，并以后台方式运行。在另一终端向该进程发送信号 (运行 kill -s 42 pid，SIGRTMIN+10 为 42)，查看结果可以看出几个关键函数的运行机制，信号集相关操作比较简单。   ok...
