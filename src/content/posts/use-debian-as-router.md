---
title: 将Debian配置成旁路由/透明网关
published: 2026-03-12
description: '将Debian配置成旁路由/主路由'
image: ''
tags: [Linux, Debian, Clash, Proxy, iptables, Tproxy]
category: '随笔'
draft: false 
lang: ''
---
# 设备
一台运行着Debian的主机, 且系统内核开启了TProxy, ipv6 nat相关的编译配置 <br>
网口数量不限 <br>

## 准备
[Ref](https://evine.win/p/%E6%88%91%E7%9A%84%E5%AE%B6%E5%BA%AD%E7%BD%91%E7%BB%9C%E8%AE%BE%E8%AE%A1%E6%80%9D%E8%B7%AF%E5%BC%80%E5%90%AFdebian%E7%9A%84%E6%97%81%E8%B7%AF%E7%94%B1%E4%B9%8B%E8%B7%AF%E5%9B%9B/#mynftablesnft) <br>
[Ref 2](https://v2.hysteria.network/zh/docs/advanced/TPROXY/#__tabbed_2_1) <br>

对于TCP连接, 可使用Tun, TProxy, Redirect三种方式, 其中最先进, 性能最高的是TProxy <br>
对于UDP连接, 可使用Tun, TProxy. <br>

下文统一使用TProxy处理TCP + UDP, 不劫持任何udp/53上的DNS, 但其他DNS请求会进一遍内核, 由配置文件决定处理操作 <br>
国内流量可选绕过内核 <br>
如果你没有IPv6, 或者你是用在旁路由上(即你无法指定局域网的IPv6网关), 可在如下配置中删去IPv6相关的部分 <br>
如果需要代理本机产生的流量, 下文统一使用 gid 的方式来绕过clash本身产生的流量, 防止重复代理.
如果你不需要代理本机产生的流量, 下面的gid配置相关的部分可以不做 <br>

## 添加用户
```shell
echo "clash:x:0:250:::/usr/sbin/nologin" >> /etc/passwd
```

## 开启IP转发
在OpenWrt等系统上这步可省略
```shell
# 检查IPv4转发
cat /proc/sys/net/ipv4/ip_forward
echo 1 > /proc/sys/net/ipv4/ip_forward

# 检查IPv6转发
cat /proc/sys/net/ipv6/conf/all/forwarding
echo 1 > /proc/sys/net/ipv6/conf/all/forwarding

## 永久开启IP转发
## 编辑 /etc/sysctl.conf
# net.ipv4.ip_forward=1
# net.ipv6.conf.all.forwarding=1
sysctl -p
## debian13及以后的系统, 原来的sysctl已迁移到 systemd-sysctl
# 只需要把原来的conf文件移动到 /etc/sysctl.d 下面即可, 并重命名, 方便分辨功能
# 完成后执行 sysctl --system
```

## 确定数据包过滤系统的后端
```shell
iptables -V
# 如果输出: iptables x.x.x (nf_tables), 代表系统使用nftables, 否则使用iptables
```
---
以下所有命令都需要在docker修改系统路由前执行, 以防止某些Bug
以后再看能否fix this
---
## iptables实现 (适用于老系统)
```shell
# 让设备具有简单的路由NAT功能 (eth0为设备上网的接口, 自行改为对应接口, 不会的看本文末尾QA)
iptables -t nat -A POSTROUTING -o eth0 -j MASQUERADE
iptables -A FORWARD -j ACCEPT
iptables -A FORWARD -m state --state RELATED,ESTABLISHED -j ACCEPT
ip6tables -t nat -A POSTROUTING -o eth0 -j MASQUERADE
ip6tables -A FORWARD -j ACCEPT
ip6tables -A FORWARD -m state --state RELATED,ESTABLISHED -j ACCEPT

## 定义内网IP, 这一步需要安装软件包ipset, 并且内核在编译时要配置CONFIG_IP_SET, 不然下面的命令会报错
ipset create localnetwork4 hash:net
ipset create localnetwork6 hash:net family inet6

ipset add localnetwork4 0.0.0.0/8
ipset add localnetwork4 127.0.0.0/8
ipset add localnetwork4 10.0.0.0/8
ipset add localnetwork4 169.254.0.0/16
ipset add localnetwork4 192.168.0.0/16
ipset add localnetwork4 224.0.0.0/4
ipset add localnetwork4 240.0.0.0/4
ipset add localnetwork4 172.16.0.0/12
ipset add localnetwork4 100.64.0.0/10

ipset add localnetwork6 ::/128
ipset add localnetwork6 ::1/128
ipset add localnetwork6 ::ffff:0:0/96
ipset add localnetwork6 ::ffff:0:0:0/96
ipset add localnetwork6 64:ff9b::/96
ipset add localnetwork6 100::/64
ipset add localnetwork6 2001::/32
ipset add localnetwork6 2001:20::/28
ipset add localnetwork6 2001:db8::/32
ipset add localnetwork6 2002::/16
ipset add localnetwork6 fe80::/10
ipset add localnetwork6 ff00::/8

## 创建clash链
iptables -t mangle -N clash
ip6tables -t mangle -N clash

# 对所有进入设备的流量, 将数据包转到clash链处理
iptables -t mangle -A PREROUTING -j clash
ip6tables -t mangle -A PREROUTING -j clash

# 对进入clash链的数据包, DNS请求以及内网IP直接RETURN, 不打标
# 如果需要放行doh dot等流量, 自行修改
iptables -t mangle -A clash -p udp --dport 53 -j RETURN
iptables -t mangle -A clash -m set --match-set localnetwork4 dst -j RETURN
ip6tables -t mangle -A clash -p udp --dport 53 -j RETURN
ip6tables -t mangle -A clash -m set --match-set localnetwork6 dst -j RETURN

# 国内流量也不打标, 即国内流量绕过内核功能, 可选
iptables -t mangle -A clash -m set --match-set cn_ipv4 dst -j RETURN
ip6tables -t mangle -A clash -m set --match-set cn_ipv6 dst -j RETURN

# 对流量打标, 再将其转发到127.0.0.1:7894, --on-ip 默认为流量入站IP, 由于clash tproxy入站只能监听127.0.0.1, 所以这里也对应填127.0.0.1
# clash配置中的tproxy-port: 7894 只能监听127.0.0.1, 如果需要ipv6, 请额外配置listeners监听tproxy ::1. 具体见 https://wiki.metacubex.one/config/inbound/
iptables -t mangle -A clash -p tcp -j TPROXY --on-ip 127.0.0.1 --on-port 7894 --tproxy-mark 1
iptables -t mangle -A clash -p udp -j TPROXY --on-ip 127.0.0.1 --on-port 7894 --tproxy-mark 1
ip6tables -t mangle -A clash -p tcp -j TPROXY --on-ip ::1 --on-port 7894 --tproxy-mark 1
ip6tables -t mangle -A clash -p udp -j TPROXY --on-ip ::1 --on-port 7894 --tproxy-mark 1

# 让打了标的流量进入INPUT链 TProxy Mark和常规Mark都算
ip rule add fwmark 1 table 100 priority 30000
ip route add local 0.0.0.0/0 dev lo table 100
ip -6 rule add fwmark 1 table 100 priority 30000
ip -6 route add local ::/0 dev lo table 100

# 以上是代理局域网设备的, 为了实现代理本机流量, 需要额外操作:
# 由于本机上网流量直接进入OUTPUT链, 后经POST Routing发出, 所以可在 OUTPUT 链对数据包打标记, 让相应的包重路由到 PREROUTING 链上
# 要求: DNS请求以及DST为内网IP的永远直连, 不要重定向
# 代理本机流量有两者方法, 1是过滤指定标记的数据包, 这种方法需要你的代理软件能设置出站流量的数据包标记, 主流软件都支持
# 第二种是gid方案, 让clash以特定uid gid运行, 再在iptables配置指定gid的流量其OUTPUT直连
## 方法1: 打标 (clash对应routing-mark: 255) ipv6类似, 这里省略了
iptables -t mangle -N clash_self
iptables -t mangle -A clash_self -p udp --dport 53 -j RETURN
iptables -t mangle -A clash_self -m set --match-set localnetwork4 dst -j RETURN
iptables -t mangle -A clash_self -j RETURN -m mark --mark 0xff # 0xff即255的十六进制
iptables -t mangle -A clash_self -j MARK --set-mark 1
iptables -t mangle -A OUTPUT -j clash_self
## 方法2: gid过滤
iptables -t mangle -N clash_self
iptables -t mangle -A clash_self -p udp --dport 53 -j RETURN
iptables -t mangle -A clash_self -m set --match-set localnetwork4 dst -j RETURN
iptables -t mangle -A clash_self -j MARK --set-mark 1
## 新建clash用户, 专门运行clash软件, uid=0, gid=250的用户 (如果不想uid=0, 那在启动clash时要给予CAP_NET_ADMIN权限)
## 只有Clash发出的流量能直接OUTPUT, 其余流量进入clash_self子链
iptables -t mangle -A OUTPUT -m owner ! --gid-owner 250 -j clash_self
```

---

## nftables实现 (现代系统方案)
nftables天生就支持ip集合功能, 并且可以一条规则同时匹配IPv4 + IPv6, 免去了近一半操作, 非常好用 <br>
而且, 若ip集合发生变化, nftables能自动处理, 不需要重配路由 <br>
nftables还允许以脚本的形式自动处理规则, 不需要像iptables那样一条一条敲命令. <br>

:::tip
nft中, inet/ip/ip6类型表的优先级对应表:
  - raw     -300
  - mangle  -150
  - dstnat  -100
  - filter  0
  - security    50
  - srcnat  100
:::
```shell title=change_fw.nft
#!/usr/sbin/nft -f
# 指定出口网卡
define out_interface = {eth0}

define add_mark = 1

define tproxy_port = 7894

table inet clash {
    set local_ipv4 {
        type ipv4_addr
        flags interval
        elements = {
            0.0.0.0/8, 10.0.0.0/8, 127.0.0.0/8, 169.254.0.0/16,
            172.16.0.0/12, 192.168.0.0/16, 224.0.0.0/4, 240.0.0.0/4
        }
    }
    set local_ipv6 {
        type ipv6_addr
        flags interval
        elements = {
            ::/128, ::1/128, ::ffff:0:0/96, 64:ff9b::/96, 100::/64,
            2001::/32, 2001:20::/28, 2001:db8::/32, 2002::/16,
            fe80::/10, ff00::/8, fc00::/7
        }
    }
    set geoip_cn_ipv4 {
        type ipv4_addr
        flags interval
    }
    set geoip_cn_ipv6 {
        type ipv6_addr
        flags interval
    }
    # flow offload, 这里可能还需要更改一下
    # flowtable ft {
    #     hook ingress priority 0;
    #     devices = { eth0 };
    # }
    chain allow_forward {
        # 链的type可以为filter, nat, route, 各自能hook的对象不同, 如果你不知道要选哪种就选filter
        # hook的对象只能是prerouting, forward, postrouting, input, output中一种
        # 优先级 priority只对hook对象相同的所有链生效, 如果不同链都hook了不同的对象, 那优先级无作用
        type filter hook forward priority 0; policy accept;
        # ip protocol { tcp, udp } flow offload @ft
        ct state established,related accept
    }

    chain src_nat {
        type nat hook postrouting priority 100; policy accept;
        oifname $out_interface masquerade
        # 如果有多个上网口, 最终具体走哪个口由路由表决定
    }
    # chain的定义中如果有type xxx hook xxx, 表示这是一个基本链, 否则是常规链
    # 在基本链中, return 就相当于该基本链的默认策略
    # 在常规链中, return从常规链返回, 回到调用该常规链的上一个链中继续匹配
    # 如果规则匹配成功, 且动作是accept, 后续规则不会再执行
    chain tmark_incoming_traffic {
        type filter hook prerouting priority -150; policy accept;
        udp dport 53 return         # 绕过经过本机的DNS请求
        ip daddr @local_ipv4 return # 绕过所有内网地址
        ip6 daddr @local_ipv6 return
        ip daddr @geoip_cn_ipv4 return    # 国内IP直连
        ip6 daddr @geoip_cn_ipv6 return
        meta l4proto { tcp, udp } meta mark set $add_mark tproxy ip to 127.0.0.1:$tproxy_port accept
        meta l4proto { tcp, udp } meta mark set $add_mark tproxy ip6 to [::1]:$tproxy_port accept
    }
    # 本机上网流量过滤及打标
    # 容器上网流量不属于本机流量, 其容器相当于都连接到podman0这个接口, 自己就是容器的网关
    chain mark_self_out_traffic {
        type route hook output priority -150; policy accept;
        udp dport 53 return         # 本机发起的DNS请求直连
        ip daddr @local_ipv4 return # 绕过所有内网地址
        ip6 daddr @local_ipv6 return
        ip daddr @geoip_cn_ipv4 return    # 国内IP直连
        ip6 daddr @geoip_cn_ipv6 return
        meta skgid 250 return       # 绕过gid=250的应用发出的所有流量
        meta l4proto { tcp, udp } meta mark set $add_mark accept        # 其他流量重路由到prerouting
    }
}

```
### NFT配套脚本
这个脚本在change_fw.nft执行后运行, 用于向ip集合中添加IP <br>

:::tip
ip route显示默认路由表(main路由表) <br>
ip route show table xxx, 显示xxx路由表 <br>

ip rule 显示系统当前的路由策略/规则, 显示的内容是所有 ip rule show table xxx 的合集 <br>
输出结果中, lookup local代表查找local路由表, 同时也表示该行规则属于local路由表 <br>
:::
```shell title=before_start.sh
#!/bin/bash

# 添加国内IP, OpenClash同款源
(echo "add element inet clash geoip_cn_ipv4 {"; curl -s https://ispip.clang.cn/all_cn.txt | sed 's/$/,/'; echo "}") | nft -f -
(echo "add element inet clash geoip_cn_ipv6 {"; curl -s https://ispip.clang.cn/all_cn_ipv6.txt | sed 's/$/,/'; echo "}") | nft -f -

exit 0
```
systemd service
```
[Service]
Type=exec
User=clash
ExecStartPre=/root/config/clash/change_firewall.nft
ExecStartPre=ip rule add fwmark 1 table 100 priority 30000
ExecStartPre=ip route add local 0.0.0.0/0 dev lo table 100
ExecStartPre=ip -6 rule add fwmark 1 table 100 priority 30000
ExecStartPre=ip -6 route add local ::/0 dev lo table 100
ExecStartPre=/root/config/clash/before_start.sh
ExecStart=/root/config/clash/mihomo-linux-arm64 -f /home/radxa/clash/clash.yaml -d /root/config/clash
ExecStop=/bin/kill $MAINPID
ExecStopPost=nft delete table inet clash
ExecStopPost=ip rule del fwmark 1 table 100
ExecStopPost=ip route del local 0.0.0.0/0 dev lo table 100
ExecStopPost=ip -6 rule del fwmark 1 table 100
ExecStopPost=ip -6 route del local ::/0 dev lo table 100
Restart=on-failure
```

:::tip
start = ExecStartPre + ExecStart
stop = ExecStop + ExecStopPost
restart = stop + start
:::

:::warn
在安装了Docker的系统上, 如果docker使用nftables, 并且docker在clash.service之前运行, docker先添加的防火墙会让后添加的clash防火墙无法正常运行 <br>
直接表现就是断网, 要解决该问题, 需要想办法让clash在docker之前运行, 即在clash.service文件的[Unit]部分添加 `Before=docker.service`
:::
## 使用Tun处理TCP与UDP流量(Redir TCP + TUN UDP)
如果你的设备内核比较老, 或者是那种阉割了一堆功能的内核, 你还改不了的那种, 可以试一试Tun设备方案 <br>
该方案同样需要内核配置启用了tun设备, 如果没有, 建议换设备
```shell
# clash配置文件中的Tun:
## 开启 auto-route: 使用Tun处理tcp 和 udp
### 再开启 auto-redirect: TCP流量使用Redirect而不是tun, 可提高性能; UDP还是用Tun
# 两者都开启时也就是Openclash的增强模式, 但不知为什么OC两者都是false, 明明效果一样的

# Tun模式下, ICMP请求也会被劫持, 如果DST域名是要走代理的fakeip, 那就变成ping fakeip了

# Tun模式auto route启用后, 路由策略(ip rule)相关内容, 从上往下匹配
0:      from all lookup local # 所有流量找local表
9000:   from all to 198.18.0.0/30 lookup 2022 #所有去198.18.0.0/30的流量找2022表, 其内容: default via 198.18.0.2 dev Meta, 即去Meta接口
9001:   not from all dport 53 lookup main suppress_prefixlength 0 # 所有目标端口非53的流量找main表, 但不匹配main表中前缀<=0的路由 (即默认路由)
9001:   from all iif Meta goto 9010 # 所有从Meta接口入站的流量转到9010号策略
9002:   not from all iif lo lookup 2022 # 所有从非lo接口入站的流量找2022表, DNS入站走这里
9002:   from 0.0.0.0 iif lo lookup 2022 # 所有从lo接口入站, 来自0.0.0.0的流量找2022表, 未知用途
9002:   from 198.18.0.0/30 iif lo lookup 2022 # 所有来自198.18.0.0/30, 从lo接口入站的流量找2022表, 即Meta出站
9010:   from all nop

# dns-hijack项的内容是指劫持目标IP是xxx, 目标端口xxx的请求, 但无论怎么填, nft都会把目标53的流量劫持到198.18.0.2, 也许是个BUG?

# 在关闭auto-route后, dns劫持不再生效, clash只会创建tun接口, 要自己配置iptables
```

## Q & A
### 创建uid=0, gid=250的用户
```shell
# 创建250号组, 名字为clash
sudo groupadd -g 250 clash
# 添加用户clash
sudo useradd -u 0 -g 250 -s /bin/false clash


# 更简单的方法是直接编译/etc/passwd文件, 并添加:
clash:x:0:250::/dev/null:/usr/sbin/nologin
```
### 为实现绕过内网, 为什么不直接在mangle表的pre routing链中添加内网return规则?
iptables对数据包只能ACCEPT DROP RETURN或者跳转到子链继续匹配 <br>
默认情况下pre routing一般都是ACCEPT数据包的, 除非你需要屏蔽某些IP; 如果是DROP, 那我怎么代理局域网? 如果是RETURN, 即回到上一链, 但现在已经是pre routing链, 没有上一链, 这种情况下是直接继续匹配 <br>
所以必须创建一个子链, 在子链中配置内网IP直接RETURN到pre routing链, 进而让内网IP可以走FORWAD链

### 什么情况下Iptables需要指定 --protocol 协议?
在指定要操作哪个表哪个链后, 需要指定匹配规则, 如 -p tcp 或者 -s 10.1.1.0/24, 并且规则要明确清晰, 如tcp的200端口, 你不能只写一个200端口 <br>
只有明确匹配规则后才能 -j 指定匹配动作 <br>

-p 默认指all, 代表 tcp, udp, udplite, icmp, icmpv6, esp, ah, sctp, mh 这些协议

### 为什么clash设置监听0.0.0.0:port时, netstat -l 显示是只监听ipv6
/proc/sys/net/ipv6/bindv6only 这个值决定是否开启ipv4 to ipv6的映射, 默认0代表开启, 此时ipv4的地址会被转成 *::ffff:10.1.1.1* 这种形式, 这样一个端口能同时处理IPv4 IPv6

0.0.0.0 代表所有可选地址, 自然也包括IPv6地址

tproxy端口最好只监听127.0.0.1 和 ::1, 防止流量被盗

:::tip
tproxy打标这一步不会更改数据包目标IP和端口, 它是通过在数据包上添加特定标记实现的路由重定向
:::

### 反向DNS解析
通过向DNS请求 xxxx.in-addr.arpa 这种域名, 返回 PTR (rDNS) (需要域名配置了PTR解析, 否则这样的请求无返回结果)

### Tproxy mark和常规Mark
TProxy打标不会触发数据包重路由, mark常规打标会.

### TProxy和Redirect
Tproxy不会修改数据包的目标, 而Redirect会

### DNS
在系统启动时, 由于要联网以下载geoip, 如果此时DNS填127.0.0.1, 会因clash还未启动, 无法提供DNS服务, 导致无法下载geoip
解决方法: 系统启动时就用dhcp获取的DNS, 等clash运行后再执行覆盖命令.

dhcp dns支持情况:
  - 使用NetworkManager: NetworkManager会在启动后填DHCP获取的DNS或者自己设置的DNS到/etc/resolv.conf, 已在debian13 rootfs上验证.
  - netplan + NetworkManager-backend: 常见于ubuntu, 无论怎么设置都不会去动/etc/resolv.conf, 全权由自己控制. 即使在/run/NetworkManager/system-connection下面的配置文件中或者nmtui配置了DNS. 已在ubuntu 24.04上验证
  - netplan + systemd-networkd-backend: 常见于armbian-minimal系统

实测, 如果直接添加如下文本, nm有时会因网络延迟导致clash.service先写dns, 后nm再覆盖DNS
ExecStartPre=echo "nameserver 127.0.0.1" > /etc/resolv.conf
解决方法: 再加一行sleep到这一行之前, 这样做后大概率是NM先写DNS, clash再写DNS

:::TIPS
netplan指定NM为后端时, 生成的配置文件在: /run/NetworkManager/system-connections <br>
nmtui指定的配置, 保存在: /etc/NetworkManager/system-connections
:::

### IPV6
各大主流系统内修改的网关只针对于IPV4生效, 但windows把ipv4和ipv6分开设置了
要想实现旁路由, 就必须把ipv6网关也指向旁路由, 在这些不能修改ipv6网关的设备上, 只能在主路由处配置dhcpv6, 把特定设备分配特定的ipv6网关
并且ipv6网关地址要能随拨号前缀而变化

或者, 使用本地链路地址fe80 ?

## 中转服务器搭建
部分VPS线路太烂, 只能通过中转使用 <br>
而且国内VPS价格还算不错, 最差的阿里云也有99一年3M, 不限流量, 这个应急用下还是可以的, 防止失联 <br>

xray tun转发:
根据xray会监听5600端口, 猜测其是应用层转发, 按理说应该没效果的, 毕竟TLS协商在这之前
但使用v2rayN实测发现, 转发到RN时, 支持TCP/UDP, 转发到KR时, 只能UDP, TCP需要特定的指纹才能连上
使用clash, 无论什么指纹, 都无法使用基于TCP的协议

使用三层转发时:
```shell
echo 1 > /proc/sys/net/ipv4/ip_forward

# TCP流量处理
iptables -t nat -A PREROUTING -i ens5 -p tcp --dport 5600 -j DNAT --to-destination 43.155.173.61:443
iptables -t nat -A POSTROUTING -o ens5 -p tcp -d 43.155.173.61 --dport 443 -j MASQUERADE

# UDP流量处理
iptables -t nat -A PREROUTING -i ens5 -p udp --dport 5600 -j DNAT --to-destination 43.155.173.61:443
iptables -t nat -A POSTROUTING -o ens5 -p udp -d 43.155.173.61 --dport 443 -j MASQUERADE
```
不清楚为何指纹选chrome时才能用, 别的不行, 这情况和Tun转发类似
选别的指纹时, 发送client hello, 中转VPS会直接回复 RST
而clash无论选什么都会回复tcp rst