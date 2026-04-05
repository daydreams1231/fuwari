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

## 安全弹出硬盘
> umount并不是安全弹出, 它只是取消挂载, 但设备实际还是能被访问的 <br>

```shell
# 使用gio命令. Doc: https://manpages.debian.org/testing/libglib2.0-bin/gio.1.en.html
sudo apt install gio

gio mount -t /dev/sda1
```

```shell
# 使用 udisks2
sudo apt install udisks2

udisksctl power-off -b /dev/sda
```

# 检查磁盘坏块
```shell
badblocks -s -v -o /root/bb.log /dev/sdb
```
# 检查磁盘
```shell
fsck -a /dev/sdb
```
# 查看WiFi AP的mac:
```shell
sudo iw dev wlan0 link
```
# 查看周围WLAN (需要NetworkManager)
```shell
nmcli dev wifi list
```
# 磁盘读写测速
```shell
sudo dd if=/dev/zero of=temp.bin bs=1M count=1024 status=progress oflag=direct <br>
sudo dd if=temp.bin of=/dev/null bs=1M count=1024 status=progress iflag=direct
```

# 不使用Netplan更改接口跃点数
```shell
sudo apt install ifmetric
sudo ifmetric 网卡名 跃点数
```
# cpupower
```shell
sudo apt install linux-cpupower
# 查看当前配置
sudo cpupower -c all frequency-info

# 查看可用的配置
cat /sys/devices/system/cpu/cpufreq/policy*/scaling_available_governors
##有几行就代表你的CPU有几个Cluster, Arm CPU一般会有多个Cluster
## 每一行代表当前Cluster可用的governor

# 临时换策略
sudo cpupower -c all frequency-set --governor schedutil

# 持久化配置（Debian/Ubuntu）
/etc/systemd/system/cpufreq.service
```text
[Unit]
Description=Set CPU scaling governor to performance

[Service]
ExecStart=/usr/bin/cpupower frequency-set -g performance

[Install]
WantedBy=multi-user.target
```
sudo systemctl enable --now cpupower
```