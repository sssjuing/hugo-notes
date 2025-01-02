---
title: Docker
date: "2025-01-03"
weight: 6
---

---

## Docker 配置国内源

创建`/etc/docker/daemon.json`文件，写入如下内容，之后输入`sudo systemctl restart docker`重启 Docker 服务。

```json
{
  "registry-mirrors": [
    "https://docker.m.daocloud.io",
    "https://docker.mirrors.ustc.edu.cn"
  ]
}
```

## 安装 pytorch-notebook 容器

首先，确保系统中已经安装 nvidia 驱动，可通过命令行输入 `nvidia-smi` 查看。随后，根据 [此文档](https://docs.nvidia.com/datacenter/cloud-native/container-toolkit/latest/install-guide.html) 安装 nvidia-container-toolkit。然后，输入以下命令部署容器。

```bash
sudo docker run -it -p 8888:8888 -v "${PWD}":/home/jovyan/work --gpus all -d --restart unless-stopped quay.io/jupyter/pytorch-notebook:cuda12-latest start-notebook.py --IdentityProvider.token=''
```

以上命令将 docker pytorch-notebook 容器挂载 Nvidia 驱动并启动，同时设置了开机自动启动，后台运行，无需登录。
