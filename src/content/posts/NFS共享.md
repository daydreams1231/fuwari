---
title: NFS共享
published: 2025-09-16
description: 'Linux配置NFS教程'
image: ''
tags: [Linux, NFS, 文件共享]
category: '教程'
draft: false 
lang: ''
---
:::tip
我自己一般不用NFS, 毕竟主力是Windows, 内网连NAS用的SMB, Openlist处理局域网文件用FTP(smb老是read error, response error: The network name was deleted.只能单线程跑任务) <br>
NFS没用户鉴权机制, 即使在内网我也不太愿意用
:::

## 安装:
对于Debian, `nfs-kernel-server`是NFS服务器必需的, `nfs-common`是用来挂载NFS用的
```
sudo apt update
sudo apt install nfs-kernel-server nfs-common
```

## 配置:
如下例子配置/mnt/nfs_share为共享目录, 允许192.168.1.0/24子网内的客户端访问, 其中`192.168.1.0/24`可以换成单个IP或者通配符`*`来允许所有IP访问, 但不推荐 <br>
当然, 其也能用域名来代替IP <br>
如下出现的参数请自行查阅NFS文档. <br>
```
<!-- /etc/exports -->
/mnt/nfs_share	192.168.1.0/24(rw,no_subtree_check,crossmnt,fsid=0)
```
值得注意的是, 目标目录需要进行如下配置:
```
sudo chown nobody:nogroup /mnt/nfs_share
sudo chmod 777 /mnt/nfs_share
```
NFS没有像SMB那样的用户权限机制, 请尽量用于可信环境如内网 <br>
如一定要在外网用, 请配置好IP范围 <br>

配置完成后:
```
exportfs -ra
exportfs -v # 查看启用的共享
systemctl enable nfs-server
systemctl restart nfs-server
```
## 在客户机上挂载NFS
CLI:
```
sudo mount.nfs4 192.168.1.X:/home /users
```
或者:
```
<!-- /etc/fstab -->
192.168.1.X:/nfs    /LOCAL_DIR nfs4    soft,intr,rsize=8192,wsize=8192
```
如果是OpenWRT, 需要先安装nfs-utils 和 kmod-fs-nfs包

如果挂载提示 No Such Device, 请检查:
lsmod | grep nfs
看是否有nfsd这个模块
