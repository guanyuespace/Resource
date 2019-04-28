---
title: Socket使用
date: 2019-04-27 18:27:23
---

<!-- TOC -->

- [Socket整理](#socket%E6%95%B4%E7%90%86)
  - [数据结构](#%E6%95%B0%E6%8D%AE%E7%BB%93%E6%9E%84)
    - [地址相关](#%E5%9C%B0%E5%9D%80%E7%9B%B8%E5%85%B3)
      - [sockaddr](#sockaddr)
      - [**sockaddr_in**](#sockaddr_in)
  - [socket创建](#socket%E5%88%9B%E5%BB%BA)
    - [参数说明　　](#%E5%8F%82%E6%95%B0%E8%AF%B4%E6%98%8E)
  - [基本操作](#%E5%9F%BA%E6%9C%AC%E6%93%8D%E4%BD%9C)
  - [Other](#other)
    - [操作路由表](#%E6%93%8D%E4%BD%9C%E8%B7%AF%E7%94%B1%E8%A1%A8)
    - [字节序](#%E5%AD%97%E8%8A%82%E5%BA%8F)
      - [相关函数](#%E7%9B%B8%E5%85%B3%E5%87%BD%E6%95%B0)
- [add_ons](#add_ons)
  - [**sockaddr_un**](#sockaddr_un)
    - [创建socket](#%E5%88%9B%E5%BB%BAsocket)
    - [绑定](#%E7%BB%91%E5%AE%9A)
    - [监听](#%E7%9B%91%E5%90%AC)
    - [连接](#%E8%BF%9E%E6%8E%A5)

<!-- /TOC -->


# Socket整理
>Socket网络编程

## 数据结构
### 地址相关
<!--类型：字节数   char:1 short:2 long:4 -->

#### sockaddr
struct sockaddr是一个通用地址结构，这是为了统一地址结构的表示方法，统一接口函数，使不同的地址结构可以被bind() , connect() 等函数调用。  

```c
struct sockaddr{  
    unsigned short sa_family;     //2  bytes
    char sa_data[14];             //14 bytes
};  
```

通用sockaddr,具体到sockaddr_in,sockaddr_un
#### **sockaddr_in**   
网络地址sockaddr Internet

```c
struct sockaddr_in{  
      short   int   sin_family;               //2字节 协议簇/地址簇
      unsigned   short   int   sin_port;      //2字节 端口
      struct   in_addr   sin_addr;            //4字节 ip
      unsigned   char   sin_zero[8];          //8字节 无意义仅为对齐
};  
```

sin_family: AF_INET, PF_INET
>其实是TCP/IP的设计者一开始想多了。   
PF是protocol family，AF是address family，作者一开始以为可能某个协议族有多种形式的地址，所以在API上把它们分开了，创建socket用PF，bind/connect用AF。    
结果一个PF只有一个AF，从来没有过例外，所以就混用了。   
摘自：[PF_INET AF_INET的区别是什么？](https://blog.csdn.net/xiaolei251990/article/details/83030523)

**IP地址**
```c
struct in_addr{  
            union {
               struct { u_char s_b1,s_b2,s_b3,s_b4; } S_un_b;
               struct { u_short s_w1,s_w2; } S_un_w;
               u_long S_addr; 
             } S_un;
        #define s_addr  S_un.S_addr
};  
```
32位ip地址3中表达方式
或者：

```c
struct in_addr {
    in_addr_t s_addr;
};
```
**使用**

```c
struct sockaddr_in my_addr;  
my_addr.sin_addr.s_addr=inet_addr("192.168.0.1");       
```
**inet_addr:**    
inet_addr是一个计算机函数，功能是将一个点分十进制的IP转换成一个长整数型数（u_long类型）等同于inet_addr().     
若字符串有效则将字符串转换为32位二进制网络字节序的IPV4地址，否则为INADDR_NONE    


**inet_aton:**   
inet_aton是一个计算机函数，功能是将一个字符串IP地址转换为一个32位的网络序列IP地址。   
如果这个函数成功，函数的返回值非零，如果输入地址不正确则会返回零。使用这个函数并没有错误码存放在errno中，所以它的值会被忽略    



## socket创建

```
int socket(int domain, int type, int protocol);
```
### 参数说明　　
**domain：** 协议域，又称协议族（family）。常用的协议族有AF_INET、AF_INET6、AF_LOCAL（或称AF_UNIX，Unix域Socket）、AF_ROUTE等。协议族决定了socket的地址类型，在通信中必须采用对应的地址，如AF_INET决定了要用ipv4地址（32位的）与端口号（16位的）的组合、AF_UNIX决定了要用一个绝对路径名作为地址。     
**type：** 指定Socket类型。常用的socket类型有SOCK_STREAM、SOCK_DGRAM、SOCK_RAW、SOCK_PACKET、SOCK_SEQPACKET等。流式Socket（SOCK_STREAM）是一种面向连接的Socket，针对于面向连接的TCP服务应用。数据报式Socket（SOCK_DGRAM）是一种无连接的Socket，对应于无连接的UDP服务应用。     
**protocol：** 指定协议。常用协议有IPPROTO_TCP、IPPROTO_UDP、IPPROTO_STCP、IPPROTO_TIPC等，分别对应TCP传输协议、UDP传输协议、STCP传输协议、TIPC传输协议。   
[IPPROTO_TCP 的具体数值](https://blog.csdn.net/huabiaochen/article/details/78963634)

***注意：***   
1. type和protocol不可以随意组合，如SOCK_STREAM不可以跟IPPROTO_UDP组合。当第三个参数为0时，会自动选择第二个参数类型对应的默认协议。  
2. WindowsSocket下protocol参数中不存在IPPROTO_STCP   

## 基本操作

int bind(SOCKET socket, const struct sockaddr* address, socklen_t address_len);
int accept(SOCKET socketServer,struct sockaddr * addr,int * addrlen);
int recv(SOCKET socket, char FAR* buf, int len, int flags);
ssize_t recvfrom(int sockfd, void buf, int len, unsigned int flags, struct socketaddr* from, socket_t* fromlen);

## Other

### 操作路由表
>[网络编程学习笔记(ioctl操作)](https://blog.csdn.net/xiexingshishu/article/details/40918971)    
[ioctl百度百科](https://baike.baidu.com/item/ioctl/6392403?fr=aladdin)    
ioctl是设备驱动程序中对设备的I/O通道进行管理的函数。


```c
//操作路由表 data:info
//route add -net 192.168.2.XXX netmask 255.255.255.255 dev rmnet_data1
bool addRoute(BASEROUTEINFO *info)
{
    struct rtentry route;//linux 路由结构
    struct sockaddr_in *addr;
    int skfd;

    memset((char *)&route, 0x00, sizeof(route));

    //设置net方式
    route.rt_flags = RTF_UP ;

    //设置需要加入路由的ip
    addr = (struct sockaddr_in*) &route.rt_dst;
    addr->sin_family = AF_INET;
    addr->sin_addr.s_addr = inet_addr(info->host);

    //设置netmask
    addr = (struct sockaddr_in*) &route.rt_genmask;
    addr->sin_family = AF_INET;
    addr->sin_addr.s_addr = inet_addr(info->netmask);

    //设置dev
    route.rt_dev = info->dev;

    //创建socket
    skfd = socket(AF_INET, SOCK_DGRAM, 0);

    //删除路由表项route：ioctl(skfd, SIOCDELRT, &route)
    //增加路由
    if(ioctl(skfd, SIOCADDRT, &route) < 0)//ioctl: 设备驱动程序中对设备的I/O通道进行管理的函数。
    {
        ALOGD(LOG_TAG, "addRoute SIOCADDRT fail!\n");
        close(skfd);
        return false;
    }
    close(skfd);
    return true;
}
```



### 字节序
主机字节序，网络字节序<!-- 对于Java接触不到-->

网络字节序： 高位低地址
data=0x0102030405060708...

网络字节序读出第一字节"01"第二字节"02"....


#### 相关函数
hton[s|l] : h:host   n:net   s:16位数值  l:32位数值


---
# add_ons

## **sockaddr_un**
>[UNIX Domain Socket IPC (sockaddr_un )](https://blog.csdn.net/ace_fei/article/details/6412069)

进程间通信的一种方式是使用UNIX套接字，人们在使用这种方式时往往用的不是网络套接字，而是一种称为 **本地套接字的方式** 。这样做可以避免为黑客留下后门。

socket API原本是为网络通讯设计的，但后来在socket的框架上发展出一种IPC机制，就是UNIX Domain Socket。虽然网络socket也可用于同一台主机的进程间通讯（通过loopback地址127.0.0.1），但是UNIX Domain Socket用于IPC更有效率： **不需要经过网络协议栈，不需要打包拆包、计算校验和、维护序号和应答等，只是将应用层数据从一个进程拷贝到另一个进程。**  这是因为，***IPC机制本质上是可靠的通讯，而网络协议是为不可靠的通讯设计的。*** UNIX Domain Socket也提供面向流和面向数据包两种API接口，类似于TCP和UDP，但是面向消息的UNIX Domain Socket也是可靠的，消息既不会丢失也不会顺序错乱。

UNIX Domain Socket是全双工的，API接口语义丰富，相比其它IPC机制有明显的优越性，目前已成为使用最广泛的IPC机制，比如X Window服务器和GUI程序之间就是通过UNIX Domain Socket通讯的。

使用UNIX Domain Socket的过程和网络socket十分相似，也要先调用socket()创建一个socket文件描述符，`address family`指定为`AF_UNIX`，`type`可以选择`SOCK_DGRAM`或`SOCK_STREAM`，`protocol`参数仍然指定为`0`即可。

UNIX Domain Socket与网络socket编程最明显的不同在于地址格式不同，用结构体`sockaddr_un`表示， **网络编程的socket地址是IP地址加端口号，而UNIX Domain Socket的地址是一个socket类型的文件在文件系统中的路径，这个socket文件由bind()调用创建，如果调用bind()时该文件已存 在，则bind()错误返回。**    
```c
struct sockaddr_un{
  sun_family: sa_family_t;//sa_family_t的类型是WORD,即unsiged short
  sun_path: array [0..107] of Char;
}
```

### 创建socket
使用套接字函数socket创建，不过传递的参数与网络套接字不同。域参数应该是`PF_LOCAL`或者`PF_UNIX`，而不能用`PF_INET`之类。本地套接字的通讯类型应该是`SOCK_STREAM`或`SOCK_DGRAM`，协议为默认协议。例如：
```c
int sockfd;
sockfd = socket(PF_LOCAL, SOCK_STREAM, 0);
```
### 绑定
创建了套接字后，还必须进行绑定才能使用。不同于网络套接字的绑定，本地套接字的绑定的是struct sockaddr_un结构。struct sockaddr_un结构有两个参数：sun_family、sun_path。sun_family只能是AF_LOCAL或AF_UNIX，而sun_path是本地文件的路径。通常将文件放在/tmp目录下。例如：

```c
struct sockaddr_un sun;
sun.sun_family = AF_LOCAL;        //sun_family
strcpy(sun.sun_path, filepath);   //sun_path

bind(sockfd, (struct sockaddr*)&sun, sizeof(sun));
```

### 监听
本地套接字的监听、接受连接操作与网络套接字类似。

### 连接
连接到一个正在监听的套接字之前，同样需要填充struct sockaddr_un结构，然后调用connect函数。

连接建立成功后，我们就可以像使用网络套接字一样进行发送和接受操作了。甚至还可以将连接设置为非阻塞模式，这里就不赘述了。
