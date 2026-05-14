# Storage Policy（存储挂载流程）

## 1. 目标

在 GitOps 模式下，统一规范存储挂载（persistence）变更流程，确保：

- 挂载配置只通过 Git 变更，不通过集群手改
- 配置按分层结构管理（`apps/envs/clusters/releases`）
- 生产环境高风险变更必须 MR 审批
- Jenkins 不参与存储字段变更

## 2. 职责边界

- 开发/平台工程师：提交存储配置变更 MR
- Reviewer（至少 1-2 人）：重点审核数据风险字段
- Jenkins：只更新 `releases/<env>/<appName>.yaml` 中 `image.tag`，禁止改 persistence
- Argo CD：根据 Git 最终状态执行同步

## 3. 分层落点规则（最重要）

存储配置应该落在哪一层：

1. 长期、跨环境通用挂载：写入 `apps/<appName>.yaml`
2. 某个环境的特殊挂载：写入 `releases/<env>/<appName>.yaml`
3. 环境默认存储策略（默认存储类/容量）：写入 `envs/<env>.yaml` 的 `persistenceDefaults`
4. 集群差异（如集群默认 `storageClassName` 能力）：写入 `clusters/<cluster>.yaml`

禁止做法：

- 在 Helm 模板里写死 `storageClassName`、`mountPath`、PVC 名称
- 为每个环境复制一份完整 values（破坏分层）

## 4. 标准变更流程（SOP）

### 步骤 1：需求分类

先判断这次挂载变更属于哪类：

1. 新增挂载（不影响历史数据）
2. 调整容量（同存储后端扩容）
3. 替换挂载路径/声明名（可能影响读写）
4. 存储后端切换（如 NFS -> Ceph）
5. 删除挂载/PVC（高风险）

### 步骤 2：风险分级

高风险（必须人工重点评审）：

1. 修改 `storageClassName`
2. 修改 `accessModes`
3. 修改 PVC 名称
4. 修改 `existingClaim`
5. 修改 `mountPath`
6. 删除 volume/PVC
7. 切换存储后端

中低风险（仍需 MR，但可标准评审）：

1. 新增不影响旧路径的挂载
2. `emptyDir` 参数调优
3. 新增只读 `configMap/secret` 挂载

### 步骤 3：按分层修改文件

常见场景与文件：

1. 服务长期新增上传目录：改 `apps/<appName>.yaml`
2. 仅 release 需要更大模型盘：改 `releases/release/<appName>.yaml`
3. 环境级默认容量策略调整：改 `envs/release.yaml`

### 步骤 4：渲染校验（必做）

每次存储改动后必须执行 `helm template`：

```bash
helm template test ./charts/common-backend \
  -f apps/order-service.yaml \
  -f envs/dev.yaml \
  -f clusters/dev-cluster-a.yaml \
  -f releases/dev/order-service.yaml
```

如涉及多类型服务（gpu/algorithm/worker），需分别渲染对应 chart。

### 步骤 5：MR 审批

MR 描述必须包含：

1. 变更类型（新增/修改/删除）
2. 影响环境（dev/test/release）
3. 数据影响评估（是否迁移、是否有中断）
4. 回滚方案（改回旧 tag/旧配置）
5. 验证结果（helm template 成功）

release 环境的存储变更必须人工审批通过后合并。

### 步骤 6：Argo CD 同步与验证

1. 合并后由 Argo CD 同步
2. 检查 Pod 挂载是否生效
3. 检查业务路径读写与权限
4. 检查磁盘容量与事件告警

## 5. 标准字段结构

统一字段：

```yaml
persistence:
  enabled: true
  volumes: []
```

支持类型：

- `pvc`
- `existingPvc`
- `emptyDir`
- `configMap`
- `secret`
- `hostPath`（生产慎用）
- `nfs`（可用于腾讯云 SFS 按项目直挂）

### 5.1 pvc 示例

```yaml
persistence:
  enabled: true
  volumes:
    - name: upload-data
      type: pvc
      mountPath: /data/upload
      readOnly: false
      pvc:
        create: true
        storageClassName: nfs-client
        accessModes:
          - ReadWriteOnce
        size: 20Gi
```

### 5.2 existingPvc 示例

```yaml
persistence:
  enabled: true
  volumes:
    - name: model-data
      type: existingPvc
      existingClaim: model-data-pvc
      mountPath: /models
      readOnly: true
```

### 5.3 emptyDir 示例

```yaml
persistence:
  enabled: true
  volumes:
    - name: shm
      type: emptyDir
      mountPath: /dev/shm
      emptyDir:
        medium: Memory
        sizeLimit: 8Gi
```

### 5.4 腾讯云 SFS（NFS）直挂示例

当你希望“每个项目在 values 里直接输入 SFS 地址与路径”时，使用 `type: nfs`：

```yaml
persistence:
  enabled: true
  volumes:
    - name: sfs-data
      type: nfs
      mountPath: /data
      subPath: ""
      readOnly: false
      nfs:
        server: 10.110.128.13
        path: /bayuegua-nfs/order-service
        readOnly: false
```

说明：

1. `mountPath` 是容器内路径（映射目标）。
2. `nfs.server` 和 `nfs.path` 是 SFS 挂载源。
3. 同服务跨环境不同 SFS 建议放到 `releases/<env>/<appName>.yaml` 覆盖，避免污染全局基础配置。

### 5.5 分层落点示例（SFS）

长期固定 SFS（所有环境一致）：

- 写入 `apps/<appName>.yaml`

仅 release 环境使用独立 SFS：

- 写入 `releases/release/<appName>.yaml`

## 6. Jenkins 与存储变更隔离

Jenkins 仅允许变更：

```yaml
image:
  tag: xxx
```

Jenkins 禁止变更：

- `persistence`
- `storageClassName`
- `accessModes`
- `mountPath`
- `existingClaim`
- `resources`
- `ingress`
- `service`

建议在 CI 增加守卫：当检测到 Jenkins 提交包含上述字段 diff 时直接失败。

## 7. 回滚流程

推荐回滚优先级：

1. 回滚 `releases/<env>/<appName>.yaml` 的 `image.tag`
2. 如为配置问题，回滚对应 persistence 配置 commit
3. 由 Argo CD 执行同步恢复

禁止常态化使用 `kubectl rollout undo` 作为主回滚手段。

## 8. 验收清单

1. 配置落点符合分层规则（apps/envs/clusters/releases）
2. Helm 模板无硬编码存储参数
3. `helm template` 渲染通过
4. release 存储变更已 MR 审批
5. Jenkins 提交未触碰 persistence 字段
6. 变更后读写路径验证通过

## 9. nfs-data-provisioner 全链路打通

仓库内已支持通过 `nfs-data-provisioner` 完成 NFS/SFS 动态供给，流程如下：

1. 配置 provisioner 应用：
   - `apps/nfs-data-provisioner.yaml`
2. 按环境配置 NFS 地址与路径、StorageClass 名称：
   - `releases/dev/nfs-data-provisioner.yaml`
   - `releases/test/nfs-data-provisioner.yaml`
   - `releases/release/nfs-data-provisioner.yaml`
3. Argo CD 渲染并下发：
   - ServiceAccount
   - RBAC（Role/RoleBinding/ClusterRole/ClusterRoleBinding）
   - Deployment（nfs-subdir-external-provisioner）
   - StorageClass
4. 业务应用提交 PVC（`type: pvc`）后，provisioner 动态创建 PV 并绑定 PVC。
5. Pod 启动后完成挂载。

说明：

1. `PROVISIONER_NAME` 必须与 `storageClass.provisioner` 一致。
2. 建议每个环境使用不同 StorageClass 名称，避免跨环境冲突。
3. 若必须静态供给，可使用 `common-worker` 新增的 `persistentVolumes/persistentVolumeClaims` 模板能力。
