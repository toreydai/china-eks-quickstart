# Demo05 — Velero 备份与恢复

## 实验简介

本实验使用 Velero 为中国区 EKS 集群搭建备份与灾难恢复能力，将集群资源备份到 S3，并通过删除-恢复演练验证可恢复性，最后配置定时备份。中国区 S3 建桶需指定 `LocationConstraint`，Velero 镜像统一从集中预置的 0910 仓库拉取。

**实验目标：**
- 掌握 Velero 的安装、备份、恢复与定时备份配置
- 理解基于 Pod Identity 的 Velero S3/EC2 权限授权
- 能够完成一次「删除命名空间→从备份恢复」的灾难演练

**实验流程：**
1. 设置环境变量
2. 创建 Velero S3 Bucket 并开启版本控制
3. 创建 Velero IAM Policy 与 Pod Identity Role
4. 用 Helm 安装 Velero
5. 创建测试应用并执行备份
6. 删除命名空间模拟灾难并恢复
7. 配置每日定时备份

**预计 AI 执行时长：** 30-40 分钟

---

## 前提条件

- **工具**：AWS CLI v2、kubectl、helm
- **权限**：EKS、S3、IAM 权限
- **前提**：Demo01 已完成

---

## 步骤

### 1. 设置环境变量

复用集群环境，定义 Velero 备份桶、命名空间、Role 与版本号。

```bash
source /tmp/demo-eks.env
export VELERO_BUCKET=eks-velero-backup-${ACCOUNT_ID}
export VELERO_NS=velero
export VELERO_ROLE=eks-velero-role
export VELERO_VERSION=1.15.2
export ASSET_DIR=${ASSET_DIR:-$(pwd)/offline-assets}
```

### 2. 创建 Velero S3 Bucket

中国区创建 S3 桶必须带 `--create-bucket-configuration LocationConstraint=`，否则会报错。开启版本控制可防止备份对象被误覆盖。

```bash
aws s3api create-bucket \
  --bucket ${VELERO_BUCKET} \
  --region ${AWS_REGION} \
  --create-bucket-configuration LocationConstraint=${AWS_REGION} || true

aws s3api put-bucket-versioning \
  --bucket ${VELERO_BUCKET} \
  --versioning-configuration Status=Enabled

echo "S3 桶已创建: ${VELERO_BUCKET}"
```

**预期输出**：打印桶名

### 3. 创建 Velero IAM Policy 和 Role

Velero 需要 EC2 快照与 S3 读写权限。注意 S3 资源 ARN 使用 `arn:${AWS_PARTITION}:`（中国区为 `aws-cn`），信任主体为 `pods.eks.amazonaws.com`。

```bash
cat > /tmp/velero-policy.json <<EOF
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "ec2:DescribeVolumes","ec2:DescribeSnapshots",
        "ec2:CreateTags","ec2:CreateVolume",
        "ec2:CreateSnapshot","ec2:DeleteSnapshot"
      ],
      "Resource": "*"
    },
    {
      "Effect": "Allow",
      "Action": [
        "s3:GetObject","s3:DeleteObject","s3:PutObject",
        "s3:AbortMultipartUpload","s3:ListMultipartUploadParts"
      ],
      "Resource": "arn:${AWS_PARTITION}:s3:::${VELERO_BUCKET}/*"
    },
    {
      "Effect": "Allow",
      "Action": "s3:ListBucket",
      "Resource": "arn:${AWS_PARTITION}:s3:::${VELERO_BUCKET}"
    }
  ]
}
EOF

aws iam create-policy \
  --policy-name VeleroAccessPolicy \
  --policy-document file:///tmp/velero-policy.json || true

export VELERO_POLICY_ARN=$(aws iam list-policies \
  --query "Policies[?PolicyName=='VeleroAccessPolicy'].Arn" \
  --output text)

# 创建 Pod Identity Role
cat > /tmp/velero-trust.json <<EOF
{
  "Version": "2012-10-17",
  "Statement": [{
    "Effect": "Allow",
    "Principal": {"Service": "pods.eks.amazonaws.com"},
    "Action": ["sts:AssumeRole","sts:TagSession"]
  }]
}
EOF

aws iam create-role \
  --role-name ${VELERO_ROLE} \
  --assume-role-policy-document file:///tmp/velero-trust.json || true

aws iam attach-role-policy \
  --role-name ${VELERO_ROLE} \
  --policy-arn ${VELERO_POLICY_ARN}

export VELERO_ROLE_ARN=$(aws iam get-role \
  --role-name ${VELERO_ROLE} \
  --query 'Role.Arn' --output text)

echo "Velero Role ARN: ${VELERO_ROLE_ARN}"
```

**预期输出**：Velero Role ARN 输出

### 4. 安装 Velero

通过 Helm 安装，关键在于将主镜像和插件 initContainer 都指向集中预置的 0910 仓库（`dockerhub/velero/velero:v1.15.2` 与 `dockerhub/velero/velero-plugin-for-aws:v1.11.0`），配置 S3 backupStorageLocation，并用 `credentials.useSecret=false` 让 Velero 走 Pod Identity 而非静态密钥。

```bash
cp ${ASSET_DIR}/velero-12.0.1.tgz /tmp/velero-12.0.1.tgz

kubectl create namespace ${VELERO_NS} --dry-run=client -o yaml | kubectl apply -f -

aws eks create-pod-identity-association \
  --cluster-name ${CLUSTER_NAME} \
  --namespace ${VELERO_NS} \
  --service-account velero \
  --role-arn ${VELERO_ROLE_ARN} \
  --region ${AWS_REGION} || true

helm upgrade --install velero /tmp/velero-12.0.1.tgz \
  -n ${VELERO_NS} \
  --set upgradeCRDs=false \
  --set configuration.backupStorageLocation[0].provider=aws \
  --set configuration.backupStorageLocation[0].bucket=${VELERO_BUCKET} \
  --set configuration.backupStorageLocation[0].config.region=${AWS_REGION} \
  --set configuration.volumeSnapshotLocation[0].provider=aws \
  --set configuration.volumeSnapshotLocation[0].config.region=${AWS_REGION} \
  --set image.repository=048912060910.dkr.ecr.cn-northwest-1.amazonaws.com.cn/dockerhub/velero/velero \
  --set image.tag=v1.15.2 \
  --set initContainers[0].name=velero-plugin-for-aws \
  --set initContainers[0].image=048912060910.dkr.ecr.cn-northwest-1.amazonaws.com.cn/dockerhub/velero/velero-plugin-for-aws:v1.11.0 \
  --set initContainers[0].volumeMounts[0].mountPath=/target \
  --set initContainers[0].volumeMounts[0].name=plugins \
  --set serviceAccount.server.create=true \
  --set serviceAccount.server.name=velero \
  --set credentials.useSecret=false

kubectl -n ${VELERO_NS} rollout status deployment/velero
```


> ⚠️ **首次拉取 048912060910 镜像若出现 ImagePullBackOff（403）**：这是节点 ECR credential provider 冷启动的短暂问题，无需修改任何权限配置。等待约 30 秒后执行 `kubectl rollout restart deployment/<name> -n <ns>` 即可恢复。节点一旦缓存镜像后，同节点后续 Pod 直接走本地缓存不再触发。

**预期输出**：`deployment "velero" successfully rolled out`

### 5. 创建测试应用并备份

部署一个含 Deployment 和 ConfigMap 的测试命名空间，创建 Backup CR 并轮询其 `status.phase` 直到 Completed，确认备份对象已落到 S3。

```bash
kubectl create namespace backup-test --dry-run=client -o yaml | kubectl apply -f -
kubectl create deployment nginx-test \
  --image=048912060910.dkr.ecr.cn-northwest-1.amazonaws.com.cn/ecrpublic/docker/library/nginx:1.27-alpine \
  -n backup-test
kubectl create configmap app-config \
  --from-literal=env=demo \
  -n backup-test
kubectl -n backup-test rollout status deployment/nginx-test

# 创建备份
cat > /tmp/backup.yaml <<'EOF'
apiVersion: velero.io/v1
kind: Backup
metadata:
  name: backup-test-ns
  namespace: velero
spec:
  includedNamespaces:
    - backup-test
  storageLocation: default
  ttl: 720h
EOF

kubectl apply -f /tmp/backup.yaml

until [ "$(kubectl get backup backup-test-ns -n ${VELERO_NS} \
  -o jsonpath='{.status.phase}' 2>/dev/null)" = "Completed" ]; do
  echo "备份中..."; sleep 10
done

echo "备份完成"
kubectl get backup backup-test-ns -n ${VELERO_NS}
```

**预期输出**：备份状态为 `Completed`

### 6. 模拟灾难并恢复

删除整个命名空间模拟数据丢失，再用 Restore CR 从备份恢复。轮询 Restore 状态至 Completed 后，确认命名空间、Deployment 和 ConfigMap 都被还原。

```bash
# 删除命名空间模拟灾难
kubectl delete namespace backup-test
sleep 10
kubectl get namespace backup-test 2>/dev/null || echo "命名空间已删除"

# 执行恢复
cat > /tmp/restore.yaml <<'EOF'
apiVersion: velero.io/v1
kind: Restore
metadata:
  name: restore-test-ns
  namespace: velero
spec:
  backupName: backup-test-ns
  includedNamespaces:
    - backup-test
EOF

kubectl apply -f /tmp/restore.yaml

until [ "$(kubectl get restore restore-test-ns -n ${VELERO_NS} \
  -o jsonpath='{.status.phase}' 2>/dev/null)" = "Completed" ]; do
  echo "恢复中..."; sleep 10
done

kubectl get namespace backup-test
kubectl get all -n backup-test
kubectl get configmap app-config -n backup-test
```

**预期输出**：命名空间和资源已恢复，configmap `app-config` 存在

### 7. 配置定时备份

用 Schedule CR 配置每日凌晨 2 点的定时备份（cron `0 2 * * *`），TTL 设为 168 小时，实现自动化的周期性数据保护。

```bash
cat > /tmp/schedule.yaml <<'EOF'
apiVersion: velero.io/v1
kind: Schedule
metadata:
  name: daily-backup
  namespace: velero
spec:
  schedule: "0 2 * * *"
  template:
    includedNamespaces:
      - backup-test
    storageLocation: default
    ttl: 168h
EOF

kubectl apply -f /tmp/schedule.yaml
kubectl get schedule daily-backup -n ${VELERO_NS}
```

**预期输出**：Schedule 创建成功，显示 Cron 表达式 `0 2 * * *`

---

## 验收标准

完成本实验后，你应当能够：
- [ ] Velero 在集群中正常运行（Deployment readyReplicas 为 1）
- [ ] 成功创建一个状态为 Completed 的命名空间备份
- [ ] 删除命名空间后从备份完整恢复出 Deployment 与 ConfigMap
- [ ] 配置一个每日运行的定时备份 Schedule

---

## 验证检查点

| # | 检查命令 | 期望精确输出 |
|---|---------|-------------|
| 1 | `kubectl get deployment velero -n velero -o jsonpath='{.status.readyReplicas}'` | `1` |
| 2 | `kubectl get backup backup-test-ns -n velero -o jsonpath='{.status.phase}'` | `Completed` |
| 3 | `kubectl get restore restore-test-ns -n velero -o jsonpath='{.status.phase}'` | `Completed` |
| 4 | `aws s3 ls s3://eks-velero-backup-$(aws sts get-caller-identity --query Account --output text)/backups/ --region cn-northwest-1 \| grep backup-test-ns \| wc -l` | 大于 `0` |

---

## 实验总结

本实验用 Velero 为中国区 EKS 建立了完整的备份与灾难恢复链路：S3 后端存储、Pod Identity 授权、按需备份、删除-恢复演练以及定时备份。你掌握了中国区 S3 建桶约束和 Velero 的私有镜像安装方式。下一个 Demo（Demo06）将进入 Helm 打包与基于 ArgoCD 的 GitOps 持续部署。

---

## 清理

```bash
source /tmp/demo-eks.env

kubectl delete schedule daily-backup -n ${VELERO_NS} || true
kubectl delete restore restore-test-ns -n ${VELERO_NS} || true
kubectl delete backup backup-test-ns -n ${VELERO_NS} || true
kubectl delete namespace backup-test || true

helm uninstall velero -n ${VELERO_NS}
kubectl delete namespace ${VELERO_NS}

aws eks delete-pod-identity-association \
  --cluster-name ${CLUSTER_NAME} \
  --association-id $(aws eks list-pod-identity-associations \
    --cluster-name ${CLUSTER_NAME} \
    --namespace velero \
    --service-account velero \
    --query 'associations[0].associationId' --output text) \
  --region ${AWS_REGION} || true

aws iam detach-role-policy \
  --role-name ${VELERO_ROLE} \
  --policy-arn ${VELERO_POLICY_ARN}
aws iam delete-role --role-name ${VELERO_ROLE}
aws iam delete-policy --policy-arn ${VELERO_POLICY_ARN}

aws s3 rm s3://${VELERO_BUCKET} --recursive
aws s3api delete-bucket --bucket ${VELERO_BUCKET} --region ${AWS_REGION}
```
