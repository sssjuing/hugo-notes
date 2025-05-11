---
title: SMB
date: "2025-05-11"
---

---

## 简介

SMB（Server Message Block）是一种用于网络文件共享的协议，允许不同操作系统之间的计算机共享文件、打印机、串行端口等资源。Samba 是一个开源的软件套件，它实现了 SMB 协议，使得 Linux 和类 Unix 系统能够与 Windows 系统无缝集成。

Samba 是一个开源的软件套件，实现了 SMB 协议，使得 Linux 和类 Unix 系统能够与 Windows 系统无缝集成。

## 安装与配置

首先，执行以下命令安装 samba 服务器

```sh
sudo apt update
sudo apt install samba
```

之后执行 `sudo mkdir /srv/smb/shared` 创建一个目录用于共享，并执行以下命令设置合适的权限。

```sh
sudo chmod -R 777 /srv/samba/shared
sudo chown -R nobody:nogroup /srv/smb/shared
```

随后，编辑 samba 的配置文件 "/etc/samba/smb.conf", 在文件末尾添加以下语句：

```ini
[shared]
path = /srv/samba/shared
available = yes
valid users = @smbuser
read only = no
browsable = yes
public = yes
writable = yes
```

其中，`[shared]` 说明这个目录在远程访问时的目录名为 "shared"，"@smbuser" 为下一步要配置的 SMB 用户。

## 配置用户

首先，需要在宿主机系统上新建一个不登录系统的用户，用于 smb 远程访问。

```sh
sudo useradd -r -s /usr/sbin/nologin smbuser
```

参数说明:

- `-r`：
  创建一个系统用户（system user）。系统用户通常用于运行服务或守护进程，而不是用于登录系统。系统用户的 UID（用户 ID）通常在较低的范围内（如 1-999），具体范围取决于系统的配置。
- `-s /usr/sbin/nologin`：
  指定用户的登录 shell 为 /usr/sbin/nologin。/usr/sbin/nologin 是一个特殊的 shell，它不允许用户登录系统。这通常用于那些不需要登录但需要运行服务的用户账户，例如 Samba 用户。

之后，执行以下命令，将刚才创建的用户添加到 SMB 用户列表中。

```sh
sudo smbpasswd -a smbuser
```

## 启动和访问

输入以下命令重启 SMB 服务，以使刚才的设置生效。

```sh
sudo systemctl restart smbd
```
