---
title: PVE在关机前安全弹出所有硬盘
published: 2026-03-13
tags: [Linux, Debian, PVE, Proxmox]
category: '随笔'
draft: false
---

# 创建systemd服务
```text title="/etc/systemd/system/safe-eject.service"
[Unit]
Description=Safely eject all disks (external)
After=pve-guests.service udisks2.service
Requires=pve-guests.service udisks2.service
Before=shutdown.target reboot.target halt.target
DefaultDependencies=no

[Service]
Type=oneshot
RemainAfterExit=yes
ExecStop=/root/safe-eject.sh

[Install]
WantedBy=multi-user.target
```
```shell
sudo systemctl daemon-reload
sudo systemctl enable --now safe-eject
```
# 编写sh脚本
```shell title="/root/safe-eject.sh"
#!/bin/bash

DEVICES=("/dev/disk/by-id/ata-BC711_NVMe_SK_hynix_512GB_FYB4N04831260512K" "/dev/disk/by-id/ata-ST2000LM015-2E8174_ZDZNBZ9H")

for dev in "${DEVICES[@]}"; do
    echo "处理设备: $dev"
    
    # 检查设备是否存在
    if [ ! -e "$dev" ]; then
        echo "设备 $dev 不存在，跳过"
        continue
    fi
    
    # 获取该磁盘所有已挂载的分区及其挂载点
    # lsblk 输出格式：NAME MOUNTPOINT（无表头）
    partitions=$(lsblk -ln -o NAME,MOUNTPOINT "$dev" | awk '$2 {print $1, $2}')
    
    if [ -n "$partitions" ]; then
        echo "发现已挂载的分区:"
        while read -r part mountpoint; do
            echo "  分区 /dev/$part 挂载于 $mountpoint，正在卸载..."
            umount "$mountpoint" 2>&1
        done <<< "$partitions"
    else
        echo "设备 $dev 没有已挂载的分区"
    fi

    # 安全断电
    echo "向 $dev 发送关机命令"
    udisksctl power-off -b "$dev" 2>&1
done

echo "$(date): 完成"
```

