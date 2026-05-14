# release-flow

## CI/CD 职责边界

- Jenkins 只负责 CI：拉代码、构建、测试、打包镜像、推送镜像、更新 GitOps release 文件。
- Argo CD 只负责 CD：监听 GitOps 变更并同步到 Kubernetes。
- Jenkins 严禁直接操作 Kubernetes。

禁止 Jenkins 执行：
- kubectl apply
- kubectl set image
- kubectl rollout restart
- kubectl delete
- kubectl patch
- helm upgrade --install
- helm rollback
- argocd app sync

## Jenkins 改造后的发布流程

1. Checkout Source
2. Build Package
3. Generate Image Tag (`<env>-<gitCommitShort>-<buildNumber>`)
4. Docker Build
5. Docker Push
6. Clone GitOps Repo
7. Update `releases/<env>/<appName>.yaml` 的 `image.tag`
8. Git Commit
9. Git Push / Merge Request（`release` 环境必须走 MR 审批）

## Jenkins 更新示例

```bash
VALUES_FILE="releases/${ENV}/${APP_NAME}.yaml"

if [ ! -f "${VALUES_FILE}" ]; then
  echo "ERROR: ${VALUES_FILE} not found"
  exit 1
fi

yq -i ".image.tag = \"${IMAGE_TAG}\"" "${VALUES_FILE}"

git config user.name "jenkins"
git config user.email "jenkins@example.com"
git add "${VALUES_FILE}"
git commit -m "deploy(${APP_NAME}): ${ENV} update image to ${IMAGE_TAG}" || echo "No changes to commit"
git push origin main
```

## release 环境策略

- Jenkins 不直接 push `main` 修改 `releases/release/*`
- Jenkins 推送 release 分支并创建 MR
- MR 审批通过后合入 main
- Argo CD 再执行同步（推荐手动）
