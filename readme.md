# Helm Charts

## 核心概念

三个核心对象：

| 对象 | 是什么 | 例子 |
|------|--------|------|
| **Chart** | 应用的打包格式，包含模板和默认值 | `metallb/` |
| **Repository** | 存放 Chart 的 HTTP 服务器 | `https://metallb.github.io/metallb` |
| **Release** | Chart 的一次部署实例 | `helm install metallb ./metallb` |

## 目录结构

```
charts/metallb/
  Chart.yaml   # chart 元信息 + 依赖声明
  values.yaml  # 覆盖上游默认值
```

本项目用的是 **umbrella chart** 模式：自己写一个轻量 chart，通过 `dependencies` 引用上游 chart，只覆盖需要的值。

## Chart.yaml

```yaml
apiVersion: v2
name: metallb                          # 本地 chart 名
type: application
version: 0.1.0                         # 本地 chart 版本
dependencies:
  - name: metallb                      # 上游 chart 名
    repository: https://...            # 上游 repo 地址
    version: 0.16.1                    # 上游 chart 版本
```

`name` 相同是故意的 — umbrella chart 名和上游 chart 名通常一致，覆盖时通过本地 `values.yaml` 注入自定义配置。

## values.yaml

上游 chart 有自己的默认 `values.yaml`，你的 `values.yaml` 会**合并覆盖**它。例如 metallb 上游有几十项配置，这里只覆写实际的 IP 池和 L2 宣告：

```yaml
ipAddressPools:
  main:
    addresses:
      - 192.168.49.240-192.168.49.250
l2Advertisements:
  main:
    ipAddressPools:
      - main
```

查看上游有哪些值可覆写：`helm show values <repo>/<chart>`

## 另一个例子：ingress-nginx

覆写 controller 配置和 Service 类型：

```yaml
controller:
  config:
    worker-processes: "4"
  service:
    type: LoadBalancer              # metallb 接管后分配外部 IP
  ingressClass: nginx
  ingressClassResource:
    name: nginx
    default: true                   # 设为默认 IngressClass
    controllerValue: k8s.io/ingress-nginx
  watchIngressWithoutClass: true
```

本质一样：上游 chart 定义了几十项配置，`values.yaml` 只覆写需要改的部分。

## 常用操作

```bash
# 拉依赖（下载上游 chart 到 charts/ 子目录）
helm dependency update ./charts/metallb

# 安装
helm install metallb ./charts/metallb -n metallb-system --create-namespace

# 升级（修改 values.yaml 后）
helm upgrade metallb ./charts/metallb -n metallb-system

# 查看已部署 release 的值
helm get values metallb -n metallb-system

# 预览渲染结果（不实际部署，调试用）
helm template metallb ./charts/metallb

# 卸载
helm uninstall metallb -n metallb-system
```

## 部署顺序

metallb 必须在 ingress-nginx 之前部署，因为 ingress-nginx 的 `type: LoadBalancer` 需要 metallb 提供外部 IP：

```bash
# 1. metallb（提供 LoadBalancer IP）
cd charts/metallb
helm dependency update
helm upgrade --install metallb . -n metallb-system --create-namespace

# 2. ingress-nginx（申请 LoadBalancer）
cd ../ingress-nginx
helm dependency update
helm upgrade --install ingress-nginx . -n ingress-nginx --create-namespace

# 3. 验证 metallb 给 ingress-nginx 分配了 IP
kubectl get svc -n ingress-nginx ingress-nginx-controller
```

## 本地开发流程

1. 在 `Chart.yaml` 声明上游依赖（name + repo + version）
2. 运行 `helm dependency update` 拉取依赖
3. 在 `values.yaml` 覆写需要的值
4. `helm install/upgrade` 部署
5. 改 values → `helm upgrade` → 重复