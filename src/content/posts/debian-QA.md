---
title: Debian系统常见问题
published: 2026-02-26
description: '解决日常使用时可能遇到的问题, 并以此学习Linux系统, 对其他发行版大体适用'
image: ''
tags: [Linux, Debian]
category: '教程'
draft: false 
lang: 'zh_CN'
---

# ZRAM / Swap
Swap是Linux系统中的交换空间, 当物理内存不足时, 系统会将一些不常用的内存页移动到交换空间中, 以释放物理内存. <br>
ZRAM是一种内存压缩技术, 可以将一部分内存压缩成一个虚拟的块设备, 作为交换空间使用. <br>
ZRAM和Swap的区别在于, Zram压缩内存页, 压缩的结果存储在内存中, 而Swap则是将内存页移动到磁盘上. <br>

Zram 和 Swap 相辅相成, 都属于交换内存, 可以同时使用, 也可以单独使用. 具体选择要根据设备环境而定.<br>
:::info
cat /proc/sys/vm/swappiness: 该值决定了系统在内存不足时, 选择使用交换空间的倾向程度. 值越大, 越倾向于使用交换空间.
:::
### ZRAM
一般正常用的话首选ZRAM, 毕竟硬盘的性能还是远远比不上内存(主要是访问时延) <br>
> 使用ZRAM需要内核配置了: `CONFIG_ZRAM` 和 `CONFIG_ZRAM_COMPRESS` <br>
```shell
sudo apt install zram-tools

# 查看可用的内存压缩算法:
cat /sys/block/zram0/comp_algorithm

# 编辑 /etc/default/zramswap
nano /etc/default/zramswap
# 根据内存的紧张程度, 选择不同的算法:
# 例如 1G 及以下内存的VPS, 选择 zstd
# 2G / 4G 及以上的机器, 选择 lz4
# 其余选项保持默认

sudo zramctl
```

### SWAP
兼容性最好, 但性能与硬盘极度相关, 并且会增加硬盘的磨损<br>
如果ZRAM都不够用, 那就只能使用SWAP了<br>
```shell
fallocate -l 1G /swap && chmod 600 /swap && mkswap /swap && swapon /swap && echo "/swap swap swap defaults 0 0" >> /etc/fstab
```