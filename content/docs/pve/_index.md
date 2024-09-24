---
title: Proxmox VE
weight: 2
---

---

## PVE 安装 OpenWrt

参考文章：[PVE 安装 Openwrt 作为旁路由加速家庭网络](https://post.smzdm.com/p/a5x7924l/)

1. 首先，上传 openwrt 镜像，将编译好的 xxx-efi.img 镜像文件上传到 PVE 中，上传完成后保存日志文件。

2. 创建虚拟机，设置名字、ID，操作系统选择 "不使用任何介质"，删除磁盘，网络选项卡中取消勾选 "防火墙"，其余默认。

3. 创建完成后，在硬件选项卡中删除 CD/DVD 驱动器。

4. 登录到 PVE 后台，执行以下命令，其中 <VM_ID> 是虚拟机编号，img 文件是第一步中上传的镜像文件，路径从上述保存的日志文件中的 `target file` 字段获取，等待命令执行完毕。

   ```shell
   qm importdisk <VM_ID> /var/lib/vz/template/iso/xxx-efi.img local-lvm
   ```

5. 切换到 Web UI 界面，找到刚才创建的虚拟机，在硬件选项卡中多出一个未使用磁盘，选中后点击编辑，修改为 sata 类型，保存。

6. 在选项选项卡中，修改引导顺序，将上述引入的磁盘调整到第一位，保存。

7. 启动虚拟机，等待安装完成。输入 `vim /etc/config/network` 修改 lan 口 ip，之后输入 `/etc/init.d/network reload` 重载网络。
