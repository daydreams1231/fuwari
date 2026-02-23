---
title: ArchLinux 安装记录
published: 2025-12-29
description: '一次安装ArchLinux的记录'
image: ''
tags: [Linux]
category: '随笔'
draft: false 
lang: ''
---

# Before DIY:
使用源: mirrors.aliyun.com <br>
UEFI系统 <br>
一切操作推荐通过远程SSH再进行, 方便粘贴命令

# Start
```shell
# 禁用Reflector
systemctl stop reflector.service
# 检查是否为EFI启动模式
ls /sys/firmware/efi/efivars
# 检查网络
ip a
# 一般如果使用有线连接时不会没IP的, 如果使用无线连接, 参考以下步骤:
iwctl
device list
station wlan0 scan
station wlan0 get-networks
station wlan0 connect WIFI_NAME
exit

# test
ping www.baidu.com
```

# 配置NTP时钟
```
timedatectl set-ntp true
timedatectl status
```

# 换源
nano /etc/pacman.d/mirrorlist <br>
保证最前面是:
Server = https://mirrors.aliyun.com/archlinux/$repo/os/$arch

# 磁盘分区
cfdisk /dev/sda <br>
EFI分区1G, 以后可能要做多系统或者换别的引导 <br>
swap按情况给, 如果不是长时间开机或者内存很小, 建议1G即可 <br>
```shell
# 格式化EFI分区
mkfs.fat -F32 /dev/sda1
# 格式化rootfs, 使用ext4 / btrfs 均可, 如果用不到特定文件系统的高级特性建议ext4即可
mkfs.btrfs -L rootfs /dev/sda2
# SWAP
mkswap /dev/sda3
```

# 挂载分区
```shell
# 仅btrfs系统需要加入 -o compress=zstd
mount -o compress=zstd /dev/sda2 /mnt
mkdir -p /mnt/boot && mount /dev/sda1 /mnt/boot
swapon /dev/sda3
````

# 安装系统
```shell
# 基本系统
pacman -Sy archlinux-keyring && pacstrap /mnt base base-devel linux linux-firmware
# 额外软件
pacstrap /mnt networkmanager vim sudo zsh zsh-completions btrfs-progs nano dosfstools
systemctl enable NetworkManager
```

# 完成安装
```shell
genfstab -U /mnt > /mnt/etc/fstab

arch-chroot /mnt

# 改root密码
passwd

echo "ArchLinux" > /etc/hostname

ln -sf /usr/share/zoneinfo/Asia/Shanghai /etc/localtime && hwclock --systohc

# 删去en_US.UTF-8 UTF-8
# zh_CN.UTF-8 UTF-8
vim /etc/locale.gen 

locale-gen && echo 'LANG=en_US.UTF-8'  > /etc/locale.conf

# CPU微码
pacman -S intel-ucode # amd-ucode

pacman -S grub efibootmgr os-prober
grub-install --target=x86_64-efi --efi-directory=/boot --bootloader-id=ARCH

grub-mkconfig -o /boot/grub/grub.cfg

exit

umount -R /mnt && reboot
```
