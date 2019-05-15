---
title: 数据粘包
date: 2019-05-15 14:46:50
categories:
- TCP

tags:
- TCP

---
# 数据粘包
>TCP发送端为了将多个发往接收端的包，更有效的发到对方，使用了优化方法（Nagle 算法），将多次间隔较小且数据量小的数据，合并成一个大的数据块，然后进行封包。     
>[为什么会TCP粘包](https://blog.csdn.net/m0_38002798/article/details/78307870)


<!-- more -->
