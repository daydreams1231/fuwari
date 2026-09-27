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
## 瑞芯微 Loader模式 与 Maskrom模式
Maskrom模式是瑞芯微的底层刷机模式, 可类比高通设备的9008. 可用于刷写镜像文件到板载存储里, 例如刷armbian到emmc, 刷spi image到spi-nor, 也可清除某个硬盘的数据. <br>
Loader模式: 如果有板载存储设备, 且其有miniloader / uboot SPL/TPL, 此时按REC键上电即可进入Loader模式, 该模式一般用于对某一个分区进行刷写, 如更换rootfs <br>

## 启动流程及优先级
[瑞芯微启动流程](https://opensource.rock-chips.com/wiki_Boot_option#Boot_flow). <br>
[ophub Discussions](https://github.com/ophub/amlogic-s9xxx-armbian/discussions/1634) <br>
[FriendlyElecWiki](https://wiki.friendlyelec.com/wiki/index.php?title=Template:RockchipBootPriority/zh&redirect=no) <br>
在芯片上电后, 会自动执行内部的MaskRom代码, 之后跳转到外部硬盘, 依次运行 bl2 和 bl33, 即bootloader, 一般为uboot. <br
如果外部硬盘没bootloader, 会自动进入maskrom模式. <br>
uboot在初始化完成后会找内核, 启动内核, 挂载rootfs等等 <br>
:::note
如果有多个存储设备, 比如 spi-nor 和 emmc同时存在, 如果uboot烧写在spi-nor上, uboot在寻找内核以及dtb等文件时, 具体会从哪个设备上找, 取决于uboot env配置
:::

Loader1区域(bl2): idbloader.img, 分为传统的ubootSPL/TPL或者瑞芯微官方的miniloader, 这个阶段一般用来初始化外设, 在瑞芯微中, TPL负责初始化内存 <br>
Loader2区域(bl33): uboot.itb, 真正的uboot程序在此运行 <br>


优先级(从上至下): <br>
  - SPI NOR
  - SPI NAND
  - EMMC
  - MMC device (sd/tf card)

## 使用RK Dev Tool时使用的Loader是什么东西?
RK的Loader有两种: miniloader和uboot SPL/TPL, 两者都包含 ddrBin usbplug<br>
在maskrom模式下, 刷写系统时, 第一行的Loader即上述的miniloader或者ubootSPL/TPL, 其运行在内存中, 用于 "和 rkdevtool 通讯以及写 flash 等操作" <br>

> idbloader是特殊的loader, 由上面的Loader加上ddrBin, 再按IDB格式打包而成

上述的Loader都属于BL2阶段 <br>

> 和全志对比一下, idbloader.img和u-boot.itb一起对应全志的u-boot-with-spl.bin

# 疑难解答
### 无法启动
使用 armbian 官方 desktop + vendor内核的镜像, 刷到 TF 卡中 <br>
串口日志:
```text
INFO:    Preloader serial: 2
NOTICE:  BL31: v2.3():v2.3-868-g040d2de11:derrick.huang, fwver: v1.48
NOTICE:  BL31: Built : 15:02:44, Dec 19 2024
INFO:    spec: 0x1
INFO:    code: 0x88
INFO:    ext 32k is not valid
INFO:    ddr: stride-en 4CH
INFO:    GICv3 without legacy support detected.
INFO:    ARM GICv3 driver initialized in EL3
INFO:    valid_cpu_msk=0xff bcore0_rst = 0x0, bcore1_rst = 0x0
INFO:    l3 cache partition cfg-0
INFO:    system boots from cpu-hwid-0
INFO:    disable memory repair
INFO:    idle_st=0x21fff, pd_st=0x11fff9, repair_st=0xfff70001
INFO:    dfs DDR fsp_params[0].freq_mhz= 2112MHz
INFO:    dfs DDR fsp_params[1].freq_mhz= 528MHz
INFO:    dfs DDR fsp_params[2].freq_mhz= 1068MHz
INFO:    dfs DDR fsp_params[3].freq_mhz= 1560MHz
INFO:    BL31: Initialising Exception Handling Framework
INFO:    BL31: Initializing runtime services
WARNING: No OPTEE provided by BL2 boot loader, Booting device without OPTEE initialization. SMC`s destined for OPTEE will return SMC_UNK
ERROR:   Error initializing runtime service opteed_fast
INFO:    BL31: Preparing for EL3 exit to normal world
INFO:    Entry point address = 0x200000
INFO:    SPSR = 0x3c9
```
原因: uboot内的 BL2 没有提供 OPTEE（可信执行环境）镜像，因此无法初始化 opteed_fast 运行时服务. 一般情况下不影响启动, 但armbian vendor内核镜像会长时间卡在这里, 无法得知启动情况, 不过从功耗来看应该是没启动的; current内核的镜像只会卡在这里一会, 等一会就进系统了
解决方案: 换个系统镜像
:::note
armbian官方desktop + current内核的镜像尽管能启动, 但实际上只有CLI, 并没有桌面, 还得自己手动装个桌面才行, 而且由于某些Bug, 系统上电后还是进cli(已设置boot target为graphical)
:::
### 从Nvme启动时串口没uboot日志
原因: spi-nor里的uboot是官方版本的, 你需要换成其他uboot, 比如ophub提供的 [SPI Image](https://github.com/ophub/u-boot/tree/main/u-boot/rockchip/rock5b/spi), 或者从TF/EMMC启动armbian官方系统后再armbian-config来安装SPI Image
> 内核日志由 uboot 传递的 cmdline 决定是否输出


# 编译Uboot
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

# 以我编译的uboot为例, idbloader.img大小为424个扇区, u-boot.itb大小为3099个扇区

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
Rock5B的风扇不是常规意义上的PWM风扇, 而是一个普通的双线风扇, 一般PWM风扇内会有一个PWM控制器, 由系统通过PWM信号控制风扇转速, 而Rock5B的风扇则是直接接在GPIO上, 通过PWM控制GPIO的电压大小来控制风扇速度, 类似于PWM风扇 <br>

[Radxa Wiki](https://docs.radxa.com/rock5/rock5b/getting-started/interface-usage/fan) <br>

一般DTS文件内已配置PWM GPIO, 不需要在系统内额外配置GPIO为PWM输出 <br>
对于Rock5B, 其风扇控制命令:
```shell
echo FAN_SPEED_NUMBER | sudo tee /sys/devices/platform/pwm-fan/hwmon/hwmon*/pwm1
```
:::info
FAN_SPEED_NUMBER取值范围为0-255, 0为风扇停止, 255为全速, 其他数值代表不同的转速 <br>
但需注意, 其值与风扇强相关, 同一个值在不同的风扇上可能会有不同的效果 <br>
:::

一般, 内核已配置了该PWM风扇控速策略为step-wise, 如需手动更改, 前往 /sys/class/thermal/ 查找对应风扇设备, 更改 policy 即可


如果要在用户空间控制风扇, 需先确保PWM策略为 user space, 后:
```shell
#!/bin/bash

# 读取当前 CPU 温度（单位：摄氏度）
get_temp() {
    cat /sys/class/thermal/thermal_zone0/temp | awk '{print int($1/1000)}'
}
# 设置风扇转速 (0-255)
set_fan_speed() {
    echo "$1" | sudo tee /sys/devices/platform/pwm-fan/hwmon/hwmon*/pwm1 > /dev/null
}
# 主逻辑循环
# 具体控制逻辑见 if-else if部分, 如果看不懂可以问AI
while true; do
    temp=$(get_temp)
    echo "Temp: $temp"
    if [[ $temp -gt 50 ]]; then
        set_fan_speed 255
        sleep 300
    else
      set_fan_speed 0
      sleep 15
    fi
done
```
为了让这个温控脚本能开机自启, 这里创建一个systemd service:
```text title="/etc/systemd/system/fan-control.sevice"
[Unit]
Description=Rock5B Fan Control

[Service]
User=root
Type=simple
ExecStart=/bin/bash /root/fan-control.sh

[Install]
WantedBy=multi-user.target
```
之后, reload一下systemd:
```shell
sudo systemctl daemon-reload
sudo systemctl enable --now fan-control
```

## GPU驱动
### 系统层面
闭源libmali, 开源panfrost/panthor, 具体差别见[这里](https://docs.radxa.com/rock5/rock5b/radxa-os/mali-gpu)
panfrost和panthor区别: 前者是社区逆向出来的GPU驱动, 后者是arm官方做的, linux 6.10亮相. <br>
老硬件用前者, 新硬件的用后者. 如果要让旧内核( <6.10 )也用上panthor, 需要自行移植补丁. 已知rk vendor内核是有panthor选项的 <br>
都需要内核开启对应的支持:
  - CONFIG_DRM_PANFROST
  - CONFIG_DRM_PANTHOR
如果你的内核没编译这两项之一, 很遗憾, 你需要手动重新编译内核 <br>
> ophub的rk3588内核中, 6.1.115/118是开启了Panfrost, 没开Panthor的, 6.1.141是都没开的. 不过该大佬做的系统能方便换内核.

另外, 这些都只是内核空间需要的, 要让GPU真正起作用, 还需要用户空间程序 Mesa <br>
为了更好的支持, 一般推荐自己编译mesa([教程](https://docs.mesa3d.org/drivers/panfrost.html)), 而不是使用发行版里的 <br>

### 应用层面
尽管mesa + panfrost/panthor已经能驱动GPU了, 但只限于3D性能, 如果遇到视频编解码场景极大概率还是软件负责的
> add-apt-repository 是ubuntu特有的软件包, 故推荐使用ubuntu系统
```shell
sudo add-apt-repository ppa:liujianfeng1994/panfork-mesa
sudo add-apt-repository ppa:liujianfeng1994/rockchip-multimedia
sudo apt update
sudo apt dist-upgrade
sudo apt install rockchip-multimedia-config mali-g610-firmware
```
支持rk视频硬件加速的软件:
  - chromium-browser
  - gstreamer1.0-rockchip
  - clapper
  - ffmpeg
  - kodi
  - moonlight-embedded
  - moonlight-qt
具体介绍见[这里](https://forum.radxa.com/t/introduction-to-rockchip-multimedia-ppa-for-ubuntu-jammy/14537)
