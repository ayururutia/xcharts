# K8sCharts

ArgoCD + Helm 管理集群基础设施。

## Helm 基础概念

| 概念 | 说明 |
|------|------|
| **Chart** | 打包 K8s 资源的单元，含模板和默认值 |
| **values.yaml** | 覆盖 Chart 默认值的配置文件 |
| **Repository** | 存放 Chart 的 HTTP 服务器 |
| **Release** | Chart 部署到集群后的实例 |
| **Template** | 带 `{{.Values.xxx}}` 占位符的 YAML，渲染时填入实际值 |

## 快速开始

本项目基于 ArgoCD，安装和仓库配置见 [docs/argo-cd.md](docs/argo-cd.md)。

## 三层模型

```
AppProject（管权限） → Application（管部署） → Chart（管模板）
```

| 层 | 作用 | 位置 |
|----|------|------|
| AppProject | 限制可操作的资源类型 | `appprojs/` |
| Application | 声明 chart 来源和目标 ns | `apps/` |
| Chart | 包装上游 chart + 自定义模板 | `charts/` |

## 示例一：metallb（umbrella chart 模式）

需自定义 CRD 模板（IPAddressPool、L2Advertisement），同时包装上游 metallb chart。

关键：**不使用** Chart.yaml 的 `dependencies`（会触发 ArgoCD 拉外网 helm repo）。改为手动下载上游 chart，解压到 `charts/` 子目录，直接提交。

```bash
# 拉取上游 chart
helm pull metallb --repo https://metallb.github.io/metallb --version 0.16.1

# 解压到子目录
mkdir -p charts/metallb/charts
tar xzf metallb-0.16.1.tgz -C charts/metallb/charts/

# 提交
git add charts/metallb/charts/ && git commit -m "chore: add metallb upstream chart"
```

`charts/metallb/Chart.yaml`（无 dependencies）：
```yaml
apiVersion: v2
name: metallb
version: 0.1.0
```

`charts/metallb/values.yaml`（自定义 key，由 templates 渲染成 CRD）：
```yaml
ipaddresspools:
  main:
    addresses:
      - 192.168.49.240-192.168.49.250
l2advertisements:
  main:
    ipAddressPools:
      - main
```

`charts/metallb/templates/` 中放自定义 CRD 模板（ipaddresspool.yaml、l2advertisement.yaml），上游模板在 `charts/metallb/charts/metallb/templates/` 中，Helm 自动合并。

`apps/config/metallb.yaml`：
```yaml
source:
  path: charts/metallb
  repoURL: <本仓库地址>
  helm:
    valueFiles:
      - values.yaml
```

## 示例二：ingress-nginx（直引上游模式）

无自定义模板，直接在 Application 中引用上游 Helm chart，values 内联。

`apps/config/ingress-nginx.yaml`：
```yaml
source:
  chart: ingress-nginx
  repoURL: https://kubernetes.github.io/ingress-nginx
  targetRevision: 4.15.1
  helm:
    values: |-
      controller:
        service:
          type: LoadBalancer          # metallb 提供外部 IP
        ingressClassResource:
          name: nginx
          default: true
```

## 部署顺序

metallb 必须先于 ingress-nginx — ingress-nginx 的 `type: LoadBalancer` 依赖 metallb 分配 IP。

## 卸载

```bash
kubectl delete -f apps/config/
kubectl delete -f appprojs/
kubectl delete ns metallb-system ingress-nginx
```