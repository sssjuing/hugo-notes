---
title: Debian VPS 配置
date: "2021-10-30"
weight: 2
---

## XRay

- [官方脚本安装的文件](https://github.com/XTLS/Xray-install)
- [模板编辑配置文件](https://github.com/XTLS/Xray-examples)
- [Xray 教程](https://tlanyan.pp.ua/xray-tutorial/)
- [V2rayN 4.12 配置教程](https://v2xtls.org/v2rayn-4-12%E9%85%8D%E7%BD%AE%E6%95%99%E7%A8%8B/)

<br/>

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

<br/>

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

<br/>

## 配置静态 IP

打开`/etc/network/interfaces`文件，写入如下信息

```
iface ens192 inet static
address 192.168.7.220
netmask 255.255.255.0
gateway 192.168.7.1
```

之后输入`reboot`重启。~~之后输入`/etc/init.d/networking restart`重启网络服务。~~

<br/>

## 修改 GRUB 引导菜单等待时间

编辑`/etc/default/grub`文件，修改**GRUB_TIMEOUT=5** 这一参数值， 且保存退出。之后执行`sudo update-grub`重新生成 GRUB。

<br/>

## Docker 配置国内源

创建`/etc/docker/daemon.json`文件，写入如下内容，之后输入`sudo systemctl restart docker`重启 Docker 服务。

```json
{
  "registry-mirrors": ["https://docker.mirrors.ustc.edu.cn"]
}
```

<br/>

## 查看 IP 是否被封

https://ping.pe/
