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

- [Ubuntu 20.04 物理机 QEMU-KVM + Virt-Manager 创建桥接模式的虚拟机 - NavyW - 博客园](https://www.cnblogs.com/whjblog/p/17213359.html)
- [配置 KVM 网络（桥接模式）-基于 KVM 虚拟机安装 MySQL-安装指南-MySQL-开源使能-鲲鹏 BoostKit 数据库使能套件开发文档-鲲鹏社区](https://www.hikunpeng.com/document/detail/zh/kunpengdbs/ecosystemEnable/MySQL/kunpengmysql8017_03_0020.html)
- [Linux 虚拟化-Ubuntu22.04 之 KVM 桥接网络](https://mmy83.online/posts/linux%E8%99%9A%E6%8B%9F%E5%8C%96-ubuntu22.04%E4%B9%8Bkvm%E6%A1%A5%E6%8E%A5%E7%BD%91%E7%BB%9C/)
- [Ubuntu 22.04 KVM 配置网卡桥接\_ubuntu kvm 桥接-CSDN 博客](https://blog.csdn.net/shadow6907/article/details/138283306)
- [ubuntu20.04 为 kvm 设置桥接并配置静态 IP*Linux 笔记*苏老的学习笔记](https://www.sulao.cn/post/841.html)

## 使用 `qemu-img` 转换镜像格式

```bash
qemu-img convert -p -f qcow2 -O raw /folder/image_name.qcow2 /folder/image_name.raw
```

参数说明：

- -p: presenting the conversion progress

- -f: format of the source image

- -O: format of the target image
