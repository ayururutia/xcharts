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

需自定义 CRD 模板，同时引用上游 metallb chart。

`charts/metallb/Chart.yaml`：
```yaml
apiVersion: v2
name: metallb
version: 0.1.0
dependencies:
  - name: metallb
    repository: https://metallb.github.io/metallb
    version: 0.16.1
```

修改 `Chart.yaml` 的依赖后，本地执行：

```bash
helm dependency update charts/metallb
```

生成 `Chart.lock`（提交）和 `charts/*.tgz`（gitignore 忽略）。ArgoCD 同步时会自动跑 `helm dependency build` 从上游 Helm repo 拉取。

如果集群内不通外网，可暂将 `.tgz` 也提上来

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
# 1. 删除 Application（ArgoCD 会清理部署的资源）
kubectl delete -f apps/config/

# 2. 等 ArgoCD 清理完后，删 namespace 和 AppProject
kubectl delete ns metallb-system ingress-nginx --ignore-not-found
kubectl delete -f appprojs/

# 3. 清理 CRD
kubectl get crd -o name | grep 'metallb\.io\|frrk8s\.metallb\.io' | xargs kubectl delete
```