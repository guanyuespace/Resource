---
title: Select函数
date: 2019-04-27 19:51:23
---


# Select函数
[http://hi.baidu.com/%B1%D5%C4%BF%B3%C9%B7%F0/blog/item/e7284ef16bcec3c70a46e05e.html](http://hi.baidu.com/%B1%D5%C4%BF%B3%C9%B7%F0/blog/item/e7284ef16bcec3c70a46e05e.html)

select 函数用于在非阻塞中，当一个套接字或一组套接字有信号时通知你，系统提供 select 函数来实现多路复用输入 / 输出模型，原型：       

```c
#include <sys/time.h>        
#include <unistd.h>         
int select(int maxfd,fd_set *rdset,fd_set *wrset,fd_set *exset,struct timeval *timeout);
```

## 参数:
- maxfd 是需要监视的最大的文件描述符值 + 1；
- rdset,wrset,exset 分别对应于需要检测的可读文件描述符的集合，可写文件描述符的集 合及异常文件描述符的集合。
- struct timeval 结构用于描述一段时间长度，如果在这个时间内，需要监视的描述符没有事件发生则函数返回，返回值为 0。
- **fd_set** （它比较重要所以先介绍一下）是一组文件描述字 (fd) 的集合，它用一位来表示一个 fd（下面会仔细介绍），

**对于 `fd_set` 类型通过下面四个宏来操作：**
- FD_ZERO(fd_set *fdset);    
将指定的文件描述符集清空，在对文件描述符集合进行设置前，必须对其进行初始化，如果不清空，由于在系统分配内存空间后，通常并不作清空处理，所以结果是不可知的。    
- FD_SET(fd_set *fdset);   
用于在文件描述符集合中增加一个新的文件描述符。    
- FD_CLR(fd_set *fdset);    
用于在文件描述符集合中删除一个文件描述符。
- FD_ISSET(int fd,fd_set *fdset);   
用于测试指定的文件描述符是否在该集合中。

过去，一个 fd_set 通常只能包含 <32 的 fd（文件描述字），因为 fd_set 其实只用了一个 32 位矢量来表示 fd；现在, UNIX 系统通常会在头文件 < sys/select.h> 中定义常量 FD_SETSIZE，它是数据类型 fd_set 的描述字数量，其值通常是 1024，这样就能表示 < 1024 的 fd。根据 fd_set 的位矢量实现，我们可以重新理解操作 fd_set 的四个宏：

```c
fd_set set;    
FD_ZERO(&set);         
FD_SET(0, &set);       
FD_CLR(4, &set);         
FD_ISSET(5, &set);   
```

―――――――――――――――――――――――――――――――――――――――   
注意 `fd` 的最大值必须 < `FD_SETSIZE`。
―――――――――――――――――――――――――――――――――――――――

## select 函数的接口比较简单：

```c
int select(int nfds, fd_set *readset, fd_set *writeset,fd_set* exceptset, struct tim *timeout);
```

### 功能：
测试指定的 fd 可读？可写？有异常条件待处理？    
### 参数：
#### nfds   
需要检查的文件描述字个数（即检查到 fd_set 的第几位），数值应该比三组 fd_set 中所含的最大 fd 值更大，一般设为三组 fd_set 中所含的最大 fd 值加 1（如在 readset,writeset,exceptset 中所含最大的 fd 为 5，则 nfds=6，因为 fd 是从 0 开始的）。设这个值是为提高效率，使函数不必检查 fd_set 的所有 1024 位。
#### readset  
用来检查可读性的一组文件描述字。
#### writeset
用来检查可写性的一组文件描述字。
#### exceptset
用来检查是否有异常条件出现的文件描述字。(注：错误不包括在异常条件之内)
#### timeout
用于描述一段时间长度，如果在这个时间内，需要监视的描述符没有事件发生则函数返回，返回值为 0。   


**有三种可能：**      
1. timeout=NULL（阻塞：select 将一直被阻塞，直到某个文件描述符上发生了事件）
2. timeout 所指向的结构设为非零时间（等待固定时间：如果在指定的时间段里有事件发生或者时间耗尽，函数均返回）
3. **timeout 所指向的结构，时间设为 0（非阻塞：仅检测描述符集合的状态，然后立即返回，并不等待外部事件的发生）**

### 返回值：    
***返回对应位仍然为 1 的 fd 的总数。***

### Remarks：
三组 fd_set 均将某些 fd 位置 0，只有那些可读，可写以及有异常条件待处理的 fd 位仍然为 1。

举个例子，比如 recv(),   在没有数据到来调用它的时候, 你的线程将被阻塞, 如果数据一直不来, 你的线程就要阻塞很久. 这样显然不好. 所以采用 select 来查看套节字是否可读 (也就是是否有数据读了)    

**步骤如下：**    

```c
socket   s;   
.....   
fd_set   set;   
while(1){             
  FD_ZERO(&set);//将你的套节字集合清空         
  FD_SET(s,&set);//加入你感兴趣的套节字到集合,这里是一个读数据的套节字s         
  select(0,&set,NULL,NULL,NULL);//检查套节字是否可读,                                                           
  //很多情况下就是是否有数据(注意,只是说很多情况)                                                          
  //这里select是否出错没有写         
  if(FD_ISSET(s,   &set)   //检查s是否在这个集合里面,         
  {                                           
    //select将更新这个集合,把其中不可读的套节字去掉                                                     //只保留符合条件的套节字在这个集合里面                                       
    recv(s,...);         
  }         
  //do   something   here   
}
```

理解 `select` 模型的关键在于理解 `fd_set`, 为说明方便，取 `fd_set` 长度为 1 `字节，fd_set` 中的每一 `bit` 可以对应一个文件描述符 `fd`。则 1 字节长的 `fd_set` 最大可以对应 8 个 `fd`。

```
（1）执行 fd_set set; FD_ZERO(&set); 则 set 用位表示是 0000,0000。
（2）若 fd＝5, 执行 FD_SET(fd,&set); 后 set 变为 0001,0000(第 5 位置为 1)
（3）若再加入 fd＝2，fd=1, 则 set 变为 0001,0011
（4）执行 select(6,&set,0,0,0) 阻塞等待
（5）若 fd=1,fd=2 上都发生可读事件，则 select 返回，此时 set 变为 0000,0011。注意：没有事件发生的 fd=5 被清空。
```

基于上面的讨论，可以轻松得出 select 模型的特点：

1. 可监控的文件描述符个数取决与 sizeof(fd_set) 的值。  
我这边服务 器上 sizeof(fd_set)＝512，每 bit 表示一个文件描述符，则我服务器上支持的最大文件描述符是 512*8=4096。   
据说可调，另有说虽然可调，但调整上限受于编译内核时的变量值。   
本人对调整 fd_set 的大小不太感兴趣，参考 [技术系列之 网络模型（二）](http://www.cppblog.com/CppExplore/archive/2008/03/21/45061.html)  中的模型 2（1）可以有效突破 select 可监控的文件描述符上限。   
2. 将 fd 加入 select 监控集的同时，还要再使用一个数据结构 `array` 保存放到 select 监控集中的 fd.    
一是用于再 `select` 返回后，`array` 作为源数据和 `fd_set` 进行 `FD_ISSET` 判断。   
二是 `select` 返回后会把以前加入的但并无事件发生的 `fd` 清空，则每次开始 `select` 前都要重新从 `array` 取得 `fd` 逐一加入（`FD_ZERO` 最先），扫描 `array` 的同时取得 `fd` 最大值 `maxfd`，用于 `select` 的第一个 参数。   
3. 可见 `select` 模型必须在 `select` 前循环 `array`（加 `fd`，取 `maxfd`），`select` 返回后循环 `array`（`FD_ISSET` 判断是否有时间发生）。   

下面给一个伪码说明基本 select 模型的服务器模型：    
```c
array[slect_len];   
nSock=0;
array[nSock++]=listen_fd;
(之前listen port已绑定并listen)

maxfd=listen_fd;
while{   
  FD_ZERO(&set);   
  foreach (fd in array)    {       
    fd大于maxfd，则maxfd=fd       
    FD_SET(fd,&set)   
  }   

  res=select(maxfd+1,&set,0,0,0)；   
  if(FD_ISSET(listen_fd,&set))   {       
    newfd=accept(listen_fd);    //... ...   
    array[nsock++]=newfd;       // the fd that can be read       
    if(--res=0)    continue   
  }   
  foreach 下标1开始 (fd in array)    {       
    if(FD_ISSET(fd,&set))         
     执行读等相关操作          
     如果错误或者关闭，则要删除该fd，将array中相应位置和最后一个元素互换就好，nsock减一             
     if(--res=0) continue   
   }
}
```

使用 select 函数的过程一般是：    
先调用宏 FD_ZERO 将指定的 fd_set 清零，然后调用宏 FD_SET 将需要测试的 fd 加入 fd_set，接着调用函数 select 测试 fd_set 中的所有 fd，最后用宏 FD_ISSET 检查某个 fd 在函数 select 调用后，相应位是否仍然为 1。  

以下是一个测试单个文件描述字可读性的例子：

```c
int isready(int fd)     {         
  int rc;         
  fd_set fds;         
  struct tim tv;             
  FD_ZERO(&fds);         
  FD_SET(fd,&fds);         
  tv.tv_sec = tv.tv_usec = 0;             
  rc = select(fd+1, &fds, NULL, NULL, &tv);         
  if (rc < 0)   //error         return -1;             
  return FD_ISSET(fd,&fds) ? 1 : 0;     
}
```

下面还有一个复杂一些的应用：
// 这段代码将指定测试 Socket 的描述字的可读可写性，因为 Socket 使用的也是 fd

```c
uint32 SocketWait(TSocket *s,bool rd,bool wr,uint32 timems)    {     
  fd_set rfds,wfds;
#ifdef _WIN32     
  TIM tv;
#else     
  struct tim tv;
#endif         
  FD_ZERO(&rfds);     
  FD_ZERO(&wfds);      
  if (rd)     //TRUE     
  FD_SET(*s,&rfds);   //添加要测试的描述字      
  if (wr)     //FALSE       
  FD_SET(*s,&wfds);      
  tv.tv_sec=timems/1000;     //second     
  tv.tv_usec=timems%1000;     //ms      
  for (;;) //如果errno==EINTR，反复测试缓冲区的可读性          switch(select((*s)+1,&rfds,&wfds,NULL,              (timems==TIME_INFINITE?NULL:&tv)))
  {
    //测试在规定的时间内套接口接收缓冲区中是否有数据可读      
    //0－－超时，-1－－出错         
    case 0:                  return 0;          
    case (-1):                 
      if (SocketError()==EINTR)                   
      break;                            
    return 0; //有错但不是EINTR           
    default:              
      if (FD_ISSET(*s,&rfds))
      //如果s是fds中的一员返回非0，否则返回0                   
      return 1;              
      if (FD_ISSET(*s,&wfds))                   
      return 2;              
      return 0;         
    };
  }
```



----

**阻塞监听写操作**

```c
// family： AF_INET   type: SOCK_STREAM   protocol: TCP
// 注意：1.type和protocol不可以随意组合，如SOCK_STREAM不可以跟IPPROTO_UDP组合。当第三个参数为0时，会自动选择第二个参数类型对应的默认协议。
m_sockClient = socket(family, type, 0);//create socket client ...
if(true)
{
    int optval=1;
    if(setsockopt(m_sockClient, SOL_SOCKET, SO_KEEPALIVE, (char *) &optval, sizeof(optval)))
    //set socket Option    OPTION:SO_KEEPALIVE=VALUE:1
    {
        ALOGE(LOG_TAG, "setsockopt error.%d\n", errno);
        closeSocket();
        return false;
    }
}

//... ...F_GETFL 取得文件描述符状态旗标，此旗标为open（）的参数flags
// but socket ??
flags = fcntl(m_sockClient, F_GETFL, 0);
if (flags < 0) {
    ALOGE(LOG_TAG, " get socket flags fail.%d\n", errno);
    closeSocket();
    return false;
}

//F_SETFL 设置文件描述符状态旗标，参数arg为新旗标，但只允许O_APPEND、O_NONBLOCK和O_ASYNC位的改变，
//其他位的改变将不受影响。
if (fcntl(m_sockClient, F_SETFL, flags | O_NONBLOCK) < 0) {
    ALOGE(LOG_TAG, " set socket O_NONBLOCK fail.%d\n", errno);
    closeSocket();
    return false;
}

int retries_count = 0;
int connect_success = 0;
while (retries_count < 3) {//远程连接pszServerIP --> result --> addr_in
    if(connect(m_sockClient, (sockaddr *)&addr_in, sizeof(addr_in)) == SOCKET_ERROR){
         if (errno == EINPROGRESS ||errno == EALREADY || errno == EWOULDBLOCK ||  errno == EISCONN || errno == EAGAIN) {
            retries_count++;
         }else{
            ALOGE(LOG_TAG, "connect error:%d:%s\n",errno, strerror(errno));
            break;
         }
    }else{
        connect_success = 1;
        break;
    }

    usleep(10000000); // 10s
}

if(connect_success == 1){
    struct timeval to;
    struct timeval *toptr = NULL;
    fd_set wfd;
    memset(&wfd,0,sizeof(wfd));
    int err;
    socklen_t errlen;

    FD_ZERO(&wfd);
    FD_SET(m_sockClient, &wfd);

    //select 阻塞,监听socket发生写操作时--->操作
    if (select(FD_SETSIZE, NULL, &wfd, NULL, toptr) == SOCKET_ERROR) {
        ALOGE(LOG_TAG, "select after connect error:%s",strerror(errno));
        closeSocket();
        return false;
    }

    //socket 描述符不在写操作符集
    if (!FD_ISSET(m_sockClient, &wfd)) {
        errno = ETIMEDOUT;
        ALOGE(LOG_TAG, "select after connect error ETIMEDOUT,%d\n", errno);
        closeSocket();
        return false;
    }
    err = 0;
    errlen = sizeof(err);
    if (getsockopt(m_sockClient, SOL_SOCKET, SO_ERROR, &err, &errlen) == SOCKET_ERROR) {
      //option:ERROR
        ALOGE(LOG_TAG, "getsockopt after connect error :%s",strerror(errno));
        closeSocket();
        return false;
    }

    if (err) {
        errno = err;
        ALOGE(LOG_TAG, "getsockopt after connect error :%s",strerror(errno));
        closeSocket();
        return false;
    }
}
```


**读操作状态**

```c
while(socket_status == RUNNING){
    char buf[1024] = {0} ;
    memset(buf, 0, 1024);
    fd_set ds_Read, ds_error;
    memset(&ds_Read, 0, sizeof(ds_Read));
    memset(&ds_error, 0, sizeof(ds_error));

    int status = 0;
    struct timeval timeout={TIMEOUT_INTERVAL,0};

    FD_ZERO(&ds_Read);
    FD_SET(m_sockClient,&ds_Read);

    //select： ok     非阻塞仅检查状态
    status = select((m_sockClient + 1),&ds_Read,NULL,NULL,&timeout);
    if(isSocketValid() == false){
        ALOGI(LOG_TAG, "receive_func: m_sockClient = %d\n", m_sockClient);
        break;
    }
    if (status < 0) {
        ALOGE(LOG_TAG, "receive_func: select socket status, %s, errno: %d\n",strerror(errno), errno);
        handle_disconnect();
        break;
    }else if (status == 0){
        if(ALLLOGS){
            ALOGI(LOG_TAG, "receive_func: select socket status not readable\n");
        }
        continue;//直到readable
    }

    if(ALLLOGS){
        ALOGI(LOG_TAG, "receive_func: readable fd returned:%d\n",status);
    }

    if(FD_ISSET(m_sockClient,&ds_Read) )
    {/////////ok
        int buf_len = recv(m_sockClient, buf,1024, 0) ;//read
        if(LOG_INFO){
            ALOGI(LOG_TAG, "receive buf_len = %d", buf_len);
        }

        if (buf_len > 0 )
        {
            bool result = true;
            if(cloudsink != NULL){
                result = cloudsink->received_cb(buf, buf_len);
            }

            if(result == false){
                if(ALLLOGS){
                    ALOGI(LOG_TAG, "receive_func: the creator do not need the socket any more %d\n",m_sockClient);
                }
                break;
            }
        }else if(buf_len == 0){
            ALOGE(LOG_TAG, "receive_func: recv, %s, errno: %d\n",strerror(errno), errno);
            handle_disconnect();
            break;
        }else{
            if (errno == EINTR){

            }else if(errno == EAGAIN){

            }else{
                ALOGE(LOG_TAG, "receive_func: recv < 0, %s, errno: %d\n",strerror(errno), errno);
                handle_disconnect();
                break;
            }
        }
    }else{
        ALOGE(LOG_TAG, "receive_func: FD_ISSET %d error %d\n",m_sockClient, errno);
    }
}
```
