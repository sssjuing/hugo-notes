---
title: 存储
date: "2025-05-09"
---

---

## NFS

[参考链接](https://cn.linux-console.net/?p=30931)

### NFS Server

初始化一台 Debian11 以上版本的主机, 输入 `apt install nfs-kernel-server` 安装相关服务端软件。

之后，创建 "/mnt/nfs/shared" 目录，用作 NFS 共享目录，并通过执行以下命令将此共享目录的所有权更改为 “nobody:nogroup”。

```sh
chown nobody:nogroup /mnt/nfs/shared
```

随后，编辑 "/etc/exports" 文件，在末尾添加以下语句：

```
/mnt/nfs/shared 10.9.22.0/24(rw,sync,no_subtree_check)
```

最后，执行 `systemctl restart nfs-server` 重启服务。

### NFS Client

在客户端机器上执行 `apt install nfs-common` 安装相关软件。安装完成后，新建挂载目录 "/data"，并执行如下命令将其挂载到 NFS 共享目录：

```sh
mount 10.9.22.120:/mnt/nfs/shared /data
```

挂载完成后可输入 `df -h` 验证挂载是否成功。

#### 设置启动时挂载

使用 mount 命令挂载的 NFS 共享目录在每次重启客户机电脑时会中断挂载，所以需要设置启动时挂载。编辑 "/etc/fstab" 文件, 在末尾添加如下语句（将此 IP 改为 NFS Server 的 IP）：

```
10.9.22.120:/mnt/nfs/shared /data nfs4 defaults,sync 0 0
```

随后，执行 `mount -a` 命令以挂载 “/etc/fstab” 配置文件上的所有可用文件系统。
