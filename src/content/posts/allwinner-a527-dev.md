---
title: 全志A523/A527开发
published: 2026-03-06
description: '以Radxa Cubie A5E为例的全志A523/A527/T527开发记录'
image: ''
tags: [Linux, Aliwinner, ARM, SDK, Uboot, Kernel]
category: '随笔'
draft: false 
lang: ''
---
# 芯片简介
全志A527, 代号sun55iw3p1, 有4个A55小核, 最高频率1.4G, 以及4个A55大核, 最高频率1.8G, 部分芯片是2.0G <br>
GPU是单核G57, 纯凑数用的, 和其他arm cpu一样, 没有主线视频编解码驱动 <br>
coremark跑分在4.6w分左右, 可以说是非常强悍了, 且1G版本只要129, 考虑到这个扩展性(双千兆网口 + pcie2.0x1的m2插槽), 同价位基本没一个能打的 <br>

> 天生软路由圣体啊

:::info
本文以radxa-cubie-a5e 4G版本为例, 无eMMC
:::
## 硬件相关信息
根据[原理图](https://dl.radxa.com/cubie/a5e/docs/hw/v1.2/radxa_cubie_a5e_schematic_v1.2_20250113.pdf) 和A527芯片手册 <br>
全志的sdc 0-2 就是指的SoC内部MMC控制器 SD/MMC host controller(SMHC) <br>
sdc0 MMC接口, 支持SD3.0协议, 连接到TF卡 <br>
sdc1 MMC接口, 支持SDIO3.0协议, 连接到WIFI蓝牙模块 <br>
sdc2 MMC接口, 支持MultimediaCard协议, 连接到eMMC <br>

SPI flash直接通过SPI总线连接的 <br>
但TF卡并不是通过SPI总线连接到SoC <br>

### 参考文档
[第三方博客](https://luotianyi.vc/8777.html) <br>
[Linux-Sunxi](https://linux-sunxi.org/Radxa_Cubie_A5E) <br>
[全志芯片代号](https://linux-sunxi.org/Allwinner_SoC_Family) <br>

# 全志 TinaSDK 开发
## 环境准备
Ubuntu 24.04 (x64)系统下, 使用命令:
```shell
sudo apt update
sudo apt install python-is-python3 bison build-essential subversion git-core repo libncurses5-dev zlib1g-dev gawk flex quilt libssl-dev xsltproc libxml-parser-perl mercurial bzr ecj cvs unzip lib32z1 lib32z1-dev lib32stdc++6 libstdc++6  lib32ncurses-dev libncurses-dev zlib1g-dev gawk ncurses-term openssl libssl-dev linux-tools-common gperf python3-dev device-tree-compiler libgnutls28-dev libelf-dev dwarves gcc-aarch64-linux-gnu swig qemu-system-arm qemu-user-static libexpat-dev cpio busybox -y
```

## SDK下载
SDK自行前往radxa官网的mega网盘下载

:::note
这个SDK后续开发时会非常占空间, 请确保你的磁盘有足够的空间, 推荐至少120G以上
:::



```shell
mkdir tina
tar -xvf t527_tina5.0_aiot_sdk.repo.tar.gz -C tina

#repo sync命令只会对特定文件进行同步, 不在原repo的文件或文件夹不会sync
cd tina && .repo/repo/repo sync -l && cd ..

cd target/a527/ && git remote add radxa https://github.com/radxa/allwinner-target.git
git fetch radxa && git checkout -b target-a527-v1.4.6 radxa/target-a527-v1.4.6
cd ../..
cd device/config/chips/a527/ && git remote add radxa https://github.com/radxa/allwinner-device.git
git fetch radxa && git checkout -b device-a527-v1.4.6 radxa/device-a527-v1.4.6
cd ../../../..
cd bsp/ && git remote add radxa https://github.com/radxa/allwinner-bsp.git
git fetch radxa && git checkout -b cubie-aiot-v1.4.6 radxa/cubie-aiot-v1.4.6
cd ..
cd kernel/linux-5.15/ && git remote add radxa https://github.com/radxa/kernel.git
git fetch radxa && git checkout -b allwinner-aiot-linux-5.15 radxa/allwinner-aiot-linux-5.15
cd ../..
cd brandy/brandy-2.0/u-boot-2018/ && git remote add radxa https://github.com/radxa/u-boot.git
git fetch radxa && git checkout -b cubie-aiot-v1.4.6 radxa/cubie-aiot-v1.4.6
cd ../../..
```
对uboot打下补丁, 使其支持extlinux启动
```patch
--- a/pack
+++ b/pack
@@ -235,6 +235,9 @@ ${LICHEE_CHIP_CONFIG_DIR}/configs/${PACK_BOARD}/wavefile/*:${LICHEE_PACK_OUT_DIR
 ${LICHEE_CHIP_CONFIG_DIR}/configs/${PACK_BOARD}/${PACK_TYPE}/*.bmp:${LICHEE_PACK_OUT_DIR}/boot-resource/
 ${LICHEE_CHIP_CONFIG_DIR}/boot-resource/boot-resource/bat/bempty.bmp:${LICHEE_PACK_OUT_DIR}/bempty.bmp
 ${LICHEE_CHIP_CONFIG_DIR}/boot-resource/boot-resource/bat/battery_charge.bmp:${LICHEE_PACK_OUT_DIR}/battery_charge.bmp
+${LICHEE_CHIP_CONFIG_DIR}/boot-resource/extlinux:${LICHEE_PACK_OUT_DIR}/boot-resource/
+${LICHEE_PLAT_OUT}/sunxi.dtb:${LICHEE_PACK_OUT_DIR}/boot-resource/extlinux/
+${LICHEE_PLAT_OUT}/bImage:${LICHEE_PACK_OUT_DIR}/boot-resource/extlinux/Image
 )
```
在SDK顶层目录, 执行:
```shell
cd build && patch -p1 < ../pack.patch
```

## 配置并编译
```shell
source ./build/envsetup.sh

./build.sh config
./build.sh && ./build.sh pack
# 生成的镜像在 tina/out 下

# 修改内核配置
./build.sh menuconfig
```
生成的img文件为Tina Image, 非常规rawdisk镜像, 不能dd直接刷写, 需要用官方PhoenixCard或第三方[OpenixSuit](https://github.com/YuzukiTsuru/OpenixSuit)刷写 <br>
也可以使用[这个](https://gist.github.com/jernejsk/7290b340ecc636bafd52f80fa3ed9fff)脚本把tina镜像转换为常规rawdisk镜像<br>

## 其他
在 *device/config/chips/a527/configs/cubie_a5e* 下是radxa cubie A5E的开发板配置文件 <br>
env.cfg 定义最终写到第二分区(uboot env)的内容 <br>
sys_partition.fex 定义各分区的用途, 大小等 (单位是扇区数, 也就是512 Byte) <br>

:::note
根据该文件, 发现tina SDK生成的镜像的第三分区来自boot.fex, 其在out/pack_out下 <br>
boot.fex是一个指向debian下的boot.img的软链接 <br>
假设一个tina镜像的第三分区, dd出来的是p3.img, 打包过程中使用的文件是boot.fex (boot.img) <br>
两文件md5不同, p3.img后面填了0 <br>
boot.fex / p3.img 可以用安卓boot.img解包工具解包出kernel 和 initramfs <br>
kernel经md5sum计算, 即SDK中out/a527/cubie_a5e/debian/bImage <br>
ramdisk --> out/a527/kernel/staging/ramfs.cpio.gz <br>

boot.img由于未知原因, 无法解出dtb文件 <br>
:::


tina img镜像结构:  <br>
  - 第一分区之前: uboot binary
  - 第一分区: Label: Volumn     FS: vfat/fat32   未知用途 
  - 第二分区: U-Boot Env
  - 第三分区: boot.img
  - 第四分区及以后: Rootfs


## 全志平台相关知识
全志SOC会在上电后在启动介质的第16个扇区中找bootloader (16*512/1024=8KiB) <br>
从allwinner H3开始 (2014年), 新版SOC也会找第256个扇区 (256*512/1024=128KiB) (先检查第8KiB, 再检查这个) <br>
如果是从spi flash启动, 直接从第一个扇区开始写就行 <br>
## MBR与GPT
两者都属于分区表, 位于存储设备的开头, 用于储存分区信息. <br>
mbr分区表信息保存在在第一个扇区中, 以0x55 0xAA结尾. MBR只能分配至多4个分区 <br>
gpt有两个分区表, 为了兼容MBR, 其第一个扇区为Protective MBR, 不使用 <br>
第2 到 第 34 个扇区是主GPT分区表, 另外一份位于末尾, 用作备份, 一般只管主分区表即可 <br>
## 烧写/擦除 UBOOT
> 注意烧写前计算uboot大小是否会超过第一个分区起始位置

写uboot:
```shell
# 从第16个扇区块开始
dd if=u-boot-sunxi-with-spl.bin of=/dev/sdX bs=1024 seek=8
# 从第256扇区块开始
dd if=u-boot-sunxi-with-spl.bin of=/dev/sdX bs=1024 seek=128
```
> 如果使用GPT分区表, 则只能用第二种方法

:::tip
seek参数用于指定跳过of设备的块数, 如seek=8即跳过of=/dev/sdX设备前8个块 <br>
假设第一个分区起始位置为2048, 对不同分区表计算空闲大小: <br>
对于MBR分区表, 第一分区前共2048个扇区, 减去MBR自占的1个, 剩2047个扇区, 约 2047*512/1024 = 1023.5 KiB <br>
对于GPT分区表, 第一分区前共2048个扇区, 减去GPT使用的34, 剩 2014个扇区, 约 1007 KiB <br>

如果从16扇区开始写uboot: <br>
MBR: 2048-16 = 2032, 2032*512/1024 = 1016KiB <br>
GPT: 不能从16扇区开始, 与保留区域冲突 <br>

如果从256扇区开始写uboot: <br>
MBR: (2048-256)*512/1024 = 896KiB <br>
GPT: Same As MBR <br>

故uboot极限大小为 (2048-256)*512/1024 = 896KiB左右, 再大的话第一个分区就不能从2048扇区开始了 <br>
:::

擦除UBOOT (假设第一个分区起始扇区为2048)
```shell
# MBR
sudo dd if=/dev/zero of=/dev/sdX bs=512 seek=1 count=2047 conv=fdatasync status=progress
# GPT
sudo dd if=/dev/zero of=/dev/sdX bs=512 seek=34 count=2014 conv=fdatasync status=progress
```
## 启动优先级
全志芯片在上电后会首先检查第一个mmc控制器上的存储设备, 在A5E上对应TF卡槽, TF卡和SoC之间通过SDIO总线连接 <br>
如果TF卡设备存在, 就会直接在其上寻找uboot, 如果找不到, 设备会自动去尝试下一个介质, 也就是emmc. <br>
如果emmc也不能启动(国内发售设备基本都没EMMC, 所以后文删掉Emmc相关内容), 最后会进入板载的16M SPI-Flash <br>
> 从官方下载RadxaOS, 启动后运行rsetup安装uboot到spi flash后, 开发板便能从nvme固态或者USB启动了, 不过官方nvme启动的uboot不认tf卡 <br>

uboot寻找顺序: TF卡 -> SPI-Flash <br>
uboot启动后, 会寻找可用介质来获得dtb & kernel并尝试启动, 具体顺序由uboot决定<br>

## Q & A
sdk生成的tina镜像在刷写到SD卡后, 如果此时你将其插入别的Linux系统, 执行lsblk, 会发现SD卡只有 mmcblkX 这一行内容 <br>
原因是生成的镜像的GPT分区表不知道为什么出错了, 但由于GPT分区表是主从备份了的, 还是能从备份分区表中恢复内容 <br>

另外, 如果内核启动过程中出现Kernel Panic, 并且有如下内容:
```
[    7.213701] VFS: Cannot open root device "mmcblk0p4" or unknown-block(179,4): error -6
[    7.222583] Please append a correct "root=" boot option; here are the available partitions:
[    7.231944] b300        61798400 mmcblk0
[    7.231948]  driver: mmcblk
```
代表内核无法识别SD卡的分区, 需要手动纠正分区表后才能使用 <br>
恢复的步骤:
```
sudo fdisk /dev/mmcblkX
输入p查看分区表
输入w保存分区表
```
就这么简单

由于官方initrd没处理从m2启动的情况, 使用sdk生成的kernel + sdk内置的ramfs.cpio.gz组成的镜像, 刷到固态硬盘时, 一定会进initramfs的shell, 启动失败 <br>
并且, sdk kernel必须要加个initrd才能用, 无论ramdisk里是否有kernel modules, 如果带modules, 体积大了貌似会导致无法启动

如果你想要用SDK-Kernel + Debian13-Rootfs这种形式移植系统, 记得把内核模块安装到rootfs:
```shell
# 前往SDK的out/kernel/build下
sudo make modules_install ARCH=arm64 INSTALL_MOD_PATH=/mnt/rootfs
```
如果你需要Wifi, 请把SDK的 debian/overlay-firmware/wifi-firmware/aic8800/sdio 下的所有文件复制到rootfs中的/lib/firmware下, 否则只有wifi内核模块的情况下, 内核会无法加载无线设备
如果Wifi处于打开的状态, 但没连接到任何AP, 串口内会疯狂刷AICWFDBG Debug信息, 请注意.

## 制作initrd
在任意arm64机器上执行, 获取initrd, 当然你也能从别人那直接拿过来用也ok (比如ophub/kernel):
```shell
mkinitramfs -o initrd.gz
# 使用7zip解压:
7zz x initrd.gz -snld

#看需求, 把/usr/lib/modules 下的内核模块删除
#tips: 内核模块中, btrfs f2fs nfs非常占空间, 三者加起来能占80M左右
# 重新打包initrd
find . | cpio -o -H newc | gzip > ../initrd.img.gz
```

## GPU / VPU驱动
全志A527内置了专门编解码视频的VPU, 编码只支持H264, 最高4K@24FPS, 解码支持到: <br>
H264: 4K@30FPS <br>
H265: 4K@60FPS <br>
由于编码只有H264, 感觉只能当个监控机, 要到A733才有H265编码, 明明rk3568都有h265编码的, 就是只有1080P@60帧 <br>

要驱动这个VPU, 参考: <br>
[Sunxi-Cedrus](https://linux-sunxi.org/Sunxi-Cedrus) <br>
https://blog.csdn.net/weixin_45178274/article/details/139293091 <br>
https://bbs.aw-ol.com/topic/6500/t527%E7%A1%AC%E4%BB%B6%E8%A7%86%E9%A2%91%E7%BC%96%E7%A0%81-ve-vpu <br>

简单来说, VPU的驱动叫Cedrus, 对应的用户空间程序叫v4l2 (Video4Linux2) <br>

```shell
apt install autoconf automake libtool git build-essential pkg-config
git clone https://github.com/bootlin/libva-v4l2-request -b release-2019.03
cd libva-v4l2-request
# 实际我执行下面的命令总是提示找不到Libva, 以后再弄吧
./autogen.sh && make && sudo make install
```

至于GPU, <br>
https://www.cnblogs.com/embfly/articles/18995650 <br>
https://www.cnblogs.com/embfly/p/18995741 <br>

:::tip[小知识]
ffmpeg测试编解码: <br>
查看支持的编解码情况 (只代表ffmpeg软件支持, 不代表硬件支持情况) <br>
ffmpeg -hwaccels <br>
生成测试视频: 720P@60 FPS <br>
ffmpeg -f lavfi -i testsrc=duration=10:size=1280x720:rate=60 test.mp4 <br>

查看支持的编解码格式 <br>
ffmpeg -decoders | grep XXX <br>
ffmpeg -encoders | grep XXX <br>

转码: <br>
ffmpeg -hwaccel auto -i test.mp4 out.mp4 <br>
:::



# 主线uboot开发
编译出Uboot需要Arm Trusted Firmware, 即ATF, 但arm官方库里面还没添加对于A523系列芯片的支持, 所以先仿照armbian官方用的源
```shell
git clone https://github.com/jernejsk/arm-trusted-firmware
cd arm-trusted-firmware
git checkout a523-v4

make CROSS_COMPILE=aarch64-linux-gnu- PLAT=sun55i_a523 DEBUG=1 bl31
# 编译完成后终端会输出bl31的路径, 一般在: build/sun55i_a523/debug/bl31.bin
```
编译U-boot, 源: https://ftp.denx.de/pub/u-boot/ 或者: https://source.denx.de/u-boot/u-boot 或者: https://github.com/u-boot/u-boot
```shell
export BL31=/home/code/arm-trusted-firmware/build/sun55i_a523/debug/bl31.bin
export ARCH=arm64
export CROSS_COMPILE=aarch64-linux-gnu-

make radxa-cubie-a5e_defconfig

make menuconfig

make

# 完成后需要当前目录下的 u-boot-sunxi-with-spl.bin 文件
```
实测: 自编译的uboot可启动armbian官方的6.18.2系统

## 使用extlinux启动 (推荐)
```
LABEL TEST
  LINUX /kernel
  #INITRD /uInitrd
  FDT /dtb
  APPEND root=<ROOTFS_UUID> console=ttyAS0,115200n8 rw rootwait earlycon clk_ignore_unused splash loglevel=8
```

# 主线Linux内核
[Linux Sunxi](https://linux-sunxi.org/Mainline_Kernel_Howto) <br>

Linux内核源码建议前往kernel.org上下载, Github上的Kernel sub version = 0 <br>
```shell title="编译"
export ARCH=arm64
export CROSS_COMPILE=aarch64-linux-gnu-

make defconfig

make menuconfig

# make kernel image, output: arch/arm64/boot
make -j4 Image

# Kernel DTB FILE, output: arch/arm64/boot/dts
make -j4 dtbs

make -j4 modules
```

## 内核驱动相关
/lib/firmware 用于存放外设的固件, 比如WIFI固件, modem固件等. <br>
/lib/modules/$(uname -r) 存储内核模块 (即在编译时设定为 M 的模块), 不同版本内核的ko不能通用, insmod插入大概率kernel panic <br>

## 内核 dtb initrd 相关知识
猜测uboot dtb kernel (uInitrd) 都要统一<br>
内核和dtb一定要统一, 不然dtb声明的设备内核不认识也没办法 <br>
uboot本身自带dtb和ubootEnv, 在没有指定dtb时, uboot会把自带的dtb传给内核 <br>

# 其他
### 2026.03
今天发现armbian官方的uboot(也就是加了patch的主线uboot)以及Linux内核能认nvme了, 但主线还不行, 估计未来几个月就能用上nvme了
当然, 用了nvme的话, usb口就变成usb2.0了, 如果不需要的话可以改dts文件来禁用NVME <br>
:::warn
armbian官方dts有点问题, cpu节点没有设置opp和 cpu supply, 导致CPU始终以最低频率运行 <br>
手动补全后发现还是不行, 排查发现缺了CPU时钟, 主线也没有, 只能等别的大佬补全了 <br>
另外linux-sunxi上说devfreq是WIP, 只要解决这个问题, a5e基本就是完全体了 <br>
:::
感觉以后老老实实用armbian build sdk开发内核得了, 方便快捷, 能menuconfig, 随时更新, 用的时候从生成的镜像里提取外设固件即可 <br>

> tips: 主线uboot + BSP Kernel&dtb == BootFail

### Radxa-repo
rsdk的构建完全就是下载包, 再整合, 不含自己编译的部分 <br>

https://radxa-repo.github.io/bullseye/files.list <br>
内核包即: <br>
https://radxa-repo.github.io/bullseye/pool/main/l/linux-upstream/linux-image-5.15.147-13-aw2501_5.15.147-13_arm64.deb <br>
其通过 https://github.com/radxa-pkg/linux-aw2501 自动编译生成

### GPIO
A527有两个gpiochip:
  - gpiochip0: 对应原理图GPIO2, 负责 I J K L M
  - gpiochip1: 对应原理图GPIO1, 负责 B C D E F G H

不建议手算gpio编号, 直接去[Here](https://linux-sunxi.org/Radxa_Cubie_A5E)找编号 <br>

三个LED为共阳极, 给低电平亮, 红LED代表PCIE, 由PCIE设备控制, 不常用 <br>
绿LED代表电源 <br>
蓝LED代表User <br>
根据原理图, G = PL4, B = PL5 (GPIO2) <br>
根据Wiki, G = PC12, B = PC13 (GPIO1), error for sure <br>

cat /sys/kernel/debug/gpio, 显示绿灯是356, 蓝灯357 <br>
另外gpiochip1负责 0 - 351, gpiochip0负责 352 - 415 <br>
前者负责的GPIO编号是能和wiki上 40 Pin对应的, 但后面的比计算所得的编号多352, 所以 G/B = 4/5 <br>
dts文件里也是这个数, 但最终尝试export时会失败, 原因未知 <br>