---
title: Samba / ksmbd /FTP 配置
published: 2025-10-09
description: ''
image: ''
tags: [Linux, Samba, KSMBD]
category: '教程'
draft: false 
lang: ''
---
# Kernel Samba Server(ksmbd)
自5.15后并入主线, 截至今日, 相较于另一个Samba性能更优, 但功能更少 <br>
## 安装
默认情况下应该已编译到内核中, 只需要安装一个控制其的工具即可
```shell
sudo apt install ksmbd-tools
```
## 配置共享用户
ksmbd 的用户不需要在linux系统中存在，这与SMB不同. 但实际操作中不建议这样做, 尽量添加一个主机中存在的非root用户
```shell
ksmbd.adduser -a USERNAME -p PASSWORD
```
## 配置文件共享
这个根据自己需求来设置, 示例:
```
[global]
	#是否绑定到网络接口,默认为No.如果是,samba将只在该网口上提供服务,只有该网口所属子网内的用户能够使用samba
	#bind interfaces only = no
	#在上面的bind interfaces only=yes时,下面的interfaces才会生效
	#interfaces= IP / Device

	#workgroup = WORKGROUP

	netbios name = KSMBD SERVER
	#设置无效用户
	# 需要先ksmbd.adduser -d root来检查是否有root账户，如果没有则不需要设置
	#invalid users = root

	# deadtime用来设置断掉一个没有打开任何文件的连接的时间。单位是分钟,默认0代表Samba Server不自动切断任何连接。(如果有打开的文件则samba永远不会自动关掉连接)
	deadtime = 5

	# 用户空间回复心跳帧的秒数. 如果超时,所有会话和 TCP 连接都将关闭. 当 ipc timeout = 0 时，用户空间可以随时回复。
	ipc timeout = 15

	# 将未知用户映射为访客
	map to guest = Bad User

	#是否使用samba3的加密功能
	smb3 encryption=auto

	# max connections用来指定连接Samba Server的最大连接数目。如果超出连接数目,则新的连接请求将被拒绝。0表示不限制。
	max connections = 0

# 示范共享名disk
[disk]
    comment = COMMENT
    # 配置共享是否能被浏览. 默认yes. (例如你从Windows的"网络"界面扫描到了一个主机, 点进去输密码后就能看到该主机上配置的分享, 如果配置为no, 别人就看不到主机上的分享. 如果主机上没有配置wsdd2服务, 这项可忽略, 反正别人也扫不到你)
    #browseable = no
    path = /root/test
	writable = yes
	# 权限相关
 	# 设置新文件的权限
	create mask = 0770
 	# 设置新文件夹的权限
    directory mask = 0770
    # 不建议启用force user/group项, 不太安全
```
## KSMBD权限问题
所有rwx操作均是以SMB登录用户的权限进行的, 如果登录的用户不存在于系统中, 则是用其他用户的权限(___---rwx), 匿名用户默认是其他用户<br>

```
假设一个文件夹所有者为root:root, 你为abc, 那你对应的权限是其他权限
假设一个文件夹所有者为abc:abc, 你为abc, 那对应的权限是所有者权限
```
:::WARN
如果是无法打开/查看共享, 请向共享目录添加对应用户的r x权限 <br>
如果是无法写入文件, 先确保配置文件允许写入后, 向共享目录添加对应用户的w权限 <br>
:::
> 一般推荐将文件夹的所有者改为登录用户(非root), 再chmod -R 770 共享

> ls命令需要r权限, cd命令需要目录的x权限

如果不想考虑什么权限问题, 你可以在配置文件里设置force user和force group
### 查看KSMBD版本
对于 *Debian* 系统: 
```shell
# ksmbd-tools版本:
dpkg -l | grep ksmbd
# ksmbd: (查找version行, 如果没有就可能是debian sid版本, 此时参考ksmbd-tools版本也ok)
sudo modinfo ksmbd
# 实测6.12 backports内核也存在cross_mnt问题, 详见尾部的Warning部分
```
对于 *OpenWrt* 系统: 前往软件包搜索ksmbd <br>
Linux: ksmbd.control -c <br>

> ksmbd-tools和ksmbd版本不一定是相同的, 如debian12中分别为3.4.7 和 3.4.2
> 各Linux发行版的版本汇总: https://pkgs.org/download/ksmbd-tools

### 一些命令
```shell
# 停止ksmbd
ksmbd.control -s
# 启动ksmbd
ksmbd.mountd
# 重载配置文件
ksmbd.control -r
```
### Warning:
Debian12 默认的ksmbd 3.4.2 (Linux 6.1.0-35)版本存在Bug: <br>
在/mnt/disk下, 有一inner文件, 挂载硬盘到此后, 创建一文件outside <br>
如果创建一个分享shared, 指定path为/mnt/disk, 此时在smb的shared共享中只能看到inner文件, 且尝试rwx时会报无权限 / 不存在之类的error  <br>
升级到3.5.2后share部分有个crossmnt选项，默认yes <br>
如果改为no,就会导致上述问题出现，且此时创建的文件在smb中无法看到，反倒在Linux中可以看到(Linux的是挂载的文件夹) <br>
即SMB中看到的是非挂载时的文件夹，所有的操作却在挂载的文件夹内产生 <br>
Debian13及之后版本无此问题<br>

某些系统上由于内核没有配置NLS_UTF-8, 导致挂载别的主机上的SMB共享时, 如果有非英文, 就会乱码, 而KSMBD没有这种编码适应选项<br>

### 在Windows上挂载SMB硬盘后, 能打开硬盘, 但打不开其下的文件夹, 提示xx不可用, 如果该位置位于....
ksmbd.control -s
手动umount + mount 一次硬盘
ksmbd.mountd

# Samba
smbd: samba daemon <br>
nmbd: NetBIOS daemon <br>
## 安装
```shell
sudo apt update
sudo apt install samba samba-common-bin
```
## 配置
/etc/samba/smb.conf: samba主配置文件, 和上面的类似, 但某些选项有差异, 具体自己看man手册 <br>
```
# global
unix charset = UTF-8
dos charset = GBK
display charset = UTF-8
```
注意分享目录应配置成: (如果你不想遇到千奇百怪的权限问题就照做) 
```shell
sudo chown nobody:nogroup /PATH/TO/SHARE
sudo chmod 0777 /PATH/TO/SHARE
```

## 用户配置
```shell
sudo su
adduser 新用户名
smbpasswd -a 新用户名
# 如果不进行创建用户这一步, 尽管能用Linux主机的账户登录但没有任何文件夹访问权限
systemctl restart smbd
```
## (可选) 配置网络发现服务: Web Service Discovery
注意这会降低安全性, 酌情取用
```shell
sudo apt install wsdd2
sudo systemctl start wsdd2 && sudo systemctl enable wsdd2
```

## 一些Tips
如果共享名称包含`homes`字段, 则默认是访问登录用户的home目录 (即path=/home/登录用户) <br>
当然也可以强制指定path = Anywhere, 但这样和普通共享一样了. 如果配合宏 `%s` (表示登录用户名), 如/mnt/disk/%s, 可以做到让每个用户访问不同的目录 <br>


# FTP
```shell
sudo apt install vsftpd
# 找到write_enable, 使其为YES, 不过一般还是不要动为好
sudo nano /etc/vsftpd.conf
# restart ftp server
sudo systemctl restart vsftpd

# 设置完后就能直接使用Linux上存在的用户及其密码进行登录了
```