---
title: PVE 显卡直通
date: "2024-09-16"
---

---

## 开启 IOMMU 功能

首先编辑 `/etc/default/grub` 文件，修改变量 `GRUB_CMDLINE_LINUX_DEFAULT` 为 `quiet intel_iommu=on iommu=pt`，随后输入以下命令更新 `grub`。

```shell
update-grub
```

随后输入以下命令确认是否开启了 `IOMMU` 功能，并检查输出内容中确认是否有类似 `DMAR: IOMMU enabled` 语句，如没有则说明有错误导致未成功开启。

```shell
dmesg | grep -e DMAR -e IOMMU
```

之后编辑 `/etc/modules`，加入以下语句开启 `vfio` 模块

```textile
vfio
vfio_iommu_type1
vfio_pci
vfio_virqfd #not needed if on kernel 6.2 or newer
```

输入以下语句更新 `initramfs`，并检查模块是否成功引入。

```shell
update-initramfs -u -k all # 更新 initramfs
lsmod | grep vfio # 查看是否成功引入 vfio 模块
```

最后输入以下语句验证是否启用 `IOMMU` 中断重映射。

```shell
dmesg | grep remapping
```

查看输出是否为以下内容，如是则 `IOMMU` 中断重映射启动成功。此时，除显卡以外的其他 `PCI-E` 设备均可直通。

```textile
AMD-Vi: Interrupt remapping enabled
DMAR-IR: Enabled IRQ remapping in x2apic mode ('x2apic' can be different on old CPUs, but should still work)
```

输入以下语句检查 IOMMU 是否启动成功，且能否检测到目标硬件

```shell
pvesh get /nodes/{nodename}/hardware/pci --pci-class-blacklist ""
# 将 nodename 替换为你的目标 PVE 节点，一般是 pve
```

输出的表格中包含 `iommugroup` 列，则说明 `IOMMU` 启动成功。

| class    | device | id           | iommugroup | vendor | device_name                                 |
| -------- | ------ | ------------ | ---------- | ------ | ------------------------------------------- |
| 0x010601 | 0xa353 | 0000:00:17.0 | 9          | 0x8086 | Cannon Lake Mobile PCH SATA AHCI Controller |
| 0x010802 | 0x0009 | 0000:02:00.0 | 14         | 0x1e0f | NVMe SSD                                    |
| 0x020000 | 0x15bb | 0000:00:1f.6 | 13         | 0x8086 | Ethernet Connection (7) I219-LM             |

## 在宿主机上禁用 GPU 设备

首先，需要执行以下语句将显卡的设备 ID 添加到 vfio-pci 的 options 中，同时禁用 Linux 内核的 VGA 仲裁器(vga arbitration)。设备的 ID 号可以使用 `lspci -nn` 命令获取。

```shell
echo "options vfio-pci ids=10de:1cbc,10de:0fb9 disable_vga=1" > /etc/modprobe.d/vfio.conf
```

{{<callout type="warning">}}
注意：这里需要将 Nvidia 的显卡和 HDMI 声卡同时在宿主机上禁用，否则直通显卡后无法启动虚拟机。
{{</callout>}}

随后，执行以下命令将显卡设备的驱动名称添加到宿主机驱动黑名单(/etc/modprobe.d/pve-blacklist.conf)中，避免宿主机自动根据探测到的硬件设备加载这些驱动。驱动的名称可以通过 `lspci -k` 命令获取，或者执行 `lspci -k | grep -A 3 "VGA"` 命令过滤出显卡相关的信息。

```shell
echo "blacklist nouveau" >> /etc/modprobe.d/pve-blacklist.conf
echo "blacklist nvidia*" >> /etc/modprobe.d/pve-blacklist.conf
echo "blacklist snd_hda_intel" >> /etc/modprobe.d/pve-blacklist.conf # Nvidia P600 的 HDMI 声卡驱动
```

部分情况下，您可能还需要执行以下语句，以便在加载 vfio-pci 之前设置一个 soft dependency 来加载 GPU 模块。

```shell
echo "softdep nvidiafb pre: vfio-pci" > /etc/modprobe.d/nvidiafb.conf
```

之后，执行如下命令更新模块，并重启 PVE。

```shell
update-initramfs -u -k all
reboot
```

最后，可以通过执行 `lspci -nnk` 命令验证上述设置是否生效，检查输出中所有和目标显卡相关的项目里有如下语句，或者不存在此语句。

```
Kernel driver in use: vfio-pci
```

## 创建虚拟机并直通设备

注意显卡选择 **标准 VGA**，机型选择 **q35**，以便能使用 PCI-E 直通。直通时勾选 "所有功能"，"ROM-Bar"，"PCI-Express"。BIOS 选择 OVMF(UEFI)。

完成 debian 安装后，启动虚拟机时按 Esc 键进入虚拟机的 BIOS，关闭安全模式，reset 重启。

进入虚拟机后，先修改 `/etc/apt/sources.list` 文件，确保每一行的后面有 `contrib non-free non-free-firmware`，随后输入命令 `sudo apt update` 更新源，并输入 `sudo apt install nvidia-detect` 安装 Nvidia Detect Utility，以便用其检查 GPU 型号并获取合适的驱动安装建议。

之后根据 nvidia-detect 给出的建议，输入 `sudo apt install nvidia-driver` 安装驱动。

## 参考文章

- [PCI Passthrough - Proxmox VE](https://pve.proxmox.com/wiki/PCI_Passthrough)

- [Proxmox VE Administration Guide](https://pve.proxmox.com/pve-docs/pve-admin-guide.html#qm_pci_passthrough)

- [PVE 直通显卡 & Intel SRIOV - MAENE - 博客园](https://www.cnblogs.com/MAENESA/p/18005241)

- [PCI(e) Passthrough](<https://pve.proxmox.com/wiki/PCI(e)_Passthrough>)

- [Proxmox VE(PVE)直通显卡 踩坑经验](https://qiedd.com/669.html)

- [NvidiaGraphicsDrivers](https://wiki.debian.org/NvidiaGraphicsDrivers)

- [NVIDIA Drivers for Debian 12 - Step by Step](https://www.reddit.com/r/linux4noobs/comments/18n34c3/nvidia_drivers_for_debian_12_step_by_step/)
