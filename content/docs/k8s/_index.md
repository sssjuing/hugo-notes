---
title: Kubernetes
date: "2024-09-14"
weight: 3
---

---

## TopoLVM 安装与使用

### Heml 安装 TopoLVM 时指定 volume group

首先，将 TopoLVM 加入到 Helm 仓库，以及在 `K8S` 集群中创建相关标签：

```shell
helm repo add topolvm https://topolvm.github.io/topolvm
helm repo update

kubectl create ns topolvm-system

kubectl label namespace topolvm-system topolvm.io/webhook=ignore
kubectl label namespace kube-system topolvm.io/webhook=ignoree
```

随后，创建 `my-topolvm-values.yaml` 文件并填入以下内容：

```yaml
cert-manager:
  enabled: true
lvmd:
  # lvmd.deviceClasses -- Specify the device-class settings.
  deviceClasses:
    - name: ssd
      volume-group: topolvm-vg # 硬盘中准备部署 topolvm 的 volume group 名称, 要求每个节点都有这个卷组
      default: true
      spare-gb: 20
```

然后在命令行输入以下语句安装 `topolvm` ：

```shell
CERT_MANAGER_VERSION=v1.15.1
kubectl apply -f https://github.com/cert-manager/cert-manager/releases/download/${CERT_MANAGER_VERSION}/cert-manager.crds.yaml

helm install -f my-topolvm-values.yaml --namespace=topolvm-system topolvm topolvm/topolvm
```

### 基于 TopoLVM 创建 PVC

安装 `TopoLVM` 后会自动安装相对应的 SC，名为 `topolvm-provisioner` （其模式为 `Delete`， 在移除 PVC 时 SC 不会保留其对应的 PV），因此可直接使用此 SC 创建 PVC。

```yaml
kind: PersistentVolumeClaim
apiVersion: v1
metadata:
  name: my-pvc
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 1Gi
  storageClassName: topolvm-provisioner
```
