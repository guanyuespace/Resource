---
title: 高精度定时器(posix_timer)
date: 2019-04-29 11:37:12
---

<!-- TOC -->

- [Timer](#timer)
  - [创建](#%E5%88%9B%E5%BB%BA)
    - [参数](#%E5%8F%82%E6%95%B0)
      - [sigevent](#sigevent)
  - [启动](#%E5%90%AF%E5%8A%A8)
  - [销毁](#%E9%94%80%E6%AF%81)
- [sigaction](#sigaction)
- [sigval](#sigval)
- [sigqueue](#sigqueue)
- [Test](#test)

<!-- /TOC -->

# Timer
>高精度定时器 posix_timer      
 **创建、初始化以及删除一个定时器的行动被分为三个不同的函数：`timer_create()`(创建定时器)、`timer_settime()`(初始化定时器) 以及 `timer_delete()`(销毁它)。**

## 创建
进程可以通过调用 timer_create() 创建特定的定时器，定时器是每个进程自己的，不是在 fork 时继承的。  

```c
int timer_create(clockid_t clock_id, struct sigevent *evp, timer_t *timerid)
```

### 参数
- clock_id  
clock_id 说明定时器是基于哪个时钟的
```
CLOCK_REALTIME :Systemwide realtime clock.
CLOCK_MONOTONIC:Represents monotonic time. Cannot be set. monotonic:单调的无变化的
CLOCK_PROCESS_CPUTIME_ID :High resolution per-process timer. 高分辨率
CLOCK_THREAD_CPUTIME_ID :Thread-specific timer.
CLOCK_REALTIME_HR :High resolution version of CLOCK_REALTIME.
CLOCK_MONOTONIC_HR :High resolution version of CLOCK_MONOTONIC.
```
- timerid   
装载的是被创建的定时器的 ID。该函数创建了定时器，并将他的 ID 放入 timerid 指向的位置中。  

- sigevent:evp  
参数 evp 指定了定时器到期要产生的异步通知。   
如果 evp 为 NULL，那么定时器到期会产生默认的信号，对 CLOCK_REALTIMER 来说，默认信号就是 SIGALRM。   

 如果要产生除默认信号之外的其它信号，程序必须将 evp-&gt;sigev_signo 设置为期望的信号码。

 struct sigevent 结构中的成员 evp-&gt;sigev_notify 说明了定时器到期时应该采取的行动。通常，这个成员的值为 SIGEV_SIGNAL, 这个值说明在定时器到期时，会产生一个信号。程序可以将成员 evp-&gt;sigev_notify 设为 SIGEV_NONE 来防止定时器到期时产生信号。

 如果几个定时器产生了同一个信号，处理程序可以用 evp-&gt;sigev_value 来区分是哪个定时器产生了信号。要实现这种功能，程序必须在为信号安装处理程序时，使用 struct sigaction 的成员 sa_flags 中的标志符 SA_SIGINFO。

#### sigevent

```c
struct sigevent {
  int sigev_notify; //notification type
  int sigev_signo; //signal number
  union sigval   sigev_value; //signal value
  void (\*sigev_notify_function)(union sigval);
  pthread_attr_t \*sigev_notify_attributes;
}

union sigval{
  int sival_int; //integer value
  void \*sival_ptr; //pointer value
}
```
通过将 evp->sigev_notify 设定为如下值来定制定时器到期后的行为：      
1. SIGEV_NONE：什么都不做，只提供通过 `timer_gettime` 和 `timer_getoverrun` 查询超时信息。

2. SIGEV_SIGNAL: 当定时器到期，内核会将 `sigev_signo` 所指定的信号传送给进程。在信号处理程序中，`si_value` 会被设定会 `sigev_value`。

3. SIGEV_THREAD: 当定时器到期，内核会 (在此进程内) 以 `sigev_notification_attributes` 为线程属性创建一个线程，并且让它执行 `sigev_notify_function`，传入 `sigev_value` 作为为一个参数。


## 启动
timer_create() 所创建的定时器并未启动。要将它关联到一个到期时间以及启动时钟周期，可以使用 timer_settime()。

```c
int timer_settime(timer_t timerid, int flags, const struct itimerspec *value, struct itimerspect *ovalue);

struct itimespec{
  struct timespec it_interval;
  struct timespec it_value;
}
```
如同 settimer()，it_value 用于指定当前的定时器到期时间。当定时器到期，it_value 的值会被更新成 it_interval 的值。如果 it_interval 的值为 0，则定时器不是一个时间间隔定时器，一旦 it_value 到期就会回到未启动状态。timespec 的结构提供了纳秒级分辨率：

```c
struct timespec{
  time_t tv_sec;
  long tv_nsec;
}
```
如果 flags 的值为 `TIMER_ABSTIME`，则 value 所指定的时间值会被解读成绝对值 (此值的默认的解读方式为相对于当前的时间)。这个经修改的行为可避免取得当前时间、计算“该时间” 与“所期望的未来时间”的相对差额以及启动定时器期间造成竞争条件。

如果 ovalue 的值不是 NULL，则之前的定时器到期时间会被存入其所提供的 itimerspec。如果定时器之前处在未启动状态，则此结构的成员全都会被设定成 0。

## 销毁
```c
int timer_delete (timer_t timerid)
```
一次成功的 timer_delete() 调用会销毁关联到 timerid 的定时器并且返回 0。执行失败时，此调用会返回 - 1 并将 errno 设定会 EINVAL，这个唯一的错误情况代表 timerid 不是一个有效的定时器。



# sigaction
>指定信号触发时的表现

```
 int sigaction(int signum,const struct sigaction *act ,struct sigaction *oldact);
```
<!-- signum：信号编号，kill -l 显示的信号所对应的编号 -->
函数说明 sigaction()会依参数signum指定的信号编号来设置该信号的处理函数。参数signum可以指定SIGKILL和SIGSTOP以外的所有信号。
```c
struct sigaction {
   void     (*sa_handler)(int);///////////处理函数，int参数：signum
   void     (*sa_sigaction)(int, siginfo_t *, void *);/////处理函数，带参
   sigset_t   sa_mask;
   int       sa_flags;/////////
   void     (*sa_restorer)(void);
};
```

**当注册信号捕捉函数，希望获取更多信号相关信息，不应使用sa_handler而应该使用sa_sigaction。但此时的sa_flags必须指定为SA_SIGINFO。** siginfo_t是一个成员十分丰富的结构体类型，可以携带各种与信号相关的数据。     

```c
siginfo_t {
     int      si_signo;     /* Signal number */
     int      si_errno;     /* An errno value */
     int      si_code;      /* Signal code */
     int      si_trapno;    /* Trap number that caused
                               hardware-generated signal
                               (unused on most architectures) */
     pid_t    si_pid;       /* Sending process ID */
     uid_t    si_uid;       /* Real user ID of sending process */
     int      si_status;    /* Exit value or signal */
     clock_t  si_utime;     /* User time consumed */
     clock_t  si_stime;     /* System time consumed */
     sigval_t si_value;     /* Signal value */其他Unix版本/////////传递的参数-sigval
     int      si_int;       /* POSIX.1b signal */posix
     void    *si_ptr;       /* POSIX.1b signal */posix
     int      si_overrun;   /* Timer overrun count;
                               POSIX.1b timers */
     int      si_timerid;   /* Timer ID; POSIX.1b timers */
     void    *si_addr;      /* Memory location which caused fault */
     long     si_band;      /* Band event (was int in
                               glibc 2.3.2 and earlier) */
     int      si_fd;        /* File descriptor */
     short    si_addr_lsb;  /* Least significant bit of address
                               (since Linux 2.6.32) */
     void    *si_call_addr; /* Address of system call instruction
                               (since Linux 3.5) */
     int      si_syscall;   /* Number of attempted system call
                               (since Linux 3.5) */
     unsigned int si_arch;  /* Architecture of attempted system call
                               (since Linux 3.5) \*/
}
```


# sigval
>传送的信号值

```c
sigval{
  int   sival_int;//
  void \*sival_pt//待传递的参数地址
}
```

向指定进程发送指定信号的同时，携带数据。 **但，如传地址，需注意，不同进程之间虚拟地址空间各自独立，将当前进程地址传递给另一进程没有实际意义。**


# sigqueue
>加入信号队列... ...

```c
sigqueue(pid_t pid, signal ,sigval)
```



# Test

```c
struct sigaction act;
memset(&act, 0, sizeof(act));
act.sa_handler = timeoutHandler;//处理函数
act.sa_flags = 0;//
sigemptyset(&act.sa_mask);//清空
if (sigaction(SIGUSR1, &act, NULL) == -1)///SIGUSER1信号发生时触发sigaction.sa_handler
{
    ALOGE(LOG_TAG, "fail to sigaction for timer");
    return;
}


struct itimerspec timeout_sec;
struct sigevent se;
memset(&se, 0, sizeof(struct sigevent));
se.sigev_signo = SIGUSR1;//
se.sigev_notify = SIGEV_SIGNAL;//当定时器到期，内核会将 sigev_signo 所指定的信号传送给进程。在信号处理程序中，si_value 会被设定会 sigev_value。

if( ret = timer_create(CLOCK_REALTIME, &se, timer) != 0 )//创建timer发生SIGUSER1信号，即触发sigaction
{
    ALOGD(LOG_TAG, "startTimer:%s:Error creating timer error:%d\n", __func__, ret);
    return;
}

memset(&timeout_sec, 0x0,sizeof(timeout_sec));
timeout_sec.it_value.tv_sec = sec;
timeout_sec.it_interval.tv_sec = interval_sec;
if( timer_settime(*timer, 0, &timeout_sec, NULL) != 0 )//启动
{
    ALOGD(LOG_TAG, "%s: startTimer:settimer failed\n",__func__ );
}


////do something....

////finally here ...
if(timer != NULL){
    timer_delete(timer);
    timer = NULL;
}
```
