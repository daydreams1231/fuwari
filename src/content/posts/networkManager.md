---
title: NetworkManager教程
published: 2025-09-18
description: ''
image: ''
tags: [Linux, 网络, Wifi]
category: '教程'
draft: false 
lang: ''
---
[NetworkManager文档](https://www.networkmanager.dev/docs/api/latest/)

# 获取ipv4/ipv6地址的几种模式
  - auto: 正常DHCP获取地址
  - manual: 手动配置地址和路由，DNS等信息
  - disabled: 关闭此接口的对应协议(可用于禁用ipv4/6)
  - link-local: 为接口自动分配一个169.254.0.0/16内的地址, 类似windows在连接到另一台电脑，却没有dhcp server时自己给自己分配的地址
  - shared: 将接口配置为共享模式，类似于openwrt中的lan接口，并且NetworkManager会在此接口上启动dhcp server & dns forwarder, 还会处理nat路由(出口是设备的默认网关) (这一点我没试过)

# 默认路由相关
## never use this network for default route:
同义: 仅对该网络上的资源使用此连接 <br>
字面意思，启用后该接口就不会是默认路由, 但局域网路由还有，不影响手动配置的路由 <br>

## ignore automatically obtained routes:
忽略自动获取的默认路由, 不影响局域网路由，不影响手动配置的路由 <br>
google上说该选项启动后会影响接口的所有路由，因此要手动配置默认和非默认路由 <br>
但我在debian11和ubuntu24.04上实测下来还是会有局域网路由，idk, 慎用这个选项 <br>

# Available to all users
同义: 此连接对其他用户可用 <br>
用于控制其他用户能否修改这个连接, 比如改DNS之类的 <br>
这个我没试过 <br>

