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

##

## 在 PVE 的 GRUB 引导菜单中添加 Ubuntu 启动项

PVE 系统屏蔽了 os-prober 工具，导致 apt 安装此工具并在 `/etc/default/grub` 文件中添加 `DISABLE=false` 后，输入 `update-grub` 命令也不会自动扫描各个硬盘的操作系统并添加到 PVE 的 GRUB 引导菜单中。因此需要手动添加。编辑 `/etc/grub.d/40_custom` 文件，在其下方输入如下内容，保存后输入 `update-grub` 重新生成 `/boot/grub/grub.cfg` 引导文件。

```bash
#!/bin/sh
exec tail -n +3 $0
# This file provides an easy way to add custom menu entries. Simply type the
# menu entries you want to add after this comment. Be careful not to change
# the 'exec tail' line above.

### 上面的内容不要删除, 在下面添加内容

menuentry 'Ubuntu' --class ubuntu --class gnu-linux --class gnu --class os $menuentry_id_option 'gnulinux-simple-bde2ecbe-cc15-4e31-99f2-f5a57d432174' {
        load_video
        gfxmode $linux_gfx_mode
        insmod gzio
        if [ x$grub_platform = xxen ]; then insmod xzio; insmod lzopio; fi
        insmod part_gpt
        insmod ext2
        set root='hd0,gpt2'
        if [ x$feature_platform_search_hint = xy ]; then
          search --no-floppy --fs-uuid --set=root --hint-bios=hd0,gpt2 --hint-efi=hd0,gpt2 --hint-baremetal=ahci0,gpt2  272dbd32-0415-4623-bbb2-165110ee8aae
        else
          search --no-floppy --fs-uuid --set=root 272dbd32-0415-4623-bbb2-165110ee8aae
        fi
        linux   /vmlinuz-6.8.0-45-generic root=/dev/mapper/ubuntu--vg-ubuntu--lv ro  quiet splash $vt_handoff
        initrd  /initrd.img-6.8.0-45-generic
}
```
