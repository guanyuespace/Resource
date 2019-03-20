---
title: Linux
date: 2019-02-28 09:22:12
tags:
- relax
- Linux
---
# Linux
>参考:[EFI system partition](https://wiki.archlinux.org/index.php/EFI_system_partition)   
[how do i install ubuntu alongside a pre installed windows with uefi](https://askubuntu.com/questions/221835/how-do-i-install-ubuntu-alongside-a-pre-installed-windows-with-uefi)   

## 内存空间查看
- df: df -lh
- fdisk: fdisk -l
- cfdisk
<!-- more -->

## Linux分区挂载点介绍(old, just for comprehension)  
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

## UEFI引导与BIOS引导
***Warning: When dual-booting, avoid reformatting the ESP, as it may contain files required to boot other operating systems.***   

***Keep in mind in here, maybe you have to configure disk form and booting system to GPT partition’s UEFI***      

### EFI(ESP)
>Copied from:[EFI system partition](https://wiki.archlinux.org/index.php/EFI_system_partition)   

The EFI system partition (also called ESP) is an OS independent partition that acts as the storage place for the EFI bootloaders, applications and drivers to be launched by the UEFI firmware. It is mandatory for UEFI boot.       

If you are installing Arch Linux on an UEFI-capable computer with an installed operating system, like Windows 10 for example, it is very likely that you already have an EFI system partition.    

To find out the disk partition scheme and the system partition, use fdisk as root on the disk you want to boot from:   
```shell
# fdisk -l /dev/sd*
```
The command returns:
- The disk's partition table: it indicates Disklabel type: gpt if the partition table is GPT or Disklabel type: dos if it is MBR.
- The list of partitions on the disk: Look for the EFI system partition in the list, it is a small (usually about 100–550 MiB) partition with a type EFI System or EFI (FAT-12/16/32). To confirm this is the ESP, mount it and check whether it contains a directory named EFI, if it does this is definitely the ESP.

#### create partition: efi

Warning: The EFI system partition must be a physical partition in the main partition table of the disk, not under LVM or software RAID etc.

To avoid potential problems with some UEFI implementations, **the ESP should be formatted with FAT32 and the size should be at least 512 MiB.** ***550 MiB is recommended to avoid MiB/MB confusion and accidentally creating FAT16, although larger sizes are fine.***

*According to a Microsoft note, the minimum size for the EFI system partition (ESP) would be 100 MiB,* though this is not stated in the UEFI Specification. *Note that for Advanced Format 4K Native drives (4-KiB-per-sector) drives, the size is at least 256 MiB, because it is the minimum partition size of FAT32 drives (calculated as sector size (4KiB) x 65527 = 256 MiB), due to a limitation of the FAT32 file format.*

---   

## Secure Boot
>[how do i install ubuntu alongside a pre installed windows with uefi](https://askubuntu.com/questions/221835/how-do-i-install-ubuntu-alongside-a-pre-installed-windows-with-uefi)

>Before explaining the steps to do it, I want to be clear that I have tried many ways of installing Ubuntu with versions older than 15.04 (Or any other distro for that matter) from within Windows 8 or Windows 10. No luck. Microsoft Windows really created a big mess for all Linux distributions. **If you have a pre-installed Windows 8 system, you will probably never be able to install Ubuntu or any other OS in the normal (LiveCD/LiveUSB) or Wubi way.** This is because Windows 8 introduced several new features, of which 2 are:
>- UEFI which substitutes what we have known as the BIOS (an alternative to)
>- Secure Boot which prevents anything but the installed operating system, in this case, Windows 8 from booting. This is no longer the case for Ubuntu since 12.04.2 so there is no need to disable secure boot.
>   
>On a further note I want to mention something about Secure Boot taken from the UEFI Wiki


"Secure Boot" is a new UEFI feature that appeared in 2012, with Windows 8 preinstalled computers. Ubuntu supports this feature starting with 12.10 64 bit (see this article) and 12.04.2 64 bit, but as PCs implementing support for it have only become widespread at the end of 2012 it is not yet widely tested, so it's possible that you may encounter problems booting Ubuntu under Secure Boot.            


```
once you've installed with Secure Boot disabled. As mentioned by slangasek:      
It is not required to disable SecureBoot in the firmware to install Ubuntu on a Windows 8 machine.
Ubuntu 12.04.2 and 12.10 are SecureBoot-compatible.
Any machine that ships with the recommended Microsoft Third-Party Marketplace keys in firmware will be able to boot Ubuntu under SecureBoot.
If there is any problem file a launchpad bug for the shim package.
```
<!-- This was with Secure Boot on and on an EFI enabled boot system(Ubuntu 15.04 X64). I also. Tested 4 Windows 10 PCs and it worked perfectly with 15.10 & 16.04. -->
<!-- The message above shows you can install Ubuntu whose version is higher than 15.04(X64) with SecureBoot on,but you will fail to install it with SecureBoot on somtimes.  -->
<!-- first you should enable the SecureBoot.if problems occur, you can disable it for a try later ... -->


### Installing Ubuntu
The following is a small guide to install Ubuntu with a Pre-Installed Windows 8 or 10 system. The steps HAVE TO BE done in the precise order I mention them here to get everything started. If a step is skipped or done before another, you will most likely end up with some of the problems mentioned at the bottom of this guide.

For the time, you need to do it via a LiveCD, LiveDVD or LiveUSB, assuming (actually requiring) you have the following points:       
<!--   
just be ready      
\- You are using a 64-bit version of at least Ubuntu 12.04.2. 32-bit versions will not work.   
\- Your system came with Windows 8 or 10 pre-installed (And you do not want to delete it)   
\- You are not installing Ubuntu inside of Windows 8 or 10 but rather alongside of it. Inside it is impossible because it needs Wubi which is unsupported.
\- Your system has UEFI activated (And cannot be disabled) with Secure Boot.
\- You have already created a free space for Ubuntu from within Windows 8 with at least 8 GB (I recommend to leave at least 20 GB or so, so you can test the hell out of it).   
\- You made sure that you actually have free space left on the drive to create the needed partitions and you also made sure that you did not have all primary partitions used (In case of using an MS-DOS Scheme) because this will create a problem with the Ubuntu installer showing you only the "Replace Windows" option instead of the "Alongside Windows" option.   
\- You know how to burn a LiveCD, LiveDVD or LiveUSB from within Windows 8. If not, look for Windows apps that can do that for you.^-^.   
-->   
- Windows 8 was not shutdown in either Hibernation mode(休眠模式) or any other mode ('fast start-up' which is by default on Windows 8) that leaves it on a saved state. *Shutdown Windows 8 in the normal way, with the shutdown option.* This will prevent other problems related to this from appearing. Read the bottom (TROUBLESHOOT) of this answer for more information regarding this point.
- You are installing on an MS-DOS type(MBR) disk scheme (You can only have 4 primary partitions as opposed to GPT Scheme) which has at least 1 Free Primary Partition (You can find out the type of scheme you have from here if operating on an Ubuntu Live CD or here if from Windows). **Remember that if you are already using 4 Primary Partitions no partitions will appear on the Ubuntu installer since there are no more Primary partitions left to use (MS-DOS type partitions are limited to 4 Primary ones; GPT are limited to 128 because of the limitation of Windows).** This happens a lot on many laptops that come with 4 pre-created primary partitions. If you are installing on a GPT type partition and want it to boot, you need to *leave UEFI enabled*.

#### Before we start we need to do the following:
Run`compmgmt.msc`on Windows 8. From there on, create a partition with enough size. Note that I mention creating this FROM Windows 8 because I have had cases where doing the partition from the LiveUSB rendered Windows 8 unbootable, even after doing a boot repair. So to remove that problem or have a greater chance of removing it (Or simply skipping the problem altogether) and making sure both systems work, partition your hard drive from within Windows 8 first.     
Now follow this steps to have a working Windows 8 + Ubuntu installed on your system:   
**Windows 8 + Ubuntu**   
We first need to know with `what type of motherboard options we are dealing with.` Open a terminal (By going to the start menu and typing PowerShell for example) and run the terminal as an Administrator (Right Click the app that will show in the start menu and select Run as Administrator). Now type `Confirm-SecureBootUEFI`. This can give you 3 results:   
```
True - Means your system has a Secure boot and is Enabled
False - Means your system has a Secure boot and is Disabled
Cmdlet not supported on this platform - Means your system does not support Secure boot and most likely you do not need this guide.
      You can install Ubuntu by simply inserting the LiveCD or LiveUSB and doing the installation procedure without any problems.
```
If you have it Enabled and have the necessary partitioning done then we can proceed with this guide. After booting into Windows 8 we go to the power off options and while holding the SHIFT key, click on Restart.
### Select UEFI Firmware Settings
*NOTE - In the Spanish version of Windows 8, the option for UEFI Firmware Settings is not available in several laptops, tested Lenovo, HP, and Acer.* They do have an option to boot the computer and another custom menu will appear which lets you do a couple of things. In the case of Lenovo, you will not have an option to install Ubuntu with Windows 8, the only option is to remove Windows 8 completely. This only applies if you are not using 15.04+.

### THIS IS AN IMPORTANT PART
The system will reboot and you will be allowed to go to the BIOS (If not press the appropriate key, some common are DEL,F2 or F10).   
In this part, I can't help much since each BIOS is different for each Motherboard model. There are 2 options you can take here, both of which are optional since Ubuntu might install without any problems at all. **You can either look for an option to disable Secure Boot or an option to disable UEFI.** In some cases you will be able to find both, it will show in the BIOS as an option called Secure Boot or Enable UEFI.    

If you find this options, then depending if you cannot install Ubuntu with Secure Boot enable then disable Secure Boot (Remember to report this as a bug using ubuntu-bug shim), to be able to still stay in UEFI mode and also be able to Boot with Ubuntu. In some motherboards, this will be the only option you actually need to change and also will be the only option you see related to UEFI because they will not offer the possibility to disable UEFI.

## DUAL BOOT ISSUES
<font size="+1" color="red">If you happen to install Ubuntu in Legacy Mode (No SecureBoot) you might have problems booting both, Windows and Ubuntu at the same time since they will both not appear on a Dual-Boot Menu. If you have Windows on UEFI for example and you install Ubuntu on Legacy Mode, you will only be able to boot to Ubuntu in Legacy Mode and Windows in UEFI Mode.</font><br/>

So before proceeding, make sure that** you are installing Ubuntu with the same boot options as Windows.** This way you will be able to choose which one to boot from in the same boot menu and not worry if one will work or not. From the [Ubuntu UEFI Guide](https://help.ubuntu.com/community/UEFI#Converting_Ubuntu_into_EFI_mode) you can see that there is a section that teaches you how to know if you actually installed Ubuntu in the same Boot setup as Windows (UEFI Mode)
```
An Ubuntu installed in EFI mode can be detected the following way:
    its /etc/fstab file contains an EFI partition (mount point: /boot/efi)
    it uses the grub-efi bootloader (not grub-pc)
    from the installed Ubuntu, open a terminal (Ctrl+Alt+T) then type the following command:

    [ -d /sys/firmware/efi ] && echo "Installed in EFI mode" || echo "Installed in Legacy mode"
```  
So if you have ANY dual boot problems, this could be the problem. Please read the [Ubuntu UEFI Guide](https://help.ubuntu.com/community/UEFI) since it covers various ways of solving Dual boot problems and converting Ubuntu to Legacy or EFI mode. I have already tested this with various Ask Ubuntu members that helped me apart from 2 Laptops I was provided with for the testing. This should then solve any Dual Boot problems related to Windows 8 + Ubuntu, but I again encourage anyone with problems (same or new) to file a bug report as mentioned above. The Ubuntu Developers are working very hard in providing an easy to install solution for all cases and this is one of the top priorities.     


### Some points we should consider before continuing
- If Windows 8 was installed with UEFI enabled, it is highly recommended to stay in UEFI, although if you still want to disable it for specific reasons you can, GRUB will create the bootable part for Windows 8. But if you do disable UEFI and want to access Windows 8 afterward (before installing Ubuntu), it will not work since the boot part for Windows 8 needs UEFI (Again the Dual Boot problem).
- If you only disable Secure Boot, there is no problem in some cases. You are only disabling the part that creates the most problem between Windows and Linux, which is the one that prevents Ubuntu from booting correctly.**In either case, I encourage you to first try to install Ubuntu with UEFI/Secureboot, since in most cases it will work. if you disable any of them and install Ubuntu, you might not be able to boot to Windows 8 afterward through the GRUB Boot Menu.**

Now before saving, some motherboards offer a `Boot Mode` option. Verify that this option is not pointing to `UEFI Boot` but instead to `CSM Boot` (Compatibility Support Module) which provides support for Legacy BIOS like systems.    
Other systems offer a UEFI Boot option you can enable or disable. Depending on the options I mentioned above you can set this to the one you want.     
And lastly, others offer a UEFI/Legacy Boot First option where you select which one you wish to use first. Obviously, the option is self-explanatory.    

others ... ...

## 问题     

### 选择安装目录时检测不到空闲硬盘下分区：  
![](./part.jpg)   
将`动态磁盘`切换成`基本磁盘`,剩余空间不要新建简单卷   

### partions使用xfs分区,sda没有足够的空间写入core.img   
如果选则不创建grub引导，则会跳过此布异常，但是开机后会直接进入windows需要在windows下创建Cent OS引导项...
<!--谜之操作: 存在一次安装时，成功:将/biosboot, /boot分区挂载到C盘，其他分区不用管，**并将引导项装入C盘**，安装完成后，可以启动Linux但是Windows引导项被覆盖，无法进入windows -->   

### ~~修复windows引导项(solved)~~
>参考:[how do i install ubuntu alongside a pre installed windows with uefi](https://askubuntu.com/questions/221835/how-do-i-install-ubuntu-alongside-a-pre-installed-windows-with-uefi)   
> [DUAL BOOT ISSUES](### DUAL BOOT ISSUES)   
>
>**对于本次装机中此问题的产生原因：对Win 10装机时采用UEFI引导（SecureBoot: enable;UEFI:first）;而装双系统时，Cent OS采用了BIOS引导（SecureBoot：disable;Lengacy  Lengacy first）,因此开机时只出现Cent OS 引导而没有Win 10引导（猜测：当采用UEFI first启动时能够启动win 10）**    
> 采用UEFI引导装机（Linux）后，重启同时出现Win 10与Cent OS引导项

~~编辑`/boot/grub2/grub.cfg`文件~~     
```sh
menuentry "Windows 10" {
    insmod part_msdos  //mdr分区
    insmod ntfs
    set root='(hd0,msdos1)'
    chainloader +1    
}
```   
~~大致解释下，hd0 代表 Windows 系统所在的硬盘，msdos1 代表 Windows 系统所在的分区。需要注意的是，Grub 对所有硬盘的分类都表示为 hd，但现在 Linux 系统大多为 Grub2 引导，Grub2 对磁盘的分类更加详细。~~      
~~磁盘分类可能表示为 hd 、sd ，其中 hd0 表示第一块磁盘， hd1 表示第二块... 依此类推。对于 sd 则有所不同，sda 表示第一块磁盘，sdb 表示第二块... 依此类推。~~    
~~Linux 中通过 `df -TH` 命令可以查看硬盘的具体信息，比如你的 Windows 系统所在的硬盘为 sdb4，则代表是第二块硬盘的第四分区，代码表示为 '(sd1,msdos4)'~~        
~~在grub中`list -l` 打印...~~    


#### ~~进入Cent OS前~~
~~进入系统前"press C for commandline"--&gt;grub~~  
```sh
grub>ls -l

list partions: (hd0,gpt2)

grub>insmod part_gpt
grub>insmod ntfs
grub>set root='hd0,gpt2'
grub>chainloader EFI/Miscrosoft/Boot/zh-CN/bootmgfw.efi

//invalid file

grub>chainloader EFI/Boot/bootx64.efi

//invalid file

grub>boot
```
#### ~~进入Cent OS后~~
```sh
#gedit /boot/grub2/grub.cfg

menuentry "Windows 10" {
  insmod part_gpt
  insmod ntfs
  set root='hd0,gpt2'
  chainloader +1
}

#reboot
```

~~尝试多次，失败告终...~~    

<!-- WinPE 修复引导项失败，重装系统... -->  
<!-- Windows 10重装UEFI -- GPT;BIOS -- MBR(Secure Boot: disable;Fast Boot: disable;Prefered OS: disable;). -->
<!-- SECURE BOOT功能:Windows 8中增加了一个新的安全功能,Secure Boot内置于UEFI BIOS中,用来对抗感染MBR、BIOS的恶意软件,  Windows 8 缺省将使用Secure Boot,在启动过程中，任何要加载的模块必须签名(强制的)，UEFI固件会进行验证， 没有签名或者无法验证的，将不会加载。 -->
<!-- EFI partitions: efi(esp) 260M, msr 1024M, primary -->
<!-- shift+f10 : commandline : diskpart  , help, list, sel, clean, convert gpt/mbr, create partition xxxx size=xxxx etc.  -->
~~惨败收场~~   
~~solvedsolvedsolvedsolvedsolvedsolvedsolvedsolved~~

### ~~Cent OS下无法连接wifi(无线网卡：rtl8822be)~~
No Adapter   
参照[rtlwifi_new](https://github.com/lwfinger/rtlwifi_new)  [rtlwifi_next](https://github.com/rtlwifi-linux/rtlwifi-next.git),编译失败   
怀疑：`This code will build on any kernel 4.2 and newer as long as the distro has not modified any of the kernel APIs.`    
刚安装Cent OS 7.6 内核版本3.1  

没有有线网，没有wifi，一台笔记本 jj ... ...

....

升级内核5.1后     
[rtlwifi_new](https://github.com/lwfinger/rtlwifi_new.git)编译成功,成功安装
```sh
sudo modprobe -r rtl8822be
sudo modprobe rtl8822be
```
执行到此处时，出错
modprobe ...

```
lspci | grep Wireless
```
无显示

### USB共享手机网络
MIUI 系统

USB连接选项：仅限充电
设置：更多连接设置：USB网络共享

[how to make my pci wifi card rtl8822 working on ubuntu](https://askubuntu.com/questions/926364/how-to-make-my-pci-wifi-card-rtl8822-working-on-ubuntu)  
[Lenovo A485:RTL8822BE-firmware](https://github.com/samcv/A485-RTL8822BE-firmware)   
[No WiFi adapter in ubuntu 18 04 LTS](https://h30434.www3.hp.com/t5/Notebook-Wireless-and-Networking/No-WiFi-adapter-in-ubuntu-18-04-LTS/m-p/6785800#M154574)    
...


### 内核升级
#### 配置源
```sh
# restore old yum mirrors
cd /etc/yum.repos.d/
mkdir repo_bak
mv *.repo repo_bak/
# 在CentOS中配置使用网易和阿里的开源镜像
wget http://mirrors.aliyun.com/repo/Centos-7.repo -o /etc/yum.repos.d/aliyun.repo
wget http://mirrors.163.com/.help/CentOS7-Base-163.repo -o /etc/yum.repos.d/163.repo
# 清理系统缓存
yum clean all
# 生成yum缓存
yum makecache
# 安装epel源
yum list | grep epel-release
yum install -y epel-release
# 使用阿里开源镜像提供的epel源
wget -O /etc/yum.repos.d/epel-7.repo http://mirrors.aliyun.com/repo/epel-7.repo    # 下载阿里开源镜像的epel源文件
# 清理系统缓存
yum clean all
# 生成yum缓存
yum makecache
# 查看系统可用的yum源和所有的yum源
yum repolist enabled
yum repolist all
```
#### 内核升级
参考[centos 手动升级系统内核](https://blog.csdn.net/u010654572/article/details/51755465)   
