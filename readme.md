# K8sCharts

## 目录结构

```
k8scharts/
├── appprojs/          # ArgoCD AppProject（分组权限边界）
│   ├── config.yaml        # 基础设施组
│   └── default.yaml       # 应用组
├── apps/              # ArgoCD Application（部署声明）
│   ├── config/            # 基础设施
│   │   ├── metallb.yaml
│   │   └── ingress-nginx.yaml
│   └── default/           # 业务应用（空）
└── charts/            # Helm Chart（配置 + 模板）
    ├── metallb/
    │   ├── Chart.yaml
    │   ├── values.yaml
    │   ├── .helmignore
    │   └── templates/
    └── ingress-nginx/
        ├── Chart.yaml
        ├── values.yaml
        └── .helmignore
```

## 三层关系

```
AppProject（管权限） → Application（管部署） → Chart（管模板）
```

| 层 | 是什么 | 例子 |
|----|--------|------|
| **AppProject** | 限制 ArgoCD Application 能操作哪些资源类型 | `config` 组允许 CRD、ClusterRole；`default` 组全放 |
| **Application** | 声明一个 chart 在哪里、部署到哪个 ns | `metallb.yaml` 指向 `charts/metallb`，部署到 `metallb-system` |
| **Chart** | umbrella chart — 依赖上游 + 自定义模板 + 覆写 values | `charts/metallb/` 包装 metallb 上游 + 生成 IPAddressPool CRD |

## Chart 模式

本项目用 **umbrella chart**：自己写一个轻量 chart，通过 `dependencies` 引用上游，只覆写需要的值。

### Chart.yaml

```yaml
apiVersion: v2
name: metallb
type: application
version: 0.1.0
dependencies:
  - name: metallb                      # 上游 chart 名
    repository: https://metallb.github.io/metallb
    version: 0.16.1                    # 上游 chart 版本
```

### values.yaml

上游 chart 有自己的默认 values，你的 `values.yaml` 会**合并覆盖**。自定义 key 由 `templates/` 渲染：

```yaml
# metallb — 自定义 key，由 templates/ 生成 CRD
ipaddresspools:
  main:
    addresses:
      - 192.168.49.240-192.168.49.250
l2advertisements:
  main:
    ipAddressPools:
      - main
```

```yaml
# ingress-nginx — 透传上游 key
controller:
  service:
    type: LoadBalancer              # metallb 接管后分配外部 IP
  ingressClassResource:
    name: nginx
    default: true
```

### templates/

自定义资源由 templates 生成，不依赖上游模板：

```yaml
# templates/ipaddresspool.yaml
{{`{{- range $k, $v := .Values.ipaddresspools }}
apiVersion: metallb.io/v1beta1
kind: IPAddressPool
metadata:
  name: {{ $k }}
spec:
  {{- toYaml $v | nindent 2 }}
---
{{- end }}`}}
```

## 部署

### 手动 Helm

```bash
# 拉依赖
cd charts/metallb && helm dependency update

# 安装
helm upgrade --install metallb . -n metallb-system --create-namespace
```

### 通过 ArgoCD

```bash
# 先部署 AppProjects
kubectl apply -f appprojs/

# 再部署 Applications（ArgoCD 会自动同步）
kubectl apply -f apps/config/
```

## 部署顺序

metallb 先于 ingress-nginx，因为 ingress-nginx 的 `type: LoadBalancer` 需要 metallb 提供 IP。

## Application 两种写法

**本地 chart**（有自定义模板，如 metallb）：

```yaml
source:
  path: charts/metallb
  repoURL: <本仓库>
  helm:
    valueFiles:
      - values.yaml
```

**直引上游**（无自定义模板，values 内联）：

```yaml
source:
  chart: ingress-nginx
  repoURL: https://kubernetes.github.io/ingress-nginx
  targetRevision: 4.15.1
  helm:
    values: |-
      controller:
        service:
          type: LoadBalancer
```