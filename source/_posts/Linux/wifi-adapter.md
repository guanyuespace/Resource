- 参考：    
[still-no-wifi-adapter-for-realtek-rtl8822be-found-in-18-04](https://askubuntu.com/questions/1067286/still-no-wifi-adapter-for-realtek-rtl8822be-found-in-18-04)  
[联想拯救者 + ubuntu16.04 + WIFI设置](https://www.jianshu.com/p/e513b515149f)       
[无线网卡无法使用的问题](https://www.jianshu.com/p/d7731e86379b)    
[联想笔记本安装ubuntu，无线网卡被禁用的问题及解决方法](https://blog.csdn.net/Nurke/article/details/76465968)   


I was having the same issue on a Lenovo Legion Y530 laptop, which has a Realtek rtl8822be wireless network card, with 18.10 Ubuntu Mate. What I did to fix it was, plugging it into the network via ethernet, I ran sudo apt-get update and sudo apt-get upgrage then, using your favorite text editor edit blacklist.conf
```shell
sudo gedit /etc/modprobe.d/blacklist.conf
```
I added the line:
```shell
blacklist ideapad_laptop
```
then saved and closed the file, and rebooted.

if you didnt want to make this change, without testing it, you could do this:
```shell
sudo rmmod ideapad_laptop
```
doing this, gave me instant results. wireless networks appeared, I was able to connect, and life was great again. I have no idea if your setup will have an ideapad module, but you might try it. Alternatively, you might have an HP module that needs blocking to enable this device.   
