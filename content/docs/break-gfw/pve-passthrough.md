---
title: PVE 显卡直通
date: "2024-09-16"
weight: 4
---

## 开启IOMMU功能

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

编辑 `/etc/modprobe.d/pve-blacklist.conf` 将 NVIDIA 设备加入宿主机黑名单，然后重启宿主机。

```shell
echo "blacklist nouveau" >> /etc/modprobe.d/pve-blacklist.conf
echo "blacklist nvidia*" >> /etc/modprobe.d/pve-blacklist.conf
```



参考文章

- [PCI Passthrough - Proxmox VE](https://pve.proxmox.com/wiki/PCI_Passthrough)

- [Proxmox VE Administration Guide](https://pve.proxmox.com/pve-docs/pve-admin-guide.html#qm_pci_passthrough)

- [PVE直通显卡 & Intel SRIOV - MAENE - 博客园](https://www.cnblogs.com/MAENESA/p/18005241)

- [PCI(e) Passthrough](https://pve.proxmox.com/wiki/PCI(e)_Passthrough)
