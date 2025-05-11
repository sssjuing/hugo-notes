---
title: KVM
date: "2024-10-15"
---

---

## 在 Ubuntu 中安装 KVM

首先输入以下命令安装相关的软件包

```bash
sudo apt install qemu-kvm libvirt-daemon-system libvirt-clients bridge-utils virt-manager
```

其次，输入以下命令将当前用户添加到相关的用户组，避免操作 kvm 虚拟机时需要 root 权限

```bash
sudo adduser $USER libvirt
sudo adduser $USER kvm
```

输入 `sudo systemctl status libvirtd` 命令检查 libvirt 服务是否启动

如果未启动则输入以下命令将 libvirtd 添加到服务并设置开机自动启动

```bash
sudo systemctl start libvirtd
sudo systemctl enable libvirtd
```

服务启动之后，可以通过 virt-manager 图形化操作 kvm 虚拟机，也可以使用 virsh 命令操作虚拟机，相关命令如下

```bash
virsh list --all # 列出全部虚拟机，包括已启动和未启动的
virsh start vm_name # 启动一个虚拟机
virsh shutdown vm_name # 中止一个虚拟机
virsh dominfo vm_name # 获取虚拟机的信息
virsh edit vm_name # 编辑虚拟机配置
```

## 给 KVM 虚拟机设置网桥

对于 ubuntu 22.04 及以上系统，创建 /etc/netplan/01-network-manager-all.yaml 文件，在其中粘贴以下内容。

```yaml
network:
  version: 2
  renderer: networkd
  ethernets:
    eno1:
      dhcp4: false
      dhcp6: false
  bridges:
    br0:
      interfaces: [eno1]
      dhcp4: false
      addresses: [10.9.2.61/24]
      routes:
        - to: default
          via: 10.9.2.1
      nameservers:
        addresses: [10.9.2.1, 114.114.114.114]
      parameters:
        stp: false
      dhcp6: false
```

之后输入 `netplan try` 测试刚刚创建的 yaml 文件是否正确，输入 `netplan apply` 设置网络。输入 `brctl show` 可以查看网桥情况。

此外，[这篇文章](https://www.cnblogs.com/whjblog/p/17213359.html)讲解了如何 UI 界面设置网桥。

## 使用 `qemu-img` 转换镜像格式

```bash
qemu-img convert -p -f qcow2 -O raw /folder/image_name.qcow2 /folder/image_name.raw
```

参数说明：

- -p: presenting the conversion progress
- -f: format of the source image
- -O: format of the target image

之后将转换好的 `.qcow2` 文件复制到 `/var/lib/libvirt/images/` 目录下，运行 virt-manager，选择 <u>导入现有磁盘映像</u> ，选择刚才的 `.qcow2` 文件。操作系统选择最新的 <u>Generic Linux</u> 即可。后续如有需要，可以在创建好的虚拟机的设置选项卡中设置硬件直通。
