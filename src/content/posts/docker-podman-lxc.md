---
title: Linux容器化及一些自用仓库
published: 2025-09-28
description: 'Linux容器化技术简介, 以及自己常用的一些镜像'
image: ''
tags: [Docker, Container, LXC, Podman, Linux]
category: '教程'
draft: false 
lang: ''
---

Linux下, 主流容器引擎有Docker Podman LXC(Linux Container) <br>
  - Docker作为容器化的鼻祖, 其作用就不阐述了, 自行搜索
  - Podman可以看作Docker的轻量版本, 无守护进程, 可rootless, 兼容大部分Docker镜像, 可无缝迁移
  - LXC也是一种容器化技术, 不过主要用来部署系统, 更适合需要完整操作系统环境的场景. 例如在Armbian上部署一个Ubuntu

三者原理其实大差不差, 都是隔离一个镜像的rootfs出来, 共享宿主机内核 <br>
硬要说区别, Docker和Podman一般都是用来部署特定应用的, 部署后开箱即用, 当然你把它当成一个系统用也没什么问题(比如使用容器环境编译软件) <br>

# Docker
## 安装
```shell wrap=false
# 官方脚本, 国内不推荐使用
wget https://get.docker.com -O get-docker.sh && sudo bash get-docker.sh

# 国内第三方脚本, 使用国内镜像源, 更快, 也会自动处理docker镜像源
bash <(curl -sSL https://linuxmirrors.cn/docker.sh)

# 或者手动安装:
sudo apt-get update
sudo apt-get install ca-certificates curl
sudo install -m 0755 -d /etc/apt/keyrings
sudo curl -fsSL https://download.docker.com/linux/debian/gpg -o /etc/apt/keyrings/docker.asc
sudo chmod a+r /etc/apt/keyrings/docker.asc

echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.asc] https://download.docker.com/linux/debian \
  $(. /etc/os-release && echo "$VERSION_CODENAME") stable" | \
  sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
sudo apt-get update

sudo apt-get install docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
```

## 卸载
```shell
sudo apt-get purge docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin docker-ce-rootless-extras
sudo rm -rf /var/lib/docker
sudo rm -rf /var/lib/containerd
sudo rm /etc/apt/sources.list.d/docker.list
sudo rm /etc/apt/keyrings/docker.asc
```

## 调试
如果Docker无法正常启动, 可使用如下命令来调试:
```shell
sudo dockerd --debug
```

## 查看容器资源使用情况
```shell
docker stats
```

## 从iptables迁移到nftables
现代Linux系统都换成了nftables以操作NetFilter内核模块, 但同时也保留了iptables命令, 不过其后端换成了nft <br>
```json title="/etc/docker/daemon.json"
{
    "iptables": false,
    "ip6tables": false
}
```
某些系统上, 由于内核原因, 不能使用基于nft后端的iptables, 此时就只能强制启用旧版本iptables了 <br>
在ubuntu上:
```shell
sudo update-alternatives --set iptables /usr/sbin/iptables-legacy
```

## 换源
这里提供的两个镜像源仅供参考, 请自己找适合的源
```json title="/etc/docker/daemon.json" wrap=false
{
    "registry-mirrors": ["https://docker.1ms.run", "https://docker.1panel.live"]
}
```

## 换容器存储位置
Docker数据目录一般在 `/var/lib/docker` 下, 而某些容器产生大量的文件IO, 对于eMMC设备十分不友好, 故需要换容器数据存储位置
```json title="/etc/docker/daemon.json" wrap=false
{
    "data-root": "/mnt/docker"
}
```

## 换容器使用的DNS
默认情况下, 容器使用的是在其被启动时系统的DNS文件. 除了在创建容器时添加 `--dns 223.5.5.5` 参数以指定该容器的DNS外, 还能修改如下配置文件: <br>
```json title="/etc/docker/daemon.json"
{
    "dns": ["223.5.5.5", "8.8.8.8"]
}
```
该配置会让所有`新启动`的容器的DNS为配置指定的内容 <br>

## 容器使用代理
大部分Linux软件都会在启动时尝试读取环境变量 `http_proxy` 和 `https_proxy` <br>
通过在创建容器时, 指定这些环境变量, 即可让容器内的程序自动使用代理 <br>

例如: **-e http_proxy=socks5h://user:password@192.168.1.1:1111 -e https_proxy=socks5h://user:password@192.168.1.1:1111**
> 该方法并不总是有效, 部分程序并不会去管这两个参数

## 容器网络
Docker常用网络有bridge host两种, 稍微不常用一点的是macvlan, 别的我不怎么使用 <br>
host网络中, 容器直接使用宿主机网络, 由于不需要NAT, 故性能最好, 但把容器网络暴露出来对某些程序来说可能有点危险 <br>
macvlan可类比成一个交换机, 容器有自己的mac, 和宿主机的网口一起接到软件交换机上, 具体实现请自行搜索 `macvlan原理` <br>

bridge为容器默认网络, 在创建容器时, docker会创建一对网口, 分别叫 vethxxxx@ifXX(宿主机中) 和 ethX@ifXXX(容器内)<br>
搜了下说带@的网口代表是点对点设备, 故容器上网流程: 容器 -> 容器内ethX@XXX -> 宿主机vethxxxx@ifXX -> 宿主机docker0 -> 宿主机ethXX <br>
其中docker0即是容器们的网关和DHCP服务器, 也承担容器间的数据转发 <br>

## 容器目录映射
Dockek的 -v 参数可以映射宿主机目录到容器内, 以冒号分隔, 左边是宿主机, 右边是容器内. <br>
但后续宿主机该目录发生变化时(例如挂载了一个硬盘到此处), 该变化不会同步到容器内 <br>
要解决这个问题, 可在容器 -v 参数的右边添加rslave, 让容器外的新挂载同步到容器内 <br>
例如:
```tip[Docker创建容器参数]
-v /mnt:/in/container:rslave
```

## 容器访问宿主机
以bridge网络容器为例, 假设docker0接口IP为172.16.0.1, 容器要访问宿主机, 除了直接填这个IP外, 还可以使用 `host.docker.internal` <br>
> 对于podman, 则是: `host.containers.internal` <br>

这个IP会被容器解析成docker0接口的IP

## 普通用户使用docker命令
```shell
sudo usermod -aG docker $USER
```

# PODMAN
## 安装
```shell
sudo apt update
sudo apt install podman

podman -v
```
不同用户的镜像存放路径是不同的, root用户的容器及镜像放在/var/lib/containers/storage/, 而普通用户的放在~/.local/share/containers/storage/ <br>

## 配置镜像加速
```text title="/etc/containers/registries.conf"
unqualified-search-registries = ["docker.io"]

[[registry]]
prefix = "docker.io"
insecure = false
blocked = false
location = "docker.io"
[[registry.mirror]]
location = "docker.1ms.run"
[[registry.mirror]]
location = "docker.1panel.live"
```

## 让非root用户的podman容器开机自启
podman没有守护进程, 必须借助systemd等工具实现容器自启动, 你去问ai会回复你加上--restart参数, 但那只对docker有效 <br>
> 当然, 你也可以在一些开机会自动执行的脚本(比如rc.local)里直接写命令: podman start xxx, 不过这样不是很优雅就是了 <br>

对于root用户, podman容器自启动文件放在: `/etc/containers/systemd/`, 其他用户放在: `~/.config/containers/systemd` <br>
别的位置请自行查阅podman手册 <br>

:::warn[Podman Quadlet]
podman在 4.4.x 之后, 引入了 Quadlet, 可以自动生成容器的systemd service文件, 让容器能随系统启动 <br>
但debian12的podman是4.3.1, 所以对于该系统, 要么你:
  - 使用podman generate systemd <容器ID> 来为单个容器生成systemd service文件, 再手动移动至指定目录, 最后手动enable, 但这样的坏处时如果后续容器有更新, 以上步骤还得重来一遍, 非常麻烦
  - 升级到新版本的podman
:::

在这些目录下随便新建一个xxx.container文件, 并写入如下内容:
```
[Unit]
Description= 我是描述

[Container]
ContainerName=容器名称
Image=镜像完整路径, 如: docker.io/xxx/abcd:latest
Volume=容器映射路径, 如: /mnt:/app/data:rslave
Network=容器网络模式, 不填写时默认桥接
PublishPort=桥接容器映射端口
Environment=容器环境变量, 如aaa=123
# 设置容器自动更新
AutoUpdate=registry

[Service]
# 如果容器异常退出, 大部分软件只靠重启大概也解决不了问题, 更常遇到的是配置问题引起软件退出
Restart=no

[Install]
WantedBy=multi-user.target default.target
```
执行: `systemctl --user daemon-reload` 来重载systemd配置 <br>

> 若后续不需要自启了, 就把Install部分注释掉, 再reload

:::info[与Systemd文件的不同]
可以看出, 该文件与常规systemd文件的不同在于: 多了一个[Container], 其余部分相同, 也就意味着其余部分能加入如After=xxx.target Restart=xxx等参数 <br>
但可惜的是, 使用`podman kill/stop`停止容器时, 容器的返回值非0, 触发 **systemd restrat=on-failure** 策略, 因而无法关闭容器, 必须使用`systemdctl --user stop xxx`, 除了这个解决方案, 也可以直接让 `Restrat=no` <br>

具体参数查看: https://docs.podman.io/en/latest/markdown/podman-systemd.unit.5.html
:::
在配置完以上部分后, 额外执行如下命令, 以启用自动更新
```shell
# 两个文件其实就是实现定时触发podman auto-update命令, 用cron也差不多
sudo systemctl enable podman-auto-update.{service,timer}
```

## 让非root用户的容器能持续运行
对于非root用户, systemd默认只在用户登陆时启动其用户定义的systemd服务, 在用户退出后其systemd服务也相应停止 <br>
要改变该行为, 使用:
```shell
sudo loginctl enable-linger <USERNAME>
```
让用户的systemd服务能和系统内的systemd服务一样在机器启动时启动, 机器关机时关闭

> 用户定义的systemd服务必须通过: systemctl --user daemon-reload 来reload service, 对于root用户请不要带 --user 参数

:::warn
对于systemd拉起的容器, 容器停止时, 容器也会被删除, 容器内数据也是这样, 所以如果你有一些重要数据保存在其内, 请注意持久化映射
:::

# LXC (todo)
lxc-create -t 容器模板 -n 容器名 <br>

download是一个特殊的容器模板, 从该模板创建容器时, lxc会从远程服务器下载所有可选容器列表, 后进入交互模式, 此时你再输入相关信息, 即可完成容器的下载与创建 <br>
如果不想要交互模式, 可在命令后加上: -- --dist 发行版名称 --release 版本 --arch 架构 <br>
单独的 -- 的作用是把后面的参数传给模板, 即把相关信息传给download模板 <br>

容器保存在: /var/lib/lxc/容器名 下面 <br>
其下有一config文件, 为容器的配置文件 <br>
若要设置直通物理网卡, 在其后加入: <br>
```
lxc.net.0.type = phys
lxc.net.0.link = eth1
lxc.net.0.flags = up
```

# 个人常用镜像
## OpenList
```shell wrap=false
docker run -d --name openlist --user 1000:1000 --restart unless-stopped -v /root/config/openlist:/opt/openlist/data -p 5244:5244 -e UMASK=022 openlistteam/openlist:latest
```
```text wrap=false title="Podman Quadlet.container"
[Unit]
Description=OpenList
Requires=network-online.target
After=network-online.target systemd-timesyncd.service

[Container]
ContainerName=openlist
Image=docker.io/openlistteam/openlist:latest
Volume=/root/config/openlist:/opt/openlist/data
PublishPort=5244:5244
Environment=UMASK=022
Environment=TZ=Asia/Shanghai
User=1000
Group=1000
AutoUpdate=registry

[Service]
Restart=no

[Install]
WantedBy=multi-user.target default.target
```

## CloudDrive2
一个类似openlist/alist的网盘挂载工具, 闭源, 需要登录账户使用, 免费版只能添加2个云盘 <br>
webui: IP:`19798` <br>
```shell wrap=false title="Docker CLI"
docker run -d \
    --name clouddrive \
    --restart unless-stopped \
    --env CLOUDDRIVE_HOME=/Config \
    -v /root/config/cd2:/Config \
    -p 19798:19798 \
    --pid host \
    cloudnas/clouddrive2:latest
```
```text wrap=false title="Podman Quadlet.container"
[Unit]
Description=Clouddrive2
Requires=network-online.target
After=network-online.target systemd-timesyncd.service

[Container]
ContainerName=clouddrive2
Image=docker.io/cloudnas/clouddrive2:latest
Volume=/root/config/cd2:/Config
PublishPort=19798:19798
Environment=CLOUDDRIVE_HOME=/Config
AutoUpdate=registry

[Service]
Restart=no

[Install]
WantedBy=multi-user.target default.target
```

## qBittorrent Enhanced Edition
若下载目录不属于PUID PGID的用户, 请开启ENABLE_DOWNLOADS_PERM_FIX, 或者确保该目录能被用户读写
```shell wrap=false title="docker-cli"
docker create  \
    --name=qbee  \
    -e WEBUIPORT=8080  \
    -e PUID=1000 \
    -e PGID=1000 \
    -e TZ=Asia/Shanghai \
    -e ENABLE_DOWNLOADS_PERM_FIX=false \
    --network host \
    -v /root/config/qbee:/config  \
    -v /mnt:/downloads:rslave  \
    --restart unless-stopped  \
    superng6/qbittorrentee:latest
```
```text wrap=false title="Podman Quadlet.container"
[Unit]
Description=QBEE
Requires=network-online.target
After=network-online.target systemd-timesyncd.service

[Container]
ContainerName=qbee
Image=docker.io/superng6/qbittorrentee:latest
Volume=/root/config/qbee:/config
Environment=PUID=1000
Environment=PGID=1000
Environment=TZ=Asia/Shanghai
Environment=ENABLE_DOWNLOADS_PERM_FIX=false
Environment=WEBUIPORT=5200
Volume=/mnt:/downloads:rslave
Volume=/root/config/qbee:/config
Network=host
AutoUpdate=registry

[Service]
Restart=no

[Install]
WantedBy=multi-user.target default.target
```

## Aria2
由于镜像没对下载的文件做校验, 在国内经常下载到不完整的文件, 导致执行报错, 故推荐在创建容器前手动下载文件: <br>
```shell wrap=false
wget https://p3terx.github.io/aria2.conf/aria2.conf -O /root/config/aria2/aria2.conf
wget https://p3terx.github.io/aria2.conf/script.conf -O /root/config/aria2/script.conf
wget https://p3terx.github.io/aria2.conf/core -O /root/config/aria2/script/core
wget https://p3terx.github.io/aria2.conf/clean.sh -O /root/config/aria2/script/clean.sh
wget https://p3terx.github.io/aria2.conf/delete.sh -O /root/config/aria2/script/delete.sh
```

```shell wrap=false title="Docker CLI"
sudo docker run -d \
    --name aria2-pro \
    --restart unless-stopped \
    --log-opt max-size=1m \
    -p 6800:6800 \
    -e PUID=1000 \
    -e PGID=1000 \
    -e RPC_SECRET=<PASSWORD> \
    -e RPC_PORT=6800 \
    -e LISTEN_PORT=23456 \
    -e IPV6_MODE=true \
    -e UPDATE_TRACKERS=false \
    -v /root/config/aria2:/config \
    -v /mnt/disk:/downloads:rslave \
    p3terx/aria2-pro:latest
```
```text wrap=false title="Podman Quadlet.container"
[Unit]
Description=Aria2-Pro container
Requires=network-online.target
After=network-online.target systemd-timesyncd.service

[Container]
ContainerName=aria2-pro
Image=docker.io/p3terx/aria2-pro:latest
Volume=/root/config/aria2:/config
Volume=/mnt:/downloads:rslave
Network=host
Environment=PUID=1000
Environment=PGID=1000
Environment=UPDATE_TRACKERS=false
Environment=RPC_SECRET=<PASSWORD>
Environment=RPC_PORT=6800
Environment=LISTEN_PORT=23456
Environment=IPV6_MODE=true
LogOpt=max-size=1m
AutoUpdate=registry

[Service]
Restart=no

[Install]
WantedBy=multi-user.target default.target
```

## PeerBanHelper
```shell wrap=false title="Docker CLI"
docker run -d \
    --name PeerBanHelper \
    --restart unless-stopped \
    --stop-timeout 30 \
    -p 9898:9898 \
    -v /root/config/pbh:/app/data \
    -e PUID=1000 \
    -e PGID=1000 \
    -e TZ=UTC \
    ghostchu/peerbanhelper:latest
```
```text wrap=false title="Podman Quadlet.container"
[Unit]
Description=PeerBanHelper Container
Requires=network-online.target
After=network-online.target systemd-timesyncd.service

[Container]
ContainerName=peerbanhelper
Image=docker.io/ghostchu/peerbanhelper:latest
Volume=/root/config/pbh:/app/data
PublishPort=9898:9898
Environment=PUID=1000
Environment=PGID=1000
Environment=TZ=UTC
AutoUpdate=registry

[Service]
Restart=no

[Install]
WantedBy=multi-user.target default.target
```

## Watchtower (容器自动更新)
86400秒 = 1天
```shell
docker run -d --name watchtower --restart unless-stopped --volume /var/run/docker.sock:/var/run/docker.sock containrrr/watchtower --cleanup --interval 86400
```

## LibreSpeed内网测速
其实有更方便的homebox
```shell
docker run -d --name speedtest -p 5200:8080 -e MODE=standalone -v /root/config/librespeed:/database ghcr.io/librespeed/speedtest
```

## Adguard Home
conf目录下只有一个yaml配置文件, work目录下存放查询日志, 过滤器等东西(work目录最好映射到外置存储上)
```shell
docker run -d \
    --name AdguardHome \
    --restart unless-stopped \
    --network host \
    -v /mnt/disk/adg_work:/opt/adguardhome/work \
    -v /root/config/adg/conf:/opt/adguardhome/conf \
    adguard/adguardhome:latest
```

## Pi-Hole
比ADG还轻量的DNS服务器, 同样支持DNS缓存, DNS覆写等功能. 其后端貌似是dnsmasq, pi-hole本身相当于一个配置生成器+webui<br>
以Docker运行容器的内存占用为例, ADG初始占用40M+, 而Pi-hole初始占用9M, 长期运行后, pihole内存也会逐渐增长, 具体数据与配置有关<br>
长期运行后, 会积累大量的日志文件, 不过占用量还行, 200M左右, 合理范围 <br>

在运行一段时间后, 发现它的分组功能是给域名屏蔽列表用的, 让不同分组内的客户端能使用不同的DNS屏蔽列表, *不能* 做到对不同组的客户端使用不同的DNS服务器. <br>
DNS查询列表只能显示客户端的查询域名, 至于返回了什么IP是看不到的 <br>
另外, 其DNS覆写只支持最简单的让域名指向一个IP, 若设置域名指向CNAME(常见用途: Cloudflare优选域名加速), 会强制要求目标CNAME必须手动配置DNS解析IP, 但与其这样做为什么不直接改hosts呢?

软件的web界面还算美观, 但没中文支持还是差点意思, 另外里面有个查看网口详细信息的页面挺不错的, 能记录获得DHCP的时间以及收发包的大小、数量以及网口速率. <br>
管理页面为 IP:8080/admin <br>
see doc: [Pi Hole](https://github.com/pi-hole/docker-pi-hole)
```shell wrap=false
docker run --name pihole  -p 8080:80/tcp -p 4433:443/tcp -e TZ=Asia/Shanghai -e FTLCONF_webserver_api_password="123456" -e FTLCONF_dns_listeningMode=all -v ./pihole:/etc/pihole -v ./etc-dnsmasq.d:/etc/dnsmasq.d --cap-add NET_ADMIN --restart unless-stopped pihole/pihole:latest
```

## natfrp
```shell
docker run -d \
   --name natfrp \
   --restart unless-stopped \
   -v /root/config/natfrp:/run \
  -p 7102:7102 \
   natfrp.com/launcher:latest
```

## FRP wrap=false
```shell
# Client
docker run -d --restart=unless-stopped --network host -v /root/config/frp/frpc.toml:/etc/frp/frpc.toml --name frpc snowdreamtech/frpc
# Server
docker run -d --restart=unless-stopped --network host -v /root/config/frp/frps.toml:/etc/frp/frps.toml --name frps snowdreamtech/frps
```

## 百度网盘:(5800网页，5900VNC)
```shell
docker run -d \
    --name=baidunetdisk \
    -p 5800:5800 \
    -p 5900:5900 \
    -v /root/config/baidu:/config \
    -v /disk:/config/baidunetdiskdownload \
    --restart on-failure \
    johngong/baidunetdisk:latest
```

## 迅雷
```shell
docker run -d \
  --name xunlei \
   -v /root/config/xunlei:/xunlei/data \
   -v /disk:/xunlei/downloads \
   -p 2345:2345 \
   --privileged \
   cnk3x/xunlei
```

## Firefox浏览器 (5901 VNC)
注意, 这里只允许VNC访问, VNC密码仅仅用于鉴权, 但无法防止MITM攻击, 请只在可信内网/VPN环境使用 <br>
```shell wrap=false
docker run -d --name firefox -p 5901:5901 -e TZ=Asia/Shanghai -e DISPLAY_WIDTH=1280 -e DISPLAY_HEIGHT=720 -e ENABLE_CJK_FONT=1 -e WEB_LISTENING_PORT=5900 -e VNC_LISTENING_PORT=5901 -e VNC_PASSWORD=<VNC_PASSWORD> -v /root/config/firefox:/config:rw jlesage/firefox
```
如果宿主机的内核有GPU驱动, 可加上: `--device /dev/dri` 来启用GPU加速