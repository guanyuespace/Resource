---
title: 双网卡同时上网
date: 2019-12-14 16:22:31

categories:
- 网络配置

tags:
- Network
---
# 双网卡配置
>环境：win10 要求：两张网卡，一张用于内网网络；另一张用于连接外网
<!-- 两张网卡互不影响，一张专用外网，一张专用内网 -->

## 网络环境

```
# 外网网卡
wlan0{
  address 192.168.0.111
  netmask 255.255.255.0
  gateway 192.168.0.1
}

# 内网网卡
wlan1{
  address 192.168.11.123
  netmask 255.255.255.0
  gateway 192.168.11.1
}
```

## 操作
> 修改路由表项

```
$route print
IPv4 路由表
===========================================================================
活动路由:
网络目标        网络掩码          网关       接口   跃点数
0.0.0.0          0.0.0.0      192.168.0.1    192.168.0.114     40
0.0.0.0          0.0.0.0     192.168.11.1   192.168.11.100    306
$route delete 0.0.0.0                                         //删除所有0.0.0.0相关的ip路由项
$route -p add 192.168.11.0 mask 255.255.255.0  192.168.11.1   //内网路由转发
$route -p add 0.0.0.0 mask 0.0.0.0  192.168.0.1               //外网路由转发
```