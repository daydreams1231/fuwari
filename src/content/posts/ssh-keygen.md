---
title: SSH密钥相关
published: 2025-12-29
description: '在Openwrt和常规Linux上配置SSH密钥登录'
image: ''
tags: [Linux, SSH]
category: '随笔'
draft: false 
lang: ''
---

出于方便考虑, 只在那些仅有一个root用户的主机上需要配置仅SSH密钥登录, 因为公网的密码爆破一般只爆破root用户 <br>
如果主机有别的用户作为主账户, 就去sshd_config把允许root用户登录改为no <br>
> 对于Openwrt, 由于其webui仅支持root密码登录, 所以建议采取密钥+强密码的形式

目前推荐使用ED25519 (EDDSA) 作为密钥算法 <br>

密钥的PassPhrase是可选的 <br>

# 一般Linux发行版: 生成密钥 (公钥 + 私钥)
```shell
ssh-keygen -t ed25519 -f KEY_FILE_NAME
```
执行该命令后, 当前目录下会生成 *KEY_FILE_NAME* 和 *KEY_FILE_NAME.pub* 两个文件<br>
.pub结尾的是公钥, 给目标待登录服务器使用, 即将公钥内容追加到服务器的authorized_keys文件末尾; 另一份是私钥, 只能自己用<br>

OpenWRT默认使用Dropbear作为SSH服务器, 其密钥保存在/etc/dropbear/authorized_keys, 一行一个密钥, 文件权限0600<br>
Debian等正常Linux发行版使用OpenSSH作为SSH服务器, 其密钥保存在~/.ssh/authorized_keys, 其余同上<br>
```shell
cat KEY_FILE_NAME.pub >> ~/.ssh/authorized_keys
```
> OpenWRT下的 **ssh ssh-keygen dropbearkey** 命令都链接到/usr/sbin/dropbear


使用OpenWRT的Dropbear生成的密钥, 其私钥看起来是这样的: <br>
```text
ssh-ed25519   @?v?4%u`熢籈}?b 骃礊攰令亏耸塘?過諫;?や?藷.鳥暥镔灬
```
而正常Linux发行版生成的私钥是这样的:
```
-----BEGIN OPENSSH PRIVATE KEY-----
xxx
-----END OPENSSH PRIVATE KEY-----
```
## 两种格式的私钥互转
如无特殊要求一般不建议转换成Dropbear格式私钥, 因为其在市面主流SSH软件上都有兼容问题 (不认密钥) <br>
dropbearconvert是OpenWRT上的一个软件包, 别的系统有没有不清楚 <br>
```shell title=OPENSSH转DROPBEAR
dropbearconvert openssh dropbear OPENSSH_KEYFILE DROPBEAR_KEYFILE
```
变更openssh dropbear位置即可逆转换

> 将一个私钥经2道转换后对比, 发现只有部分文本不一致, 大体是相同的, 且都能正常与公钥配对

# 为Linux服务器启用仅密钥登录
在/etc/ssh/sshd_config中，保证以下内容不被注释:
```
PermitRootLogin prohibit-password
PubkeyAuthentication yes
PasswordAuthentication no
PermitEmptyPasswords no
```
完成后:
*sudo systemctl restart ssh*