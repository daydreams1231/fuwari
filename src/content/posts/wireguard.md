---
title: Wireguard配置教程
published: 2025-09-19
description: 'Wireguard配置 + 异地组网/全局上网教程'
image: ''
tags: [Linux, VPN, Proxy]
category: '教程'
draft: false 
lang: ''
---
:::WARN
Wireguard不是为了翻墙设计的, 只适合在国内互联使用, 其流量特征较明显, 易导致IP被墙!

另外WG是基于udp进行传输的, 可能会遇到运营商的QoS限速问题
:::
# 安装
```shell
sudo apt update
sudo apt install wireguard resolvconf
```

# 配置
:::NOTE
以下操作均以`root`用户执行
:::
启用IP转发
```shell
echo "net.ipv4.ip_forward = 1" >> /etc/sysctl.conf
echo "net.ipv6.conf.all.forwarding = 1" >> /etc/sysctl.conf
sysctl -p
```
查看效果, 都输出`1`代表成功启用
```shell
# IPv4
cat /proc/sys/net/ipv4/ip_forward

# IPv6
cat /proc/sys/net/ipv6/conf/all/forwarding
```
创建配置文件夹
```shell
mkdir -p /etc/wireguard && cd /etc/wireguard
```
生成公钥和私钥 (一共有两套证书, client server各一份, 每份都有各自的公钥和私钥), 如果后面需要添加新客户端, 直接运行第二条命令, 再向配置文件添加一个Peer部分就行
```shell
wg genkey | tee server_privatekey | wg pubkey > server_publickey
wg genkey | tee client_privatekey | wg pubkey > client_publickey
```
创建配置文件 (假设上网口是ens5, 服务器监听端口为udp/5888)
> 该端口是Wireguard Server和Client间的数据通信端口, 并不是映射端口
```shell wrap=false title="服务器配置"
cat >> /etc/wireguard/wg0.conf << EOF
[Interface]
PrivateKey = $(cat server_privatekey)
Address = 172.20.1.1/24, fddd:1234::1/64
PostUp   = iptables -A FORWARD -i wg0 -j ACCEPT; iptables -A FORWARD -o wg0 -j ACCEPT; iptables -t nat -A POSTROUTING -o ens5 -j MASQUERADE
PostUp   = ip6tables -A FORWARD -i wg0 -j ACCEPT; ip6tables -A FORWARD -o wg0 -j ACCEPT; ip6tables -t nat -A POSTROUTING -o ens5 -j MASQUERADE
PostDown = iptables -D FORWARD -i wg0 -j ACCEPT; iptables -D FORWARD -o wg0 -j ACCEPT; iptables -t nat -D POSTROUTING -o ens5 -j MASQUERADE
PostDown = ip6tables -D FORWARD -i wg0 -j ACCEPT; ip6tables -D FORWARD -o wg0 -j ACCEPT; ip6tables -t nat -D POSTROUTING -o ens5 -j MASQUERADE
ListenPort = 5888
DNS = 223.5.5.5
MTU = 1420

# 客户端1
[Peer]
PublicKey =  $(cat client_publickey)
AllowedIPs = 172.20.1.2/32, fddd:1234::2/128

# 客户端2
[Peer]
PublicKey =  $(cat client2_publickey)
AllowedIPs = 172.20.1.3/32, fddd:1234::3/128

# 客户端.....
EOF
```
配置自启动
```shell
systemctl enable wg-quick@wg0
wg-quick up wg0
```
```text wrap=false title="客户端1配置"
[Interface]
PrivateKey = $(cat client_privatekey)
Address = 172.20.1.2/32, fddd:1234::2/128
DNS = 223.5.5.5
MTU = 1420

[Peer]
PublicKey = $(cat server_publickey)
AllowedIPs = 172.20.1.0/24, fddd:1234::/64
Endpoint = <服务器IP:端口>
PersistentKeepalive = 25
```
:::NOTE
Wireguard中其实不存在Server和Client的概率, 大家都是Peer, 大伙地位对等. 主动发起连接的一方被视为客户端, 客户端的配置文件中Peer字段要定义`Endpoint`来定义服务器, 被动接受连接的一方即服务端, 其配置文件的Interface字段要定义`ListenPort` <br>

以一个Peer的视角来看, `[Interface]` 定义了本地接口的相关参数, 里面填入自己这个Peer的私钥. `[Peer]`下填入对方的公钥, 其下的`AllowedIPs`定义将添加到本机的路由(示例: 10.1.1.1/32, 10.1.1.0/24, 0.0.0.0/0, ::/0, 1.1.1.1/32等) (10.1.1.1/24和10.1.1.0/24是等效的), 如有多个网段使用逗号连接。 <br>

如果填`0.0.0.0/0`代表添加默认路由, 添加后本机所有流量都发送到这个Peer那里去
如果填`172.20.1.100/32`代表添加一条专到172.20.1.100这个IP的路由, 添加后本机发往172.20.1.100的流量都会发送到这个Peer那里去

单个配置文件内若定义了多个Peer, 则这些Peer的AllowedIPs不能相同, 但是可以包含 <br>
如PeerA: 10.1.1.1/24, PeerB: 10.1.1.2/32, 现在有一数据包发往10.1.1.2/32, 结果是走PeerB
:::

以上这份配置只能保证异地组网的效果, 客户端与客户端、客户端与服务器之间是能连通的, 如果不能, 排查端口是否被阻塞了 <br>
如果把`AllowedIPs`设置成`0.0.0.0/0 ::/0`
  - 在手机版的Wireguard中, 在`路由的IP地址(段)`这, 填入`0.0.0.0/0 ::/0`后, 再选中`排除局域网`, 此时就能全局上网, 也能访问局域网, 但如果不选中, 就上不了网, 局域网也无法访问
  - 在Windows上, Wireguard提供了一个叫`拦截未经隧道的流量(kill-switch)`选项, 默认勾选时, 会导致无法访问局域网, 取消勾选后可解决问题, 此时即全局上网, 如果你在用代理软件, 此时你产生的流量先经过代理软件, 无论是否分流, 都被发往VPS, 如果流量是国内流量, VPS帮你访问, 如果是国外流量, 数据包的dst_ip是机场IP, VPS会帮你发到对应的机场.
