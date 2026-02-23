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

# 概念及Q&A
## 启动流程
[瑞芯微启动流程](https://opensource.rock-chips.com/wiki_Boot_option#Boot_flow). <br>
芯片上电 -> 运行芯片内部MaskRom(Boot rom) ->  

## 使用RK Dev Tool时使用的Loader是什么东西?

正常情况下, 芯片上电后, 会先运行芯片内部的代码Boot Rom, 或者说Mask Rom, 这个代码用于检测外部存储设备如EMMC SD卡等, 如果没存储设备, 或者在所有的存储设备中都没有Loader, 板子会idle, 并不会自己进maskrom模式, 此时串口也没任何输出 <br>
如果有Loader能加载, 比如一个spi nor flash里面刷了idbloader和uboot, 就会进入spi flash继续执行目标代码. <br>

如果是emmc或者sd卡, 里面有loader, 此时按recovery按键上电是不会进loader模式的

loader类似uboot, 也是一种boot loader, 而我们使用的loader.img其实就是编译uboot时产出的idbloader.img <br>
该loader在运行完后会继续执行u-boot.itb, 也就是uboot <br>

> 和全志对比一下, idbloader.img和u-boot.itb一起对应全志的u-boot-with-spl.bin

> 在Mask Rom模式下, 只能进行对于loader的烧写


如果你的板子没有SPI Flash, 比如Radxa Rock 5C, 由于没有任何存储介质, 上电后应该是直接进mask rom模式

loader有时也指miniloader, todo

## 瑞芯微多介质启动优先级


# Uboot

编译出的idbloader.img和u-boot.itb即我们需要的