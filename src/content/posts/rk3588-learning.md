---
title: rk3588嵌入式学习
published: 2026-01-10
description: '以Radxa Rock5B为例, 学习arm64嵌入式开发'
image: ''
tags: [arm, Linux, rockchip, uboot, kernel]
category: ''
draft: false 
lang: ''
---

# 前言
最近入手了一个RK3588的开发板 Radxa Rock 5B, 版本是1.42, 内存8G <br>
以此文记录rockchip瑞芯微的学习日志

# 准备工作
下载这两个文件并安装:
  - [Driver](https://dl.radxa.com/tools/windows/DriverAssitant_v5.0.zip)
  - [RK dev tool](https://dl.radxa.com/tools/windows/RKDevTool_Release_v2.96-20221121.rar)

前者是瑞芯微开发必备的驱动, 后者是瑞芯微开发工具, 可用于刷系统, 救砖等等操作. <br>
如果你以前捡过一些瑞芯微的arm板子, 如OEC-T、各种基于rk3399的工控板, 对这个应该不陌生 <br>

# 概念及 Q&A
## 启动流程及优先级
[瑞芯微启动流程](https://opensource.rock-chips.com/wiki_Boot_option#Boot_flow). <br>
[ophub Discussions](https://github.com/ophub/amlogic-s9xxx-armbian/discussions/1634) <br>
[FriendlyElecWiki](https://wiki.friendlyelec.com/wiki/index.php?title=Template:RockchipBootPriority/zh&redirect=no) <br>
芯片上电 -> 运行芯片内部MaskRom(Boot rom) -> Loader1区域(bl2) -> Loader2区域(bl33) -> boot.img -> rootfs <br>
简化一下就是 bootrom -> SPL -> uboot <br>
除去Maskrom阶段, 后面的步骤都是在芯片外部的存储介质上进行的, 如SPI-NOR Flash, EMMC, SD/TF <br>

Loader1区域: idbloader.img, 分为传统的ubootSPL/TPL或者瑞芯微官方的miniloader, 这个阶段一般用来初始化外设, 在瑞芯微中, TPL负责初始化内存 <br>
Loader2区域: uboot.itb, 真正的uboot程序在此运行 <br>

> 实际上, 真正的启动顺序是 maskrom -> tpl -> maskrom -> spl

优先级(从上至下): <br>
  - SPI NOR
  - SPI NAND
  - EMMC
  - MMC device (sd/tf card)
Boot Rom结束后, 进入bl2阶段, 芯片会依次照此顺序寻找idbloader.img, 如果都没找到就初始化USB. 但注意板子不会自己进maskrom模式, 此时串口也没任何输出 <br>
bl2结束后, uboot接管启动流程, 至于是从哪个介质加载uboot就不清楚了, 一般idbloader和uboot都是写在一个介质上的

TF卡里有radxa os固件, 貌似是Miniloader的
spi flash空
上电启动radxa os, 按住rec无法进loader

## 使用RK Dev Tool时使用的Loader是什么东西?
RK的Loader有两种: miniloader和uboot SPL/TPL, 两者都包含 ddrBin usbplug <br>
在maskrom模式下, 刷写系统时, 第一行的Loader即上述的miniloader或者ubootSPL/TPL, 其运行在内存中, 用于 "和 rkdevtool 通讯以及写 flash 等操作" <br>

> idbloader是特殊的loader, 由上面的Loader加上ddrBin, 再按IDB格式打包而成

上述的Loader都属于BL2阶段 <br>

> 和全志对比一下, idbloader.img和u-boot.itb一起对应全志的u-boot-with-spl.bin

> 在Mask Rom模式下, 只能进行对于loader的烧写

如果是emmc或者sd卡, 里面有uboot, 此时按recovery按键上电是不会进loader模式的
sd卡, 刷入自编译Uboot(u-boot-rockchip.bin), 无法进Loader模式

sd卡, 刷入radxa os, 无法进loader, 且串口不弹什么Uboot vXXX

## RK设备的Loader模式 和 Maskrom模式
Maskrom模式一般用于刷写固件系统到存储设备里, 一般要选择一个Loader和一个系统镜像, 选的Loader是临时运行在内存中的, 专门用来把系统镜像刷到存储设备里 <br>
Loader模式: 如果有板载存储设备, 且其有miniloader / uboot SPL/TPL, 此时按REC键上电即可进入Loader模式, 该模式一般用于对某一个分区进行刷写, 如更换rootfs <br>

# Uboot
RK官方的Uboot v2017.09: https://github.com/rockchip-linux/u-boot <br>
主线Uboot: https://gitlab.denx.de/u-boot/u-boot.git <br>
Uboot依赖: git clone https://github.com/rockchip-linux/rkbin.git <br>
rkbin存储了一些RK芯片所需的TPL ATF等等资源, 是闭源的 <br>

后文均以主线Uboot为例. <br>
```shell
# 或者去 https://ftp.denx.de/pub/u-boot/ 下载了再解压也行
git clone https://gitlab.denx.de/u-boot/u-boot.git
git clone https://github.com/rockchip-linux/rkbin.git

cd u-boot
git checkout vXXXX.XX

# 根据Soc及内存, 选择合适的TPL
export ROCKCHIP_TPL=../rkbin/bin/rk35/rk3588_ddr_lp4_2112MHz_lp5_2400MHz_v1.19.bin
# 指定 ATF, 这里直接用rkbin提供的闭源ATF, 如果需要开源的ATF也可去 https://github.com/ARM-software/arm-trusted-firmware 自行构建
export BL31=../rkbin/bin/rk35/rk3588_bl31_v1.51.elf

export ARCH=arm64
export CROSS_COMPILE=aarch64-linux-gnu-

# 查找有无自己板子的配置, 如果很不幸没有, 那得自己填一堆参数, 以后有时间可以对比一下同Soc不同板子配置的差异 & 不同SoC差异
# 比如 Rock 5B 就是 rock5b-rk3588_defconfig
ls configs/*rkXXXX*
make XXX_defconfig

make menuconfig

make -j8
```
编译产物:
  - idbloader.img: 刷到MMC设备里的uboot-SPL/TPL
  - idbloader-spi.img: 同上, 但是刷到SPI Flash设备, 一般不用
  - u-boot.itb: 刷到MMC设备里的uboot二进制文件
  - u-boot.img: 猜测是给miniloader打包用的
  - u-boot-rockchip.bin (这个有9942528 bytes, 即19419个扇区, 9.5MiB): 同idbloader.img
  - u-boot-rockchip-spi.bin (1979904 bytes, 3867 sectors, 1.9MiB): 同idbloader-spi.img

## 刷写
若是准备直接刷到SPI Flash里面, 使用: `u-boot-rockchip-spi.bin` 这个文件, 直接dd到SPI里面, 不需要任何偏移, 或者直接用RK Dev tool刷也行 <br>
下面的方法以刷到SD卡为例, emmc的等几天再加
```shell
# 方法1
# 这个 u-boot-rockchip.bin 异常大, 注意别把第一个分区覆盖了
sudo dd if=u-boot-rockchip.bin of=/dev/XXX seek=64 status=progress
sync

# 方法2
sudo dd if=idbloader.img of=/dev/XXX seek=64
sudo dd if=u-boot.itb of=/dev/XXX seek=16384

# 以我编译的uboot为例, idbloader.img大小为424个扇区, u-boot.utb大小为3099个扇区

# todo: rk wiki烧boot rootfs时候分区是32768 262144, 不确定这个是不是写死了的
```
## bootargs
以Rock5B为例, 其console=ttyS2,1500000, 不同开发板这个可能不同
```text
LABEL Test
  LINUX /kernel
  INITRD /initrd
  FDT /dtb
  APPEND root=/dev/xxx earlyprintk console=ttyS2,1500000 rw rootwait rootfstype=ext4 init=/sbin/init
```
# Kernel
瑞芯微BSP内核: https://github.com/rockchip-linux/kernel <br>
armbian从BSP移植的内核: https://github.com/armbian/linux-rockchip <br>
对于rk3588, 一般使用官方内核或者移植内核, 以求最大化利用芯片的各个能力, 如PCIE GPU VPU NPU等 <br>
无论移植的内核, 还是BSP内核, 其内核小版本总是落后于主线一截. <br>
对于其他rk芯片, 可以选择和3588一样用BSP, 或者用自编译的主线内核, 不过那样有点浪费 <br>

:::NOTE
armbian移植内核实测支持PCIE <br>
ophub的通用stable内核也支持PCIE
:::

> 截至2026.1, rk官方的内核有: 4.4, 4.19, 5.10, 6.1, 6.6, 其他移植的内核只有5.10和6.1, 当前主线最新LTS为6.18

> Linux 7.0 已支持 RK3588 VPU驱动

```shell title="编译"
export ARCH=arm64
export CROSS_COMPILE=aarch64-linux-gnu-

make defconfig

# make kernel image, output: arch/arm64/boot
make -j4 Image

# Kernel DTB FILE, output: arch/arm64/boot/dts
make -j4 dtbs

make -j4 modules
```

内核具体配置教程以后有时间再弄 <br>
编译完内核模块后记得安装到开发板的/lib/modules <br>

# 其他
Red LED = GPIO4_C5 (149) chip4 - 21
Blue LED = GPIO0_B7 (15) chip0 - 15
绿灯无法控制, 只要上电一定会亮
## TypeC作为电源输入时, 查看协商的电压
```shell
awk '{printf ("%0.2f\n",$1/172.5); }' </sys/devices/iio_sysfs_trigger/subsystem/devices/iio\:device0/in_voltage6_raw
```

## 风扇
Rock5B的风扇不是常规意义上的PWM风扇, 而是一个普通的双线风扇, 一般PWM风扇内会有一个PWM控制器, 由系统通过PWM信号控制风扇转速, 而Rock5B的风扇则是直接接在GPIO上, 通过控制GPIO的电压大小来控制风扇速度, 类似于PWM风扇 <br>

[Radxa Wiki](https://docs.radxa.com/rock5/rock5b/getting-started/interface-usage/fan) <br>

一般DTS文件内已配置PWM GPIO, 不需要在系统内额外配置GPIO为PWM输出 <br>
总结, 对于Rock5B, 其风扇控制命令:
```shell
echo FAN_SPEED_NUMBER | sudo tee /sys/devices/platform/pwm-fan/hwmon/hwmon*/pwm1
```
:::info
FAN_SPEED_NUMBER取值范围为0-255, 0为风扇停止, 255为全速, 其他数值代表不同的转速 <br>
但需注意, 其值与风扇强相关, 同一个值在不同的风扇上可能会有不同的效果 <br>
:::

假设你的风扇在0-255范围内, 均有不同的转速, 即理想情况, 可通过用户脚本来实现温控:
```shell
#!/bin/bash
# 读取当前 CPU 温度（单位：摄氏度）
get_temp() {
    cat /sys/class/thermal/thermal_zone0/temp | awk '{print int($1/1000)}'
}
# 设置风扇转速 (0-255)
set_fan_speed() {
    echo "$1" | sudo tee /sys/devices/platform/pwm-fan/hwmon/hwmon8/pwm1 > /dev/null
}
# 主逻辑循环
# 具体控制逻辑见 if-else if部分, 如果看不懂可以问AI
while true; do
    temp=$(get_temp)
    echo "Temp: $temp"
    if [[ $temp -gt 47 ]]; then
        set_fan_speed 255
    fi
    sleep 15
done
```