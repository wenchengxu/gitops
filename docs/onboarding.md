# GitOps 实施落地指南

## 1. 目标与边界

本仓库用于统一管理多环境、多集群、多服务的 Kubernetes 发布配置，目标是把发布权限从 Jenkins 剥离到 Argo CD，实现：

- Jenkins 只做 CI（构建镜像 + 更新 GitOps release 文件）
- GitOps 仓库存放期望状态（apps/envs/clusters/releases）
- Helm 负责模板化渲染
- Argo CD 负责同步和自愈

核心边界：

- Jenkins 只能修改 `releases/<env>/<appName>.yaml` 的 `image.tag`（可选 `enabled`、`cluster`）
- Jenkins 禁止直接操作 Kubernetes
- release 环境变更必须 MR 审批

## 2. 仓库分层使用方法

- `apps/`：服务长期基础配置（镜像仓库、端口、基础 ingress/service、owner）
- `envs/`：环境默认配置（副本、资源、HPA 默认值、存储默认值）
- `clusters/`：集群差异配置（domainSuffix、ingress class、storageClass、GPU 能力）
- `releases/<env>/`：每次发布薄配置（当前 tag、目标 cluster、启停）
- `charts/common-*`：按服务类型抽象的 Helm Chart
- `argocd/projects/`：按环境隔离权限
- `argocd/applicationsets/`：按环境从 `releases/<env>/*.yaml` 生成应用

Helm values 合并顺序固定为：

1. `apps/<appName>.yaml`
2. `envs/<env>.yaml`
3. `clusters/<cluster>.yaml`
4. `releases/<env>/<appName>.yaml`

## 3. 分阶段实施步骤

### 阶段 A：基础准备（1-2 天）

1. 确认 7 套集群在 Argo CD 中已完成 cluster 接入。
2. 确认镜像仓库权限（Jenkins 可 push，集群可 pull）。
3. 配置 Git 分支保护：`release` 目录必须 MR 审批。
4. 约定镜像 tag 规则：`<env>-<gitCommitShort>-<buildNumber>`，禁止 `latest`。

完成标准：

- Argo CD 可以访问 GitOps 仓库和目标集群。
- 团队对发布边界达成一致（Jenkins 不碰 K8s）。

### 阶段 B：Chart 能力收敛（2-4 天）

1. 先完成并验证 `common-backend`。
2. 补齐 `common-frontend/common-algorithm/common-gpu/common-worker` 差异能力。
3. 模板不硬编码 env、cluster、namespace、tag、storageClass、域名。
4. 新增字段先在 `values.yaml` 给默认值。

每次改 Chart 必做渲染检查：

```bash
helm template test ./charts/common-backend \
  -f apps/order-service.yaml \
  -f envs/dev.yaml \
  -f clusters/dev-cluster-a.yaml \
  -f releases/dev/order-service.yaml
```

完成标准：

- frontend/backend/gpu 试点服务都能渲染成功。
- 字段为空时模板不报错。

### 阶段 C：Argo CD 接入（1-2 天）

1. 应用 `argocd/projects/dev.yaml`、`test.yaml`、`release.yaml`。
2. 应用 `argocd/applicationsets/dev.yaml`、`test.yaml`、`release.yaml`。
3. `dev` 自动同步 + selfHeal，`release` 先手动同步。
4. 验证 ApplicationSet 从 `releases/<env>/*.yaml` 自动生成应用。

完成标准：

- 修改 `releases/dev/*.yaml` 后能自动滚动发布。
- OutOfSync 能被识别并处理。

### 阶段 D：Jenkins 改造（2-3 天）

1. 在业务 CI 流水线增加“更新 GitOps release 文件”步骤。
2. Jenkins 只做：构建镜像、推镜像、改 `image.tag`、提交 Git。
3. `dev/test` 可直推；`release` 仅允许创建 MR。
4. Jenkins 机器人账号按仓库/分支做最小权限。

推荐核心脚本：

```bash
VALUES_FILE="releases/${ENV}/${APP_NAME}.yaml"
yq -i ".image.tag = \"${IMAGE_TAG}\"" "${VALUES_FILE}"
```

完成标准：

- Jenkins 不再执行 kubectl/helm/argocd 发布命令。
- release 发布链路全部经过 MR 审批。

### 阶段 E：试点到全量（1-2 周）

1. 先迁移 3 个试点服务：`web-portal`、`order-service`、`gpu-infer`。
2. 每批迁移 5-10 个服务，按服务类型复用 chart。
3. 每批次迁移后做一次回滚演练（改回旧 tag）。
4. 稳定后扩展到全量服务。

完成标准：

- 全量服务都采用 `apps/envs/clusters/releases` 分层。
- 回滚全部通过 Git 变更完成。

## 4. 日常操作 SOP

### 新增服务

1. 新增 `apps/<appName>.yaml`。
2. 新增 `releases/dev|test|release/<appName>.yaml`。
3. 确认 `type` 与 chart 映射正确。
4. 执行 `helm template` 渲染检查。
5. 提交 MR。

### 发布新版本

1. Jenkins 生成新镜像并推送。
2. Jenkins 更新 `releases/<env>/<appName>.yaml` 的 `image.tag`。
3. 提交 Git（release 走 MR）。
4. Argo CD 同步并观察健康状态。

### 回滚

1. 把 `releases/<env>/<appName>.yaml` 的 `image.tag` 改回上一版本。
2. 提交 Git。
3. Argo CD 同步回滚。

### 存储变更

1. 判断是长期配置（改 `apps`）还是环境覆盖（改 `releases/<env>`）。
2. 涉及 `storageClassName/accessModes/mountPath/existingClaim` 一律 MR 严审。
3. release 存储变更必须人工审批。

## 5. 验收清单

1. Jenkins 无 Kubernetes 直连发布命令。
2. ApplicationSet values 合并顺序与规范一致。
3. 所有 chart 渲染通过。
4. release 环境已配置 MR 审批。
5. 所有服务不使用 `latest`。
6. 仓库中无 Secret 明文。
7. 至少有一次成功回滚演练记录。

## 6. 高风险项与控制

1. 风险：Jenkins 误改非 tag 字段。控制：CI 仅允许改 `.image.tag`，并做 diff 审核。
2. 风险：生产自动同步误发。控制：release 手动 sync 或强制分支保护 + 强制 MR。
3. 风险：PVC 变更导致数据风险。控制：存储字段改动双人 review。
4. 风险：模板缺省导致渲染失败。控制：新增字段先补默认值并跑 `helm template`。

## 7. 建议里程碑

- Week 1：完成阶段 A/B（基础能力 + chart 收敛）
- Week 2：完成阶段 C/D（Argo CD 接入 + Jenkins 改造）
- Week 3：完成阶段 E（试点迁移 + 回滚演练）
- Week 4+：按批次迁移全量服务

## 8. NFS Provisioner 全流程打通（StorageClass/PV/PVC/RBAC）

本仓库已接入 `nfs-data-provisioner` 的完整 GitOps 流程，关键文件：

1. 应用基础配置：
   - `apps/nfs-data-provisioner.yaml`
2. 三环境发布文件：
   - `releases/dev/nfs-data-provisioner.yaml`
   - `releases/test/nfs-data-provisioner.yaml`
   - `releases/release/nfs-data-provisioner.yaml`
3. worker chart 新增能力：
   - `charts/common-worker/templates/rbac.yaml`
   - `charts/common-worker/templates/storageclass.yaml`
   - `charts/common-worker/templates/pv-static.yaml`
   - `charts/common-worker/templates/pvc-extra.yaml`
4. Argo CD Project 集群级权限：
   - `argocd/projects/dev.yaml`
   - `argocd/projects/test.yaml`
   - `argocd/projects/release.yaml`

发布链路：

1. ApplicationSet 读取 `releases/<env>/nfs-data-provisioner.yaml`
2. 渲染 `common-infrastructure` 生成 `ServiceAccount + RBAC + Deployment + StorageClass`
3. Provisioner Pod 挂载你的 NFS/SFS 根路径（`/persistentvolumes`）
4. 业务应用创建 PVC（`type: pvc`）后，Provisioner 动态创建 PV 并绑定
5. Pod 最终完成挂载

注意事项：

1. `PROVISIONER_NAME` 必须与 `storageClass.provisioner` 完全一致。
2. 每个环境建议使用独立 StorageClass 名称（避免冲突）。
3. `release` 环境默认 `enabled: false`，按审批后再开启。
4. Jenkins 不允许改存储字段，只改 `image.tag`。
