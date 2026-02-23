---
title: Systemd教程
published: 2025-12-01
description: '现代Linux的Systemd教程'
image: ''
tags: [Linux, Systemd]
category: '随笔'
draft: false 
lang: ''
---

# Systemd版本
```shell
# ubuntu
systemd --version
# debian
apt search --names-only systemd | grep installed
```

# 查看systemd 启动项
```shell
systemctl list-units
```

# Systemd Service
一个以.service结尾的文件, 放在特定目录后生效 <br>
由Unit Service Install三部分组成:
  - Unit部分指定本service与其他service或者target等之间的依赖与先后顺序, 也可指定本service的描述
  - Service部分指定本service具体执行的内容
  - Install部分只在systemctl enable/disable时有用, 指定本service的安装信息. 例如, 指定multi-user.target, 即在系统启动后再启动本service

## Unit部分
文档: *man systemd.unit* <br>
```shell
# 配置弱依赖关系, 在启动时检查所列的目标是否启动, 如果没有就尝试启动. 无论结果如何, 不影响本service的启动
Wants=xxx.target xxxx.service

# 配置强依赖关系, 在启动时检查所列的目标是否启动, 如果没有就尝试启动, 如果启动失败, 会导致本服务立即失败
Requires=xxx.service mnt-docker.mount mnt.mount

# 类似Requires, 在启动时检查所列的目标是否启动, 如果没有启动会导致本服务立即失败
Requisite=

# 以上的类别都只在服务启动时检查一次, 并不会去跟踪目标的状态
# 如果你的服务运行前提是另一个服务保持运行, 若另一个服务停止, 那本服务必须也停止
# 例如网站和数据库, 网站运行的前提是数据库在运行, 如果数据库失败则必须关闭网站, 这种情况推荐使用:
BindsTo=xxx.service

# After和Before仅指定先后顺序, 不关心目标是否成功
# 如果目标服务不存在, 这一项相当于无效项, systemd会忽视此项
After=network.target xxx.service
Before=xxx.service

# 为了避免某些BUG和歧义, 如果程序有依赖关系, 必须同时指定After和Wants/Requires, 如果你的程序在启动时就要依赖某个service, 推荐设置ExecStartPre=sleep 来强制设置启动延迟, 保证程序正常运行
# 由于TimeOutStartSec的存在, 目标单元默认情况下如果启动的时间过长(90s), 会直接被认为启动失败
# 但systemd无法跟踪程序状态, 它无法知道程序要怎么样才算启动成功, 所以在systemd fork出子进程后, 根据Type的类型来在不同的时间认为程序成功启动
# 后续除非程序exitcode != 0, systemd都会认为程序正常运行/退出, 所以一般不会有过长的启动时间
```

## Service部分
具体文档查看: *man systemd.service* <br>
```shell
# 指定程序类型, 默认为simple
Type=simple
User=用户名(默认为service服务所属用户的权限)
#在主程序运行前需要运行的程序, 比如iptables命令等oneshot(no-daemon)类型
ExecStartPre=xxx
ExecStart=/usr/bin/my_service_executable
# 指定工作目录 (不一定有效, 因为程序不一定看这个, 默认值为 /)
WorkingDirectory=/data/gateway
# 程序最大启动时间, 超过这个时间还没启动的认为程序失败
TimeoutStartSec=5
# 在杀死程序时, 定义程序结束运行的最大秒数, 超过这个时间程序会被发送SIGTERM信号
TimeoutStopSec=5
Restart=always
```
**Type类型:** <br> 
Type=simple <br>
systemd在fork出子systemd进程后, 在执行exec替换本身之前, systemd就认为程序已经启动, 后续若程序退出, systemd会杀死其下的所有进程 <br>
Type=exec <br>
在fork出子systemd进程且子systemd进程exec()调用ExecStart命令成功后，systemd认为该服务启动成功, simple类型的plus版本, 正常用这个即可 <br>
Type=forking <br>
如果ExecStart指定的程序在启动后会自己fork进程到后台运行, 随后本程序作为中间父进程退出, 则适用 forking 类型, 需要声明 PIDFile 让systemd跟踪程序状态, 否则得让systemd去猜主进程是什么 <br>
典型例子是nginx, /usr/sbin/nginx在执行后会自己fork几个进程, 随后nginx进程退出  <br>
Type=oneshot <br>
如果程序只需要运行一次, 不需要在后台驻留, 比如一个脚本程序, 或者一条mount命令, 一条iptables命令, 这类程序适用 oneshot 类型 <br>
该类型可定义多个ExecStart <br>
即使程序后续退出, systemd也认为服务已启动 <br>
> 所有Type类型均可定义多个ExecStartPre

> podman容器由于用systemd负责自启动, 故在创建容器时不需要指定--restart参数, 容器的运行由systemd负责

## Install部分
指定本service的触发方式 <br>
WantedBy=multi-user.target <br>

# Systemd Target
man systemd.target <br>
man systemd.special <br>
```text
network.target: 当接口激活, 并有IP时视为达成
network-online.target: 当所有接口激活, 并有IP时, 有路由时视为达成, 常用, 该目标只会在系统启动时检查一次, 后续不检查
```