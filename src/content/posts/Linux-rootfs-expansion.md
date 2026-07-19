---
title: Linux扩容
published: 2026-07-19
description: '为Linux系统扩展根分区空间'
image: ''
tags: [Linux, Debian, Openwrt]
category: '教程'
draft: false 
lang: 'zh_CN'
---

# 须知
常规Linux系统用于定义分区用途及挂载路径的文件为 /etc/fstab, 但类openwrt系统中, /etc/fstab是空的(除了注释外), 真正有效的是 /etc/config/fstab <br>
如果你的Linux系统是通过指定UUID来挂载rootfs的, 在删除 + 重新创建分区后, 分区的PART UUID会发生变化, 但UUID不变, 自己注意修改 grub等 的启动配置文件 <br>
# 准备工具
请以root用户确认系统中存在如下命令:
```shell
# Debian自带, Openwrt需安装lsblk软件包
lsblk
# Linux自带
df
# Debian自带, Openwrt安装losetup软件包
losetup
# 这个格式化命令应该都有, 如果没有就安装e2fsprogs
mkfs.ext4
# Debian/Openwrt 安装f2fs-tools
mkfs.f2fs
resize.f2fs
# Openwrt固件安装resize2fs软件包
resize2fs
```

# 一些教程
## 如何把一个物理分区划分为多个逻辑分区
尽管linux支持将一个子分区再次分区, 但那样就看着太怪了, 一般使用losetup工具来将一个分区分为多个部分
> 方法不唯一, 类似的实现还有 LVM, dmsetup等, 以及支持子卷的文件系统等
```shell
# 假设目标分区是 /dev/mmcblk0p2
losetup -o OFFSET_in_Bytes_start --sizelimit SIZE_in_Bytes -f /dev/mmcblk0p2
```
该命令会从mmcblk2p2的 OFFSET_in_Bytes_start 作为起始位置, 一共划分 SIZE_in_Bytes 大小的空间

:::tips[learning]
在openwrt中的make menuconfig阶段, 一般会指定rootfs的大小, 在编译完成后, 内核等文件进入最终镜像的第一分区, 其他文件被squashfs打包, 可能还会压缩下, 生成一个root.squashfs镜像文件 <br>
之后, 根据menuconfig配置的`CONFIG_TARGET_ROOTFS_PARTSIZE`, 减去这个镜像文件的大小, 剩下的空间就是overlayfs的upperDir了, 会被贴上rootfs_data标签 <br>

首次启动时, 内核看到 /dev/mmcblk0p2 的开头有一个有效的 SquashFS 超级块, 就会把它当作一个 squashfs 文件系统挂载. **SquashFS 格式本身记录了数据到哪里结束**, 内核只会读到镜像结束位置. <br>
之后, 使用 `losetup -o OFFSET /dev/mmcblk0p2`创建一个loop设备, 其会作为overlayfs upperDir. 这个squashfs的大小, 即该命令中的OFFSET值.
:::

## 在更改分区参数后, 更新UUID至grub
x86_64系统:
```shell
# Update GRUB configuration
ROOT_BLK="$(readlink -f /sys/dev/block/"$(awk -e '$9=="/dev/root"{print $3}' /proc/self/mountinfo)")"
ROOT_DISK="/dev/$(basename "${ROOT_BLK%/*}")"
ROOT_DEV="/dev/${ROOT_BLK##*/}"
ROOT_UUID="$(partx -g -o UUID "${ROOT_DEV}" "${ROOT_DISK}")"
sed -i -r -e "s|(PARTUUID=)\S+|\1${ROOT_UUID}|g" /boot/grub/grub.cfg
```
arm类系统(尤其是硬路由)一般改不了bootloader相关参数, 并且相当一部分都是指定root=/dev/xxx, 而非UUID


# 通用扩容方法(指定rootfs分区)
在磁盘的空闲空间新建一个分区作为rootfs根分区, 把原来系统里的文件全复制过去, 即强制指定rootfs, 不启用overlayFS <br>

手动创建一个新的大分区, 在格式化完之后, 前往*系统* -> *挂载点* 里添加新分区为根分区, 保存并应用后执行下面命令

```shell
mkdir -p /tmp/introot
mkdir -p /tmp/extroot
mount --bind / /tmp/introot
mount 新扩容分区 /tmp/extroot
tar -C /tmp/introot -cvf - . | tar -C /tmp/extroot -xf -
umount /tmp/introot
umount /tmp/extroot
reboot
```

## OpenWRT *系统* 下没有*挂载点*
opkg install block-mount


# 其他 & 例子
## Radxa A5E 第三方OpenWRT固件 ( OverlayFS(lowerDir:ext4 + upperDir:f2fs) )
```shell
fdisk /dev/mmcblk0 # p n
mkfs.f2fs 新分区
mount 新分区 /mnt/tmp --mkdir
cp -r /overlay/* /mnt/tmp
```
去*挂载点*, 将新分区设置为*外部Overlay*

## NN6000 OpenWRT
```shell
# df -hT
Filesystem           Type            Size      Used Available Use% Mounted on
/dev/root            squashfs       42.5M     42.5M         0 100% /rom
tmpfs                tmpfs         950.6M    520.0K    950.1M   0% /tmp
/dev/loop0           f2fs            2.0G    283.1M      1.7G  14% /overlay
overlayfs:/overlay   overlay         2.0G    283.1M      1.7G  14% /
tmpfs                tmpfs         512.0K         0    512.0K   0% /dev
/dev/mmcblk0p12      ext4            5.0G      1.3M      4.8G   0% /mnt/mmcblk0p12
# lsblk
NAME         MAJ:MIN RM  SIZE RO TYPE MOUNTPOINTS
loop0          7:0    0    2G  0 loop /overlay
mmcblk0      179:0    0  7.3G  0 disk
├─mmcblk0p1  179:1    0  768K  0 part
├─mmcblk0p2  179:2    0  256K  0 part
├─mmcblk0p3  179:3    0  1.8M  0 part
├─mmcblk0p4  179:4    0  256K  0 part
├─mmcblk0p5  179:5    0  256K  0 part
├─mmcblk0p6  179:6    0  256K  0 part
├─mmcblk0p7  179:7    0  256K  0 part
├─mmcblk0p8  179:8    0  640K  0 part
├─mmcblk0p9  179:9    0  256K  0 part
├─mmcblk0p10 179:10   0   12M  0 part
├─mmcblk0p11 179:11   0    2G  0 part /rom
└─mmcblk0p12 179:12   0  5.2G  0 part /mnt/mmcblk0p12
mmcblk0boot0 179:32   0    4M  1 disk
mmcblk0boot1 179:64   0    4M  1 disk
```
这个系统的rootfs全放在 /dev/mmcblk0p11 上, 但使用OverlayFS, 将其分为了两个逻辑分区, 第一个分区是只读的squashfs, 即lowerDir, 大小42.5M, 第二个分区是可RW的f2fs作为upperDir, 用于存放修改或新产生的文件 <br>

:::tips
如果有/rom挂载, 则一般为overlayFS, 其:
lowerDir = /rom (ro)
upperDir = /overlay 或者 /rom/overlay (rw)
mergeDir = /
这种情况要扩的目标分区是upperDir对应的设备
:::

要实现扩容, 有几种方法:
  - 一: 把 mmcblk0p12 作为新的overlayfs的upperDir. 先格式化这个分区, 再把它挂载到任意一个临时目录, 后复制原 /overlay 下的所有文件到这个临时目录, 再直接去luci web里设置该分区的用途为"作为外部Overlay使用"即可. 但这种方法缺点是原来的 mmcblk0p11 的第二个逻辑分区仍然存在, 且占用着空间. 好处则是后期可随时回滚到扩容前的系统.
  - 二: 删掉 mmcblk0p12, 再进 fdisk 修改 p11 分区的结束位置, 但注意不要修改分区标签. 这一步是修改分区表中这个分区的大小, 但离生效还要让文件系统也刷新一下. 这个例子中, upperDir是f2fs, 其不支持在线调整文件系统大小, 且你也不方便把emmc拆下来, 拿到别的设备上去刷新文件系统. 故只能创建一个 upperDir 的"替身", 再对其进行操作: 记录 *losetup* 输出的OFFSET值, 使用 *losetup -f -o OFFSET /dev/mmcblk2p12* 创建 "替身", OFFSET即先前输出的值, 这一步会创建一个新的回环设备(比如/dev/loop1), 之后再 *resize.f2fs /dev/loop1* 即可

:::tips
有的人可能发现了, /etc/config/fstab 中关于 /overlay 的条目是未启用的状态, 即使删除它, 重启后系统仍然能正常挂载 /overlay <br>

实际上, openwrt在早期启动阶段, 会自动寻找LABEL=rootfs_data的内部硬盘设备, 会先挂载它到/tmp/overlay, 再尝试读取其中的 /etc/config/fstab, 如果 fstab 中没有有效的 /overlay 挂载点(或者对应设备不存在), 则最开始挂载的rootfs_data设备会成为 /overlay; 反之, 则会将fstab中指定的设备挂载为/overlay <br>

在/overlay挂载完成后, 所有修改和写入等操作均发生在/overlay对应的设备上, 如果你现在用的是外部存储设备(比如U盘)作为/overlay, 即使你手动 /etc/config/fstab 删除/overlay相关条目, 重启后U盘还是会被挂载. <br>
毕竟内部硬盘里的rootfs_data分区里面的fstab还是指定外部设备为overlay. 解决方法为直接移除外部存储设备即可 <br>
:::

:::warn
openwrt sysupgrade原理即把rootfs_data(/overlay)的内容复制到内存中, 更新完后再写回去, 如果更新失败会导致/overlay数据丢失. 故一般不推荐通过sysupgrade更新系统, 一般更新下软件包即可
:::
## OpenWRT官方x86 ext4固件
```shell
# df -hT
Filesystem           Type            Size      Used Available Use% Mounted on
/dev/root            ext4           98.3M     23.8M     72.4M  25% /
tmpfs                tmpfs         478.9M    224.0K    478.6M   0% /tmp
/dev/sda1            vfat           16.0M      6.2M      9.8M  39% /boot
/dev/sda1            vfat           16.0M      6.2M      9.8M  39% /boot
tmpfs                tmpfs         512.0K         0    512.0K   0% /dev
# lsblk
NAME     MAJ:MIN RM  SIZE RO TYPE MOUNTPOINTS
sda        8:0    0    1G  0 disk
├─sda1     8:1    0   16M  0 part /boot
│                                 /boot
└─sda2     8:2    0  104M  0 part /
```
rootfs存放在 /dev/sda2 中, 文件系统为 ext4, 整个系统未使用 OverlayFS <br>

要扩容的话很简单: 先fdisk修改分区表中 /dev/sda2 分区的大小, 注意不要移除标签, 注意修改grub中PARTUUID, 后 *resize2fs -f /dev/sda2* 即可 <br>
> 如果出现 'resize2fs: Invalid argument While trying to add group #1' 报错, 参考文末 Q&A

:::note
如果/dev/sda2 (即/dev/root)是f2fs, 由于其不支持在线扩容, 且该系统并未使用overlayfs, 可直接创建/dev/sda2的替身:
```shell
losetup /dev/loop0 /dev/sda2 2
```
再 *resize.f2fs /dev/loop0*
:::
## OpenWRT官方x86 squashfs固件
```shell
# df -hT
Filesystem           Type            Size      Used Available Use% Mounted on
/dev/root            squashfs        6.0M      6.0M         0 100% /rom
tmpfs                tmpfs         478.6M      1.8M    476.8M   0% /tmp
/dev/loop0           ext4           86.4M      1.7M     77.9M   2% /overlay
overlayfs:/overlay   overlay        86.4M      1.7M     77.9M   2% /
/dev/sda1            vfat           16.0M      6.2M      9.8M  39% /boot
/dev/sda1            vfat           16.0M      6.2M      9.8M  39% /boot
tmpfs                tmpfs         512.0K         0    512.0K   0% /dev

# lsblk
NAME     MAJ:MIN RM  SIZE RO TYPE MOUNTPOINTS
loop0      7:0    0 98.1M  0 loop /overlay
sda        8:0    0    1G  0 disk
├─sda1     8:1    0   16M  0 part /boot
│                                 /boot
└─sda2     8:2    0  104M  0 part /rom
```
和nn6000类似, /dev/sda2存储rootfs, 并通过losetup划分为两个逻辑部分, 前面是只读的squashfs, 后面是可写的ext4 <br>
这里就不重复赘述了 <br>

# BTRFS扩容
btrfs filesystem resize max /

# SquashFS / OverlayFS 技术细节
SquashFS是一个只读文件系统，仅此而已

OverlayFS和常规文件系统一样是可RWE的, 但其存在upperDir(读写) lowerDir(只读) mergeDir(最终挂载点), 其更像一个整合其他文件系统的文件系统

用户最终看到的是mergeDir文件夹

mergeDir会显示lowerDir + upperDir的总和

文件/文件夹优先显示UpperDir里的，如果没有就显示LowerDir里的，在修改文件后，新文件保存在upperDir

lowerDir和upperDir两个目录存在同名文件时，lowerDir的文件将会被隐藏，用户只能操作upperDir的文件。

如果存在同名目录，那么lowerdir和upperdir目录中的内容将会合并。

lowerDir是只读的, OP中一般格式化为squashFS, 而upperDir为用户修改后的文件保存目录

upperDir下会创建一个upper文件夹, 里面是所有新文件

/dev/root是一个指向根文件系统的链接，在挂载完成后就被删除了，所以在启动完成后/dev下没有root文件

# CMCC RAX3000M
其存储介质为spi nand, 在linux系统中显示为/dev/mtdXX, 如果用df -hT发现这些设备并未挂载到任何目录，而是一个叫/dev/ubiXXX的挂载到/rom，ubifs是运行在spi nand介质上的文件系统，一个mtd分区对应一个ubiX
使用 *cat /proc/mtd* 来查看哪个mtd分区用作ubifs
```
dev:    size   erasesize  name
mtd0: 08000000 00020000 "spi0.0"
mtd1: 00100000 00020000 "BL2"
mtd2: 00080000 00020000 "u-boot-env"
mtd3: 00200000 00020000 "Factory"
mtd4: 00200000 00020000 "FIP"
mtd5: 07200000 00020000 "ubi"
```
可以看到是mtd5分区是用作ubifs的, 再结合 *df -hT* 和 *lsblk* 的内容:
```
# df -hT
/dev/ubi0_2          ubifs          83.7M     53.3M     26.0M  67% /rom/overlay
# lsblk
NAME        MAJ:MIN RM  SIZE RO TYPE MOUNTPOINT
mtdblock0    31:0    0  128M  0 disk
mtdblock1    31:1    0    1M  0 disk
mtdblock2    31:2    0  512K  0 disk
mtdblock3    31:3    0    2M  0 disk
mtdblock4    31:4    0    2M  0 disk
mtdblock5    31:5    0  114M  0 disk
zram0       252:0    0  239M  0 disk [SWAP]
ubiblock0_1 253:0    0 11.6M  0 disk /rom
```
ubifs将spi nand抽象为两部分，其一为ubiblock0_1 (/dev/ubi0_1), 挂载为/rom, 为overlayFS的lowerDir, 其二为/dev/ubi0_2, 挂载为/rom/overlay, 为overlayFS的upperDir

# Q & A
## resize2fs -f xxx 时报错:
/dev/sda2 挂载到 / :
```
resize2fs 1.47.3 (8-Jul-2025)
Filesystem at /dev/sda2 is mounted on /; on-line resizing required
old_desc_blocks = 1, new_desc_blocks = 1
Performing an on-line resize of /dev/sda2 to 257728 (4k) blocks.
resize2fs: Invalid argument While trying to add group #1
```
解决:
```shell
mount -o remount,ro /

#Remove reserved GDT blocks
# 可能会提示运行 e2fsck -f /dev/sda2, 照做, 但完了先别重启
tune2fs -O^resize_inode /dev/sda2

#Fix part, answer yes to remove GDT blocks remnants
fsck.ext4 /dev/sda2 

resize2fs -f /dev/sda2
```