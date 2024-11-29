---
title: Service 和 Ingress
date: "2024-11-01"
---

---

## MetalLB 安装

首先，执行如下命令，设置 kube-proxy 为严格 ARP 模式：

```bash
kubectl edit configmap -n kube-system kube-proxy
```

打开编辑器后，找到对应的 ARP 配置位置，设置为如下内容：

```yaml
apiVersion: kubeproxy.config.k8s.io/v1alpha1
kind: KubeProxyConfiguration
mode: "ipvs"
ipvs:
  strictARP: true
```

{{<callout type="info">}}
注意，当使用 _kube-router_ 作为 _service-proxy_ 时，由于其默认为严格 ARP 模式，因此无需上述设置。
{{</callout>}}

其次，执行如下语句安装 MetalLB（注意将版本号改为最新版）：

```bash
kubectl apply -f https://raw.githubusercontent.com/metallb/metallb/v0.14.8/config/manifests/metallb-native.yaml
```

也可以选择使用 helm 安装：

```bash
helm repo add metallb https://metallb.github.io/metallb
helm install metallb metallb/metallb --namespace metallb-system --create-namespace
```

此命令将会在 metallb-system 命名空间下安装负载均衡器使用的 controller 和 speaker。

安装完成后 MetalLB 还不能使用（处于 idle 状态），必须将以下 yaml 配置传入 k8s 创建相应的资源后，才能在 Ingress 创建时自动创建负载均衡器并交由其管理。

```yaml
apiVersion: metallb.io/v1beta1
kind: IPAddressPool
metadata:
  name: first-pool
  namespace: metallb-system
spec:
  addresses:
    - 192.168.7.50-192.168.7.70
---
apiVersion: metallb.io/v1beta1
kind: L2Advertisement
metadata:
  name: example
  namespace: metallb-system
spec:
  ipAddressPools:
    - first-pool
```

完成后可输入 `kubectl get ipaddresspool -n metallb-system` 验证 matellb 设置是否生效。

## Nginx-Ingress 安装

执行如下语句，使用 helm 安装 nginx ingress controller（此命令在为安装时安装，已安装时更新）：

```bash
helm upgrade --install ingress-nginx ingress-nginx \
  --repo https://kubernetes.github.io/ingress-nginx \
  --namespace ingress-nginx --create-namespace
```

上述语句将会在 ingress-nginx 命名空间中安装 controller。执行如下语句确认是否安装成功：

```yaml
kubectl get pods --namespace=ingress-nginx
```

将以下配置传入 k8s 测试 Ingress 和 MetalLB 是否能正常工作。输入 `kubectl get ing` 后，对应的 External IP 列中应该是 MetalLB IP 地址池中的值。

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web
spec:
  replicas: 2
  selector:
    matchLabels:
      app: web
  template:
    metadata:
      labels:
        app: web
    spec:
      containers:
        - name: web
          image: sujingclg/monorepo-boilerplate:0.0.1
          resources:
            limits:
              memory: "128Mi"
              cpu: "500m"
          ports:
            - containerPort: 80
---
apiVersion: v1
kind: Service
metadata:
  name: svc-web
spec:
  # type: LoadBalancer
  selector:
    app: web
  ports:
    - port: 80
      targetPort: 80
---
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: nginx-ingress-microk8s
  labels:
    name: nginx-ingress-microk8s
  annotations:
    nginx.ingress.kubernetes.io/ssl-redirect: "false"
spec:
  ingressClassName: nginx
  rules:
    - http:
        paths:
          - pathType: Prefix
            path: "/"
            backend:
              service:
                name: svc-web
                port:
                  number: 80
      # host: myingress.sujing.dev
```

{{<callout type="info">}}
注意：在 microk8s 中，MetalLB 和 Ingress 无法正常工作，Ingress 的 External IP 始终是 127.0.0.1。
{{</callout>}}
