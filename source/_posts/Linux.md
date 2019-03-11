---
title: Linux
date: 2019-02-28 09:22:12
tags:
- relax
- Linux
---
# Linux

## 内存空间查看
- df: df -lh
- fdisk: fdisk -l
- cfdisk
<!-- more -->

## Linux分区挂载点介绍  
Linux分区挂载点介绍，推荐容量仅供参考不是绝对，跟各系统用途以及硬盘空间配额等因素实际调整：      

| 分区类型 | 介绍 | 备注 |
| :--- | :--- | --- |
| /boot   | 启动分区  | 一般设置100M-200M，boot目录包含了操作系统的内核和在启动系统过程中所要用到的文件。  |
| /  | 根分区  | 所有未指定挂载点的目录都会放到这个挂载点下。 |
| /home  | 用户目录  | 一般每个用户100M左右，特殊用途，比如放大文件也可再加上G。分区大小取决于用户多少。对于多用户使用的电脑，建议把/home独立出来，而且还可以很好地控制普通用户权限等，比如对用户或者用户组实行磁盘配额限制、用户权限访问等。  |
| /tmp | 临时文件  | 一般设置1-5G，方便加载ISO镜像文件使用，对于多用户系统或者网络服务器来也有独立挂载的必要。临时文件目录，也是最常出现问题的目录之一。  |
| /usr  | 文件系统  | 一般设置要3-15G，大部分的用户安装的软件程序都在这里。就像是Windows目录和Program Files目录。很多Linux家族系统有时还会把/usr/local单独作为挂载点使用。  |
| /var  | 可变数据目录  | 包含系统运行时要改变的数据。通常这些数据所在的目录的大小是要经常变化的，系统日志记录也在/var/log下。一般多用户系统或者网络服务器要建立这个分区，设立这个分区，对系统日志的维护很有帮助。一般设置2-3G大小，也可以把硬盘余下空间全部分为var。  |
| /srv  | 系统服务目录  | 用来存放service服务启动所需的文件资料目录，不常改变。  |
| /opt  | 附加应用程序  | 存放可选的安装文件，个人一般把自己下载的软件资料存在里面，比如Office、QQ等等。  |
| swap  | 交换分区  | 一般为内存2倍，最大指定2G即可 |
|   |   | 以下为其它常用的分区挂载点 |
| /bin | 二进制可执行目录  | 存放二进制可执行程序，里面的程序可以直接通过命令行调用，而不需要进入程序所在的文件夹。  |
| /sbin  | 系统管理员命令存放目录  | 存放标准系统管理员文件  |
| /dev  | 存放设备文件  | 驱动文件等  |


##  U盘装Linux: CentOS
- 制作U盘启动盘
- 安装
自定义文件所在路径：选择ISO所在磁盘
```
Press Tab for full configuration options on menu items
vmlinuz initrd=initrd.img inst.stage2=hd:LABEL=CentOS\x207\x20x86_64 rd.live.check quiet
改为:
vmlinuz initrd=initrd.img linux dd quiet
查看U盘启动盘的名称
重启后：
vmlinuz initrd=initrd.img inst.stage2=hd:/dev/sdb4 quiet   ps：/dev/sdb4就是你看到的启动盘名称
```

## 问题     

### 选择安装目录时检测不到空闲硬盘下分区：  
![](./part.jpg)   
将`动态磁盘`切换成`基本磁盘`,剩余空间不要新建简单卷   

### partions使用xfs分区,sda没有足够的空间写入core.img   
如果选则不创建grub引导，则会跳过此布异常，但是开机后会直接进入windows需要在windows下创建Cent OS引导项...
<!--谜之操作: 存在一次安装时，成功:将/biosboot 分区挂载到C盘，其他分区不用管。安装完成后，可以启动Linux但是Windows引导项被覆盖，无法进入windows -->   

### ~~Cent OS下无法连接wifi~~
No Adapter   
参照[rtlwifi](https://github.com/lwfinger/rtlwifi_new),编译失败   
怀疑：`This code will build on any kernel 4.2 and newer as long as the distro has not modified any of the kernel APIs.`    
刚安装Cent OS 7.6 内核版本3.X  

没有有线网，没有wifi，一台笔记本 jj ... ...

### ~~修复windows引导项~~   
编辑`/boot/grub2/grub.cfg`文件   
```
menuentry "Windows 10" {
    insmod part_msdos
    insmod ntfs
    set root='(hd0,msdos1)'
    chainloader +1    
}
```   
大致解释下，hd0 代表 Windows 系统所在的硬盘，msdos1 代表 Windows 系统所在的分区。需要注意的是，Grub 对所有硬盘的分类都表示为 hd，但现在 Linux 系统大多为 Grub2 引导，Grub2 对磁盘的分类更加详细。
磁盘分类可能表示为 hd 、sd ，其中 hd0 表示第一块磁盘， hd1 表示第二块... 依此类推。对于 sd 则有所不同，sda 表示第一块磁盘，sdb 表示第二块... 依此类推。
Linux 中通过 `df -TH` 命令可以查看硬盘的具体信息，比如你的 Windows 系统所在的硬盘为 sdb4，则代表是第二块硬盘的第四分区，代码表示为 '(sd1,msdos4)'

具体hd序号什么的,不明其意。尝试多次，失败告终...    

<!-- WinPE 修复引导项失败，重装系统... -->   

惨败收场
--