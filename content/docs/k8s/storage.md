---
title: Storage
date: "2024-09-14"
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
kubectl label namespace kube-system topolvm.io/webhook=ignore
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

然后在命令行输入以下语句安装 TopoLVM ：

```shell
CERT_MANAGER_VERSION=v1.15.1
kubectl apply -f https://github.com/cert-manager/cert-manager/releases/download/${CERT_MANAGER_VERSION}/cert-manager.crds.yaml

helm install -f my-topolvm-values.yaml --namespace=topolvm-system topolvm topolvm/topolvm
```

### 基于 TopoLVM 创建 PVC

安装 TopoLVM 后会自动安装相对应的 SC，名为 `topolvm-provisioner` （其模式为 Delete， 在移除 PVC 时 SC 不会保留其对应的 PV），因此可直接使用此 SC 创建 PVC。

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
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

## Kubernetes 安装基于 NFS 的 Storage Class

首先，需要准备一台 NFS server，其配置过程请参考[这里](/docs/linux/storage/nfs/#nfs-server)。

之后，需要在所有 worker 节点上安装 nfs-common, 否则 kubelet 报错：

```
MountVolume.SetUp failed for volume "nfs-subdir-external-provisioner-root" : mount failed: exit status 32
......
nfs-subdir-external-provisioner-root: bad option; for several filesystems (e.g. nfs, cifs) you might need a /sbin/mount.<type> helper program.
```

随后，输入以下命令安装 nfs-subdir-external-provisioner：

```bash
helm repo add nfs-subdir-external-provisioner https://kubernetes-sigs.github.io/nfs-subdir-external-provisioner/
helm install nfs-subdir-external-provisioner nfs-subdir-external-provisioner/nfs-subdir-external-provisioner \
    --set storageClass.name=nfs-sc-default \
    --set nfs.server=10.9.2.209 \
    --set nfs.path=/mnt/nfs/shared \
    --set storageClass.defaultClass=true \
    --namespace nfs-system --create-namespace
```

其中：

- storageClass.name ：创建的 Storage Class 的名称
- nfs.server ：安装了 nfs server 的服务器 IP
- nfs.path ：在 nfs server 中 /etc/exports 文件内设置的目录
- torageClass.defaultClass ：用于设置此 Storage Class 为默认 Storage Class

完成安装后将如下 "test-pvc-sc.yaml" 文件传入 kubectl 测试是否安装成功，如成功则可以在 NFS server 对应的目录中找到名为 SUCCESS 的文件。

{{<callout type="info">}}
注意，你需要将 storageClassName 设置为上一步中创建的 Storage Class，可通过 `kubectl get sc` 命令查看。
{{</callout>}}

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: my-pvc
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 5Gi
  storageClassName: nfs-sc-default
---
apiVersion: v1
kind: Pod
metadata:
  name: test-pod
spec:
  containers:
    - name: test-pod
      image: busybox:latest
      command:
        - "/bin/sh"
      args:
        - "-c"
        - "touch /mnt/SUCCESS && sleep 3600"
      resources:
        requests:
          cpu: 100m
          memory: 128Mi
        limits:
          cpu: 250m
          memory: 256Mi
      volumeMounts:
        - name: nfs-pvc
          mountPath: "/mnt"
  restartPolicy: "Never"
  volumes:
    - name: nfs-pvc
      persistentVolumeClaim:
        claimName: my-pvc
```
