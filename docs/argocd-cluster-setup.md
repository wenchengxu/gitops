# Argo CD 对接 Kubernetes 集群

## 1. 对接原理

Argo CD 通过 Kubernetes API 与集群通信，本质是管理 kubeconfig 中的 cluster credential。

```
Argo CD (运行在某套 K8s 中)
    │
    ├── 同集群：直接用 ServiceAccount 调用 K8s API
    │
    └── 外部集群：通过存储的 kubeconfig 调用远端 K8s API
```

## 2. 对接方式

### 2.1 同集群（默认，零配置）

Argo CD 部署在哪套 K8s，就自动能管理那套集群。当前仓库中：

```yaml
# clusters/dev-cluster-a.yaml
server: https://kubernetes.default.svc    # 指向 Argo CD 所在集群的内部 API
```

ApplicationSet 中：

```yaml
destination:
  name: '{{cluster}}'          # 对应 Argo CD 中注册的集群名称
  namespace: '{{envName}}-{{appName}}'
```

当 `cluster` 值为 `https://kubernetes.default.svc` 或 Argo CD 所在集群时，无需额外配置。

### 2.2 外部集群（需要手动注册）

#### 步骤1：在目标集群上创建 ServiceAccount

```bash
kubectl create namespace argocd-manager

# 创建 ServiceAccount
kubectl create serviceaccount argocd-manager -n argocd-manager

# 绑定 cluster-admin（生产环境应收敛权限，见第5节）
kubectl create clusterrolebinding argocd-manager-binding \
  --clusterrole=cluster-admin \
  --serviceaccount=argocd-manager:argocd-manager
```

#### 步骤2：获取 Token 和 CA

K8s 1.24+ 需要手动创建 Secret 来获取长期 token：

```bash
# 创建 Secret
kubectl apply -n argocd-manager -f - <<EOF
apiVersion: v1
kind: Secret
metadata:
  name: argocd-manager-token
  annotations:
    kubernetes.io/service-account.name: argocd-manager
type: kubernetes.io/service-account-token
EOF

# 提取 token、CA、API Server 地址
TOKEN=$(kubectl get secret argocd-manager-token -n argocd-manager -o jsonpath='{.data.token}' | base64 -d)
CA=$(kubectl config view --raw -o jsonpath='{.clusters[0].cluster.certificate-authority-data}')
SERVER=$(kubectl config view --raw -o jsonpath='{.clusters[0].cluster.server}')
```

#### 步骤3：在 Argo CD 中注册集群

**方法A：用 argocd CLI（推荐）**

```bash
# 先登录 Argo CD
argocd login <argocd-server-address>

# 从 kubeconfig 中添加集群
argocd cluster add <context-name-from-kubeconfig>
```

**方法B：手动创建 Secret（更灵活，适合自动化）**

```bash
kubectl apply -n argocd -f - <<EOF
apiVersion: v1
kind: Secret
metadata:
  name: dev-cluster-a
  labels:
    argocd.argoproj.io/secret-type: cluster
type: Opaque
stringData:
  name: dev-cluster-a
  server: https://10.110.128.13:6443
  config: |
    {
      "bearerToken": "${TOKEN}",
      "tlsClientConfig": {
        "insecure": false,
        "caData": "${CA}"
      }
    }
EOF
```

#### 步骤4：验证注册成功

```bash
argocd cluster list
# NAME             SERVER                     VERSION  STATUS
# dev-cluster-a    https://10.110.128.13:6443          Successful
# in-cluster       https://kubernetes.default.svc      Successful
```

## 3. 对应到当前仓库

当前仓库有 7 个集群配置，对接方式取决于实际架构：

### 3.1 单集群多命名空间（最常见）

所有环境在同一套 K8s 的不同 namespace 中运行，Argo CD 只需注册 1 个集群。

```yaml
# clusters/dev-cluster-a.yaml
server: https://kubernetes.default.svc

# clusters/test-cluster-a.yaml
server: https://kubernetes.default.svc

# clusters/release-cluster-a.yaml
server: https://kubernetes.default.svc
```

ApplicationSet 用 namespace 隔离环境：

```yaml
namespace: '{{envName}}-{{appName}}'   # dev-order-service / test-order-service
```

### 3.2 多集群

dev/test/release 在不同 K8s 集群中运行，Argo CD 需注册多个集群。

```yaml
# clusters/dev-cluster-a.yaml
server: https://10.110.128.13:6443     # dev 集群 API

# clusters/test-cluster-a.yaml
server: https://10.110.128.14:6443     # test 集群 API

# clusters/release-cluster-a.yaml
server: https://10.110.128.15:6443     # release 集群 API
```

需要在 Argo CD 所在集群中为每个外部集群创建 Secret。

## 4. AppProject 权限控制

当前仓库已配置了 AppProject，它控制 Argo CD 能往哪些集群的哪些 namespace 部署：

```yaml
# argocd/projects/dev.yaml
spec:
  destinations:
    - name: '*'              # 允许的集群名（* 表示所有）
      namespace: dev-*       # 只能部署到 dev- 开头的 namespace
  clusterResourceWhitelist:  # 允许创建的集群级资源
    - group: rbac.authorization.k8s.io
      kind: ClusterRole
    - group: storage.k8s.io
      kind: StorageClass
```

生产环境应收敛 `destinations.name` 为具体集群名，不用 `*`：

```yaml
# argocd/projects/release.yaml（推荐收敛写法）
spec:
  destinations:
    - name: release-cluster-a
      namespace: release-*
    - name: release-gpu-cluster
      namespace: release-*
```

## 5. 权限收敛建议

### 5.1 Argo CD 对接集群的 SA 权限

生产环境不建议使用 `cluster-admin`，应创建自定义 ClusterRole：

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: argocd-manager
rules:
  - apiGroups: ["*"]
    resources: ["*"]
    verbs: ["get", "list", "watch"]
  - apiGroups: ["", "apps", "batch", "networking.k8s.io", "storage.k8s.io"]
    resources: ["*"]
    verbs: ["create", "update", "patch", "delete"]
  - apiGroups: ["rbac.authorization.k8s.io"]
    resources: ["clusterroles", "clusterrolebindings"]
    verbs: ["get", "list", "watch", "create", "update", "patch", "delete"]
```

### 5.2 Argo CD 自身的 RBAC

在 `argocd-cm` ConfigMap 中配置 Argo CD 的用户权限：

```yaml
# argocd-rbac-cm
apiVersion: v1
kind: ConfigMap
metadata:
  name: argocd-rbac-cm
  namespace: argocd
data:
  policy.default: role:readonly
  policy.csv: |
    p, role:devops, applications, *, dev/*, allow
    p, role:devops, applications, *, test/*, allow
    p, role:admin, applications, *, release/*, allow
```

## 6. 完整对接检查清单

| # | 检查项 | 状态 |
|---|--------|------|
| 1 | Argo CD 所在集群已部署并正常运行 | |
| 2 | 目标集群 API Server 网络可达（443 端口） | |
| 3 | 目标集群已创建 ServiceAccount 和 RBAC | |
| 4 | 已获取 token 和 CA 证书 | |
| 5 | 已在 Argo CD 中注册集群（Secret 或 CLI） | |
| 6 | `argocd cluster list` 显示集群状态为 Successful | |
| 7 | AppProject 的 destinations 包含目标集群和 namespace | |
| 8 | ApplicationSet 的 cluster 字段与注册的集群名一致 | |
| 9 | 试部署一个应用验证同步成功 | |

## 7. 常见问题

### Q: Argo CD 报 `Unable to connect to cluster`

1. 检查网络连通性：`curl -k https://<api-server>:6443/healthz`
2. 检查 token 是否过期：`kubectl auth can-i list pods --as=system:serviceaccount:argocd-manager:argocd-manager`
3. 检查 Secret 中的 `caData` 是否正确

### Q: ApplicationSet 生成的 Application 报 `project not found`

确认 `argocd/projects/<env>.yaml` 已在 Argo CD 中应用：

```bash
kubectl apply -f argocd/projects/dev.yaml
```

### Q: 集群级资源（StorageClass/ClusterRole）同步失败

确认 AppProject 的 `clusterResourceWhitelist` 包含对应资源类型，且 Argo CD 的 SA 有集群级权限。

### Q: 多集群场景下 namespace 自动创建失败

确认 ApplicationSet 的 `syncOptions` 包含 `CreateNamespace=true`，且 Argo CD 的 SA 在目标集群有 namespace 创建权限。
