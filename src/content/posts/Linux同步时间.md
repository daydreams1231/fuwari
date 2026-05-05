---
title: Linux同步时间
published: 2026-04-08
description: 'Linux同步时间'
image: ''
tags: [Linux, Time]
category: '随笔'
draft: false 
lang: ''
---

timedatectl 是 systemd包 的一个命令
```shell
sudo apt install systemd-timesyncd

# 如果上面的包因为版本依赖原因无法安装, 可先用下面的命令替代
sudo apt install ntp && sudo service ntp restart

timedatectl set-timezone 'Asia/Shanghai'

# 编辑配置文件 /etc/systemd/timesyncd.conf
NTP=cn.ntp.org.cn ntp1.aliyun.com ntp.tencent.com

# 网络同步时间
timedatectl set-ntp true
```

# 强制设置时间
date -s "2025-08-08 11:53:06"