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

## ArgoCD 安装与配置

### 安装

```bash
helm repo add argo https://argoproj.github.io/argo-helm
helm repo update argo
helm upgrade --install argocd argo/argo-cd -n argocd --create-namespace

kubectl -n argocd get secret argocd-initial-admin-secret \
  -o jsonpath='{.data.password}' | base64 -d && echo
```

### GitOps 数据流

本地写代码 → `git push` 远程仓库 → ArgoCD 自动 `git pull` + `helm template` → 同步到集群。

ArgoCD 只看远程仓库，本地改完不 push，集群不会变。Application 中的 `repoURL` 必须是一个 git 能 clone 的地址。

### SSH 配置

```bash
# 注册私钥
kubectl -n argocd create secret generic argocd-ssh-key \
  --from-file=sshPrivateKey=$HOME/.ssh/id_rsa

# 注册 known_hosts（非标准端口需要指定 -p）
kubectl -n argocd apply -f - <<EOF
apiVersion: v1
kind: ConfigMap
metadata:
  name: argocd-ssh-known-hosts-cm
data:
  ssh_known_hosts: |
$(ssh-keyscan -p 2222 172.31.0.11 | sed 's/^/    /')
EOF

# 注册仓库
kubectl -n argocd patch configmap argocd-cm -p \
  '{"data":{"repositories":"[{\"url\":\"ssh://git@172.31.0.11:2222/jaken/xcharts.git\",\"sshPrivateKeySecret\":{\"name\":\"argocd-ssh-key\",\"key\":\"sshPrivateKey\"}}]"}}'

# 重启生效
kubectl -n argocd rollout restart deployment argocd-repo-server
```

### 应用配置

```bash
kubectl apply -f appprojs/
kubectl apply -f apps/config/
```

## 三层模型

```
AppProject（管权限） → Application（管部署） → Chart（管模板）
```

| 层 | 作用 | 位置 |
|----|------|------|
| AppProject | 限制可操作的资源类型 | `appprojs/` |
| Application | 声明 chart 来源和目标 ns | `apps/` |
| Chart | umbrella chart — 引用上游 + 自定义模板 | `charts/` |

## 示例一：metallb（umbrella chart 模式）

自己写轻量 chart，通过 `dependencies` 引用上游 Helm chart，只覆写需要的值。

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

无自定义模板，直接在 Application 中内联 values。

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