---
title: PC版百度网盘关闭自动更新/换老版本
published: 2026-02-23
description: '在Windows下关闭百度网盘烦人的自动更新, 或者换回老版本'
image: ''
tags: [Windows, Baidu]
category: '随笔'
draft: false 
lang: 'zh_CN'
---
# 前言
在所有国内网盘中, 百度网盘可以说是我最不喜欢的一个, 不仅仗着资源优势强制对普通会员限速, 逼迫你开SVIP, 还无差别对所有用户投放广告, 即使开了会员也要看广告, 当然PC版中这个问题还不算严重. <br>
最为恶心的是更新, 每次更新总会带来一堆用不到的功能, 除了增加广告和卡顿外一无是处, 因而有了这篇文章 <br>
事先声明, 这个教程非原创, 我从southplus上学的 <br>

## 关闭自动更新
1: 前往百度网盘安装目录, 即有`BaiduNetdisk.exe`的那个文件夹 <br>

2: 删除文件夹 `AutoUpdate` <br>

3: 删除文件 `autoDiagnoseUpdate.exe` <br>

4: 删除文件 `kernelUpdate.exe` <br>

## 换老版本
前往: [Here](https://pan.baidu.com/disk/version) <br>

在该页面选择需要下载的版本, 记住其版本号 <br>

修改如下URL后缀, 如这个就是下载7.24.1.2版本的百度盘 <br>
```text
https://issuepcdn.baidupcs.com/issue/netdisk/yunguanjia/BaiduNetdisk_7.24.1.2.exe
```