# ArgoCD

## 安装

```bash
helm repo add argo https://argoproj.github.io/argo-helm
helm repo update argo
helm upgrade --install argocd argo/argo-cd -n argocd --create-namespace

kubectl -n argocd get secret argocd-initial-admin-secret \
  -o jsonpath='{.data.password}' | base64 -d && echo

# 访问 UI
kubectl port-forward service/argocd-server -n argocd 8080:443
# https://localhost:8080  用户名 admin
```

## GitOps 数据流

本地写代码 → `git push` 远程仓库 → ArgoCD 自动 `git pull` + `helm template` → 同步到集群。

ArgoCD 只看远程仓库，本地改完不 push，集群不会变。Application 中的 `repoURL` 必须是一个 git 能 clone 的地址。

## 注册 SSH 仓库

ArgoCD v3.5 通过 Secret（label `argocd.argoproj.io/secret-type: repository`）注册仓库，不是 configmap。

```bash
cat > /tmp/repo-secret.yaml <<EOF
apiVersion: v1
kind: Secret
metadata:
  name: xcharts-repo
  namespace: argocd
  labels:
    argocd.argoproj.io/secret-type: repository
stringData:
  url: ssh://git@100.127.255.11:2222/jaken/xcharts.git
  sshPrivateKey: |
EOF

while IFS= read -r line; do
  echo "    $line" >> /tmp/repo-secret.yaml
done < ~/.ssh/x

echo '  insecure: "true"' >> /tmp/repo-secret.yaml
kubectl apply -f /tmp/repo-secret.yaml
```

## 应用项目配置

```bash
kubectl apply -f appprojs/
kubectl apply -f apps/config/
```

## 同步

ArgoCD 默认**不自动同步**，需要手动触发（或在 Application 中开启 `automated`）。

**浏览器**：进入 Application 详情页，点击 Sync。

**命令行**：

```bash
kubectl -n argocd patch application <app名> --type merge \
  -p '{"operation":{"initiatedBy":{"username":"admin"},"sync":{"revision":"HEAD"}}}'
```

如果 AppProject 的 `clusterResourceWhitelist` 缺少资源类型（如 `admissionregistration.k8s.io`），sync 会报 `not permitted in project`，需补上对应 group/kind。

## 卸载

```bash
# 先清理应用（readme 卸载步骤）
kubectl delete -f apps/config/
kubectl delete -f appprojs/

# 再卸载 ArgoCD
helm uninstall argocd -n argocd
kubectl delete ns argocd
```