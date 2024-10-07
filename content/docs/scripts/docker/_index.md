---
title: Docker
---

---

输入以下命令，docker pytorch-notebook 容器挂载 Nvidia 驱动并启动，开机自动启动，后台运行，无需登录。

```bash
sudo docker run -it -p 8888:8888 -v "${PWD}":/home/jovyan/work --gpus all -d --restart unless-stopped quay.io/jupyter/pytorch-notebook:cuda12-latest start-notebook.py --IdentityProvider.token=''
```
