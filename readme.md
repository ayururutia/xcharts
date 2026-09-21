# K8sCharts

用 ArgoCD + Helm 管理集群基础设施。

## Helm 基础概念

| 概念 | 是什么 | 类比 |
|------|--------|------|
| **Chart** | 打包 K8s 资源的单元，含模板 + 默认值 | 就像 apt 包 |
| **values.yaml** | 覆盖 Chart 默认值的配置文件 | 就像 `apt install` 时传参数 |
| **Repository** | 存放 Chart 的位置（HTTP 服务器） | 就像 apt 源 |
| **Release** | Chart 部署到集群后的实例 | 一个 chart 可以装多份，每份一个 release |
| **Template** | 带 `{{.Values.xxx}}` 占位符的 K8s YAML | 渲染时 values 填入占位符 |

核心流程：`helm install <name> <chart> -f values.yaml` 把模板渲染成最终 YAML，提交到集群。

## ArgoCD 安装

本项目通过 ArgoCD 自动同步 chart，而不是手动 `helm install`。

```bash
helm repo add argo https://argoproj.github.io/argo-helm
helm repo update argo
helm upgrade --install argocd argo/argo-cd -n argocd --create-namespace

# 获取 admin 密码
kubectl -n argocd get secret argocd-initial-admin-secret \
  -o jsonpath='{.data.password}' | base64 -d && echo

# 访问 UI
kubectl -n argocd port-forward svc/argocd-server 8443:443 --address 0.0.0.0
# https://localhost:8443  用户名 admin
```

安装 ArgoCD 后，应用本仓库的配置：

```bash
kubectl apply -f appprojs/    # 先注册项目权限
kubectl apply -f apps/config/ # 再创建 Application，ArgoCD 自动同步
```

## 三层模型

```
AppProject（管权限） → Application（管部署） → Chart（管模板）
```

| 层 | 作用 | 位置 |
|----|------|------|
| AppProject | 限制 Application 能操作哪些资源类型 | `appprojs/` |
| Application | 声明 chart 来源、部署到哪个 ns | `apps/` |
| Chart | umbrella chart — 包装上游 chart + 自定义模板 | `charts/` |

## 示例一：metallb

**Chart 模式**：自己写轻量 chart，通过 `dependencies` 引用上游，只覆写需要的值。

`charts/metallb/Chart.yaml`：
```yaml
apiVersion: v2
name: metallb
type: application
version: 0.1.0
dependencies:
  - name: metallb
    repository: https://metallb.github.io/metallb
    version: 0.16.1
```

`charts/metallb/values.yaml`：
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

`charts/metallb/templates/ipaddresspool.yaml` 中用 `{{range .Values.ipaddresspools}}` 遍历生成 CRD。

`apps/config/metallb.yaml` — Application 指向本地 chart：
```yaml
source:
  path: charts/metallb
  repoURL: <本仓库地址>
  helm:
    valueFiles:
      - values.yaml
```

## 示例二：ingress-nginx

**直引模式**：无自定义模板，直接在 Application 中内联 values。

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

`charts/ingress-nginx/` 为 umbrella chart 写法，供对比参考。

## 部署顺序

metallb 必须部署在 ingress-nginx 之前 — ingress-nginx 的 `type: LoadBalancer` 需 metallb 分配 IP。