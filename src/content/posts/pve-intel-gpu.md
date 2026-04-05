---
title: PVE Intel核显虚拟化/直通
published: 2026-03-03
description: 'PVE中Intel核显虚拟化的配置和使用'
image: ''
tags: [Linux, Debian, PVE, 虚拟化, Intel, GPU]
category: '教程'
draft: false 
lang: 'zh_CN'
---

# GPU虚拟化
## 更新
```shell
apt update
apt install build-essential dkms sysfsutils intel-gpu-tools -y
apt upgrade -y

# 内核头文件
apt install pve-headers-$(uname -r)

reboot
```
查看当前内核版本:
```shell
uname -r
## 6.17.13-1-pve
```

## 下载DKMS GPU 驱动
去[这里](https://github.com/strongtz/i915-sriov-dkms/releases)下载合适的deb包, 例如 `i915-sriov-dkms_2026.02.09_amd64.deb` <br>
部分包会带有kernel-xxxx的后缀, 代表这是针对特定内核版本的驱动包<br>

安装驱动包, 过程中应该会自动编译内核模块:
```shell
dpkg -i i915-sriov-dkms_2026.02.09_amd64.deb
```
在以后更新内核时, DKMS会自动重新编译驱动模块

## 更新Grub
```shell
nano /etc/default/grub

# 找到GRUB_CMDLINE_LINUX_DEFAULT, 修改其值为 intel_iommu=on i915.enable_guc=3 i915.max_vfs=3 module_blacklist=xe
## 具体是什么参数请见: https://github.com/strongtz/i915-sriov-dkms?tab=readme-ov-file#required-kernel-parameters
```

## VFs
功能未知, 貌似是设置虚拟GPU的数量的
```shell
echo "devices/pci0000:00/0000:00:02.0/sriov_numvfs = 3" > /etc/sysfs.conf
```

## 宿主机上关闭VFs
```shell
echo "vfio-pci" | sudo tee /etc/modules-load.d/vfio.conf

cat /sys/devices/pci0000:00/0000:00:02.0/device
# 输出的值带入到下面命令的 REPLACE_ME 中, 例如我的N5105: 0x4e61

sudo tee /etc/udev/rules.d/99-i915-vf-vfio.rules <<EOF
# Bind all i915 VFs (00:02.1 to 00:02.7) to vfio-pci
ACTION=="add", SUBSYSTEM=="pci", KERNEL=="0000:00:02.[1-7]", ATTR{vendor}=="0x8086", ATTR{device}=="REPLACE_ME", DRIVER!="vfio-pci", RUN+="/bin/sh -c 'echo \$kernel > /sys/bus/pci/devices/\$kernel/driver/unbind; echo vfio-pci > /sys/bus/pci/devices/\$kernel/driver_override; modprobe vfio-pci; echo \$kernel > /sys/bus/pci/drivers/vfio-pci/bind'"
EOF

update-grub && update-initramfs -u
reboot
```

## 验证
```shell
lspci -nnk -s 00:02

# 应该出现多个项目, 且主GPU驱动是i915, 其他的则是vfio-pci
## 我的N5105无法使用SRIOV虚拟化
```

# GPU直通
```shell
nano /etc/modprobe.d/pve-blacklist.conf
# 添加如下内容:
## i915显卡驱动
blacklist i915
## intel声卡驱动
blacklist snd_hda_intel

update-initramfs -u -k all
```
前往: https://github.com/lixiaoliu666/pve-anti-detection/releases 下载 `pve-qemu-kvm_10.1.2-7_amd64.deb`
```shell
dpkg -i pve-qemu-kvm_10.1.2-7_amd64.deb

# 禁止该软件包的更新, 若后续需要换回原版, 把 hold 改为unhold 即可
apt-mark hold pve-qemu-kvm
```
前往 [这里](https://github.com/lixiaoliu666/intel6-14rom/releases) 下载核显Rom文件, 对于N5105, 下载 `6-14-qemu10.rom`
将其移动到 `/usr/share/kvm/` 目录下, 再重启

## 虚拟机创建
机型推荐q35, BIOS选 OVMF(UEFI), CPU模板选择host, 预注册密钥随意, win11可能还需要TPM <br>
之后正常完成虚拟机安装 <br>

安装完成后, 编辑 /etc/pve/qemu-server/虚拟机ID.conf, 添加如下内容:
```text
args: -set device.hostpci0.bus=pcie.0 -set device.hostpci1.bus=pcie.0 -set device.hostpci0.addr=0x02.0 -set device.hostpci1.addr=0x03.0 -set device.hostpci0.x-igd-gms=0x2 -set device.hostpci0.x-igd-opregion=on -set device.hostpci0.x-igd-lpc=on

hostpci0: 0000:00:02.0,romfile=6-14-qemu10.rom
hostpci1: 0000:00:1f.3
```
如果你的系统是Windows这类, 需要与虚拟机硬件交互, 则需要去PVE WebUI上, 把 `硬件` -> `显示` 修改为 `无`, 不然Windows会优先把画面显示在PVE的虚拟显示器上, 而不是核显上 <br>
> Windows注意安装驱动 <br>

如果你只是想调用核显用于视频编解码等用途, 而不需要显示输出, 则不需要像Windows那样修改显示设置.