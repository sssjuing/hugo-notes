---
title: Kubernetes 脚本
date: "2024-09-12"
weight: 3
---

---

### Heml 安装 topolvm 时指定 volume group

```shell
helm install -f my-topolvm-values.yaml --namespace=topolvm-system topolvm topolvm/topolvm
```

```yaml
# my-topolvm-values.yaml
cert-manager:
  enabled: true
lvmd:
  # lvmd.deviceClasses -- Specify the device-class settings.
  deviceClasses:
    - name: ssd
      volume-group: topolvm-vg
      default: true
      spare-gb: 10
```
