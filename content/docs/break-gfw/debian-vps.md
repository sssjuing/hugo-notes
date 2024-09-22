---
title: Debian VPS 配置
date: "2021-10-30"
weight: 3
---

---

## XRay

- [官方脚本安装的文件](https://github.com/XTLS/Xray-install)
- [模板编辑配置文件](https://github.com/XTLS/Xray-examples)
- [Xray 教程](https://tlanyan.pp.ua/xray-tutorial/)
- [V2rayN 4.12 配置教程](https://v2xtls.org/v2rayn-4-12%E9%85%8D%E7%BD%AE%E6%95%99%E7%A8%8B/)

## 将语言更改成英语

首先输入下列语句，打开 locale 文件。

```shell
sudo vim /etc/default/locale
```

显示如下

```
LANG="zh_CN.UTF-8"
LANGUAGE="zh_CN:zh"
```

将其内容更改成如下内容，然后重启。

```
LANG="en_US.UTF-8"
LANGUAGE="en_US:en"
```

## 配置 sudo

```shell
apt install sudo
chmod 777 /etc/sudoers # 将sudoers修改为可编辑权限
vim /etc/sudoers
```

在`root ALL=(ALL:ALL) ALL`下添加`user ALL=(ALL) ALL`

```shell
chmod 440 /etc/sudoers # 将sudoers改回只读权限
```

## 配置静态 IP

打开`/etc/network/interfaces`文件，写入如下信息

```
iface ens192 inet static
address 192.168.7.220
netmask 255.255.255.0
gateway 192.168.7.1
```

之后输入`reboot`重启。~~之后输入`/etc/init.d/networking restart`重启网络服务。~~

## 修改 GRUB 引导菜单等待时间

编辑`/etc/default/grub`文件，修改**GRUB_TIMEOUT=5** 这一参数值， 且保存退出。之后执行`sudo update-grub`重新生成 GRUB。

## 修改主机名、域名

### 设置主机名

相关命令：使用 `hostname` 查看主机名，使用 `hostname -f` 查看域名

可以直接修改 `/etc/hostname` 文件，或者使用如下命令设置 `hostname` 。

```shell
hostnamectl set-hostname <your_host_name>
```

### 设置域名

修改文件 `/etc/hosts`

```
127.0.0.1     <your_host_name>.debian.local  <your_host_name>
```

## 允许 ssh 远程登录 root 账号

首先，修改 root 密码（可选）。

```shell
sudo passwd root
```

其次，编辑 `/etc/ssh/sshd_config` 文件

```diff
- #PermitRootLogin prohibit-password
+ PermitRootLogin yes
```

之后输入 `sudo systemclt retart sshd` 重启 ssh 服务。

## 查看 IP 是否被封

https://ping.pe/

## Docker 配置国内源

创建`/etc/docker/daemon.json`文件，写入如下内容，之后输入`sudo systemctl restart docker`重启 Docker 服务。

```json
{
  "registry-mirrors": ["https://docker.mirrors.ustc.edu.cn"]
}
```

## Debian 使用 Wifi 联网

需要安装 NetworkManager，他提供了命令行方式 nmcli 和 图形界面方式 nmtui 进行网络连接和管理，避免用户通过修改相关文件进行 wifi 网络连接，方便使用。

首先输入以下命令安装和启动服务

```shell
apt install -y netwok-manager

systemctl start NetworkManager.service
systemctl enable NetworkManager.service
```

随后将配置文件 `/etc/NetworkManager/NetworkManager.conf` 中的内容替换为以下内容。其中 managed=true 使全部网卡纳入到 NetworkManager 的管理中，unmanaged-devices 将那些非 wifi 网络设备排除掉。

```toml
[main]
plugins=ifupdown,keyfile

[keyfile]
unmanaged-devices=*,except:type:wifi,except:type:wwan

[ifupdown]
managed=true
```

之后可输入 `nmcli d` 命令查看设备状态。其他命令可参考[文档](https://docs.redhat.com/zh_hans/documentation/red_hat_enterprise_linux/8/html/configuring_and_managing_networking/configuring-networkmanager-to-ignore-certain-devices_configuring-and-managing-networking)。
