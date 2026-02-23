---
title: Linux查看PCIE设备使用通道数及版本
published: 2025-12-29
description: 'Linux查看PCIE设备当前使用了多少通道数及使用的PCIE版本'
image: ''
tags: [Linux, PCIE]
category: '随笔'
draft: false 
lang: ''
---

# 查看PCIE设备的版本及通道数
```shell
# 找到目标设备, 记下其最前面的 XX:XX.X
# 有些设备可能是 XXXX:XX:XX.X 这种格式, 无伤大雅, 把后面的记住就行
lspci

# 输出格式: XX:XX.X aaaa:bbbb:cccc, 只需要bbbb:cccc部分
lspci -n |grep -i XX:XX.X

sudo lspci -n -d bbbb:cccc -vvv |grep -i width
# 该命令输出结果中, LnkCap 代表设备支持的PCIE版本及通道, 如 Speed 16GT/s, Width x4 代表 PCIE 4.0 x 4
# LnkSta为当前使用PCIE情况, Speed 8GT/s (downgraded), Width x2 (downgraded), 代表PCIE3.0 x 2
# 如果该命令无输出结果, 代表设备不使用PCIE
# 5GT -> PCIE 2.0
# 2.5GT -> PCIE 1.0
```

# 个人设备数据
康耐信N5105 四i225V 2.5G网口小主机, 其M.2 PCIE为PCIE3.0 x 2 <br>
攀升千兆超小主机, 为PCIE 3.0 X1