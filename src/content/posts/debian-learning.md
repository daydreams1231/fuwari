---
title: Debian系统常见问题
published: 2026-02-26
description: '解决日常使用时可能遇到的问题, 并以此学习Linux系统, 对其他Linux发行版一般也适用'
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
## ZRAM
一般设备(ram <= 8GiB)首选ZRAM, 大于8G的设备一般不需要SWAP/ZRAM. <br>
如果用了Zram后内存还是不足, 请考虑Swap <br>
> 使用ZRAM需要内核配置了: CONFIG_ZRAM=m CONFIG_ZSMALLOC=y <br>

```shell
sudo apt install zram-tools

# 查看可用的内存压缩算法, 其实选哪个差别不大, 没内存时你应该考虑升级设备
cat /sys/block/zram0/comp_algorithm

# 编辑 /etc/default/zramswap
nano /etc/default/zramswap

sudo zramctl
```

## SWAP
兼容性最好, 但性能与硬盘强相关, 并且会增加硬盘的写入磨损 <br>
```shell
fallocate -l 1G /swap && chmod 600 /swap && mkswap /swap && swapon /swap && echo "/swap swap swap defaults 0 0" >> /etc/fstab
```

## SWAP + ZRAM
```text title="/etc/fstab"
/dev/zram0  none    swap    defaults,priority=100   0   0
/swap       none    swap    defaults,priority=10    0   0
```

# 安全弹出硬盘
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

export TARGET=/dev/mmcblk1
# 连续写入 (1MB块)
fio --name=seq_write --filename=$TARGET --size=4G \
    --bs=1M --rw=write --direct=1 --ioengine=libaio \
    --iodepth=32 --numjobs=1 --group_reporting --unlink=1

# 连续读取 (1MB块)
fio --name=seq_read --filename=$TARGET --size=4G \
    --bs=1M --rw=read --direct=1 --ioengine=libaio \
    --iodepth=32 --numjobs=1 --group_reporting --unlink=1

# 4K随机写入
fio --name=rand4k_write --filename=$TARGET --size=4G \
    --bs=4k --rw=randwrite --direct=1 --ioengine=libaio \
    --iodepth=1 --numjobs=1 --group_reporting --unlink=1

# 4K随机读取
fio --name=rand4k_read --filename=$TARGET --size=4G \
    --bs=4k --rw=randread --direct=1 --ioengine=libaio \
    --iodepth=1 --numjobs=1 --group_reporting --unlink=1

7:3 混合读写
fio --name=mix_70r30w --filename=$TARGET --size=4G \
    --bs=4k --rw=randrw --rwmixread=70 --direct=1 --ioengine=libaio \
    --iodepth=32 --numjobs=1 --group_reporting --unlink=1
```

# 不使用Netplan/NM更改接口跃点数
```shell
sudo apt install ifmetric
sudo ifmetric 网卡名 跃点数
```

# CPU调速器
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

```
## 持久化配置（Debian/Ubuntu）
```text title="/etc/systemd/system/cpufreq.service"
[Unit]
Description=Set CPU scaling governor to performance

[Service]
ExecStart=/usr/bin/cpupower frequency-set -g performance

[Install]
WantedBy=multi-user.target
```
之后, 运行:
```shell
sudo systemctl enable --now cpupower
```

# 创建新用户
```shell
sudo adduser 用户名

# 加入sudo组, 使能sudo权限
sudo usermod -aG sudo 新用户名

# 删除用户
deluser 用户名
```
> 如果不希望用户能登录到服务器, 前往`/etc/passwd`修改末尾为`/usr/sbin/nologin`

# 修复NTFS硬盘 (dirty flag)
```shell
sudo apt install ntfs-3g
sudo ntfsfix -d /dev/sdX1
```

# 配置IPv6
使用`ip -a`查看eth0网卡有没有inet6部分, 如果没有说明系统禁用了IPv6, 需额外执行如下命令:
sudo sysctl -w net.ipv6.conf.all.disable_ipv6=0
后前往:
/etc/network/interfaces
加入:
iface eth0 inet6 dhcp

# 禁用Systemd-Resolved (53端口被占用)
```shell
sudo systemctl stop systemd-resolved
sudo systemctl disable systemd-resolved
```
如果不想禁用:
sudo vim /etc/systemd/resolved.conf
在`[Resolve]`下让 `DNSStubListener=no` 后重启systemd-resolved即可

# nano配置tab缩进为4个空格
nano ~/.nanorc, 写入:
set tabsize 4
set tabstospaces

# 配置sudo不输密码
```shell
sudo visudo
# 找到如下这行
%sudo ALL=(ALL:ALL) ALL

# 添加
<USERNAME> ALL=(ALL) NOPASSWD:ALL
```

# 让APT支持HTTPS源
apt在某个版本后自带https支持, 但还是最好安装如下软件包
```shell
sudo apt install apt-transport-https ca-certificates --reinstall
```

# 换内核
```shell
# 注意启用Backport源
sudo apt update
sudo apt search linux-image
# 找同一个版本的linux-headers和linux-image, 根据平台选择对应的后缀内核, 或者无后缀的通用内核
# 推荐只安装linux-headers, 让apt自动处理依赖, 且安装完后应该会自动运行update-grub
sudo apt install linux-headers-XXX
# 如果上一步没自动运行update-grub, 手动运行
sudo update-grub
# 安装重启完后清理
sudo apt autoremove
```
新版内核一般是带ntfs3驱动的, 不过是以Module形式存在

# debian换源 & 更新
Debian官方对不同类型源的定义: [Here](https://wiki.debian.org/SourcesList) <br>
简而言之:
  - stable: 代表当前主流版本
  - testing: 代表下一个主流版本
  - sid: 不稳定版本, unstable, 可视作滚动发行版
testing和sid都是会经常更新的, 区别仅在于一个新软件会先进入sid, 经测试后进入testing, 最后才会进stable <br>

XXX-updates: 一些软件可能由于某些原因, 需尽快更新到新版本, 而等下个大版本发布可能又来不及, 此时就会进入 updates 源 <br>
XXX-backports: 从testing和sid里重新为stable版本编译的软件源 <br>
XXX-security: 安全更新专用源 <br>

### 一键换源
自动处理老式/新式源, 首选方式
```shell
bash <(curl -sSL https://linuxmirrors.cn/main.sh)
```
### 大版本更新
```shell
# 前往 /etc/apt/sources.list.d/debian.sources 等文件, 将里面的版本代号更改为新版本
apt-get update

# apt upgrade和这差不多, 但依赖处理不如这个智能
apt-get dist-upgrade
```
## 注意事项
Debian13及更新的版本中NTFS3 == module <br>
### Debian12及之前的apt源
每个大版本都会有独立的代号, 比如debian12是bookworm, url后的第一个词除了stable/sid这些之外也可以是版本代号 <br>
```text title="/etc/apt/sources.list"
deb https://mirrors.aliyun.com/debian/ trixie main contrib non-free non-free-firmware

deb https://mirrors.aliyun.com/debian/ trixie-updates main contrib non-free non-free-firmware

deb https://mirrors.aliyun.com/debian-security/ trixie-security main contrib non-free non-free-firmware

deb https://mirrors.ustc.edu.cn/debian bookworm-backports main contrib non-free non-free-firmware
```
:::warn
注意debian11及之前没有 `non-free-firmware` ! <br>

如果版本过老, 镜像URL结尾可能不是 `debian` 而是 `debian-archive` <br>

源码src镜像源会拖慢apt update 速度，如有需要可自行添加 <br>
:::
### Debian12及之前的版本升级到Debian13及之后的版本
在更新完成后, 使用如下命令来迁移apt源
```shell
sudo apt modernize-sources
```
### Debian13及之后的源
```text title="/etc/apt/sources.list.d/debian.sources"
Types: deb
URIs: https://mirrors.aliyun.com/debian
Suites: trixie
Components: main contrib non-free non-free-firmware
Signed-By: /usr/share/keyrings/debian-archive-keyring.gpg

Types: deb
URIs: https://mirrors.aliyun.com/debian
Suites: trixie-updates
Components: main contrib non-free non-free-firmware
Signed-By: /usr/share/keyrings/debian-archive-keyring.gpg

Types: deb
URIs: https://mirrors.aliyun.com/debian-security
Suites: trixie-security
Components: main contrib non-free non-free-firmware
Signed-By: /usr/share/keyrings/debian-archive-keyring.gpg
```

# 无线网卡驱动
```shell
## Intel
apt-get install firmware-iwlwifi
## RTL
apt-get install firmware-realtek
## 高通
apt-get install firmware-atheros
```

# mount挂载的硬盘无法写入
注意挂载点的权限不影响最终能否RWE <br>
如果被挂载设备是主机上物理设备, 挂载后的挂载点可以进行chmod chown等操作, 但如果是cifs等虚拟磁盘则不能进行同样的操作 <br>
mount添加选项: uid=1000,gid=1000 即可指定权限, 否则默认root权限 <br>

# 安装时下载慢 (to-check)
在选择是否用网络源安装的界面, 选择"是", 但不要下一步 <br>
CTRL + ALT + F2 <br>
cd /target/etc/apt/sources.list, 手打字, 改security源 <br>
改完CTRL+ALT+F5 回到图形界面 <br>