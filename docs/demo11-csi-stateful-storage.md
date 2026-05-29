# Demo11 — CSI 有状态存储（EBS / EFS / S3）

## 实验简介

本实验通过三种 AWS CSI 驱动为 EKS 提供有状态存储能力：EBS（块存储，ReadWriteOnce）、EFS（共享文件系统，ReadWriteMany）和 S3 Mountpoint（对象存储挂载）。三者授权方式不同——EBS 用 addon 自动配置 Pod Identity，S3 CSI 则使用节点实例 Role。中国区 EFS/S3 创建还涉及 NFS 安全组与建桶 LocationConstraint。

**实验目标：**
- 掌握 EBS/EFS/S3 三种 CSI Addon 的安装与差异
- 理解 ReadWriteOnce 与 ReadWriteMany 访问模式的适用场景
- 能够创建动态/静态 PV 并验证多 Pod 共享存储

**实验流程：**
1. 设置环境变量
2. 安装 EBS CSI Addon
3. 测试 EBS 动态卷
4. 安装 EFS CSI Addon
5. 创建 EFS 文件系统与 Mount Target
6. 测试 EFS ReadWriteMany 卷
7. 安装 S3 Mountpoint CSI Addon

**预计时长：** 30-40 分钟

---

## 前提条件

- **工具**：AWS CLI v2、kubectl、eksctl
- **权限**：EKS、EC2、EFS、S3、IAM 权限
- **前提**：Demo01 已完成
- **预计耗时**：30-40 分钟

---

## 步骤

### 1. 设置环境变量

复用集群环境，定义 EFS 名称、S3 桶名，并查询节点实例 Role（S3 CSI 依赖它授权）。

```bash
source /tmp/demo-eks.env
export EFS_NAME=eks-demo-efs
export S3_MOUNTPOINT_BUCKET=eks-s3-mount-${ACCOUNT_ID}
export NODE_ROLE=$(aws cloudformation describe-stack-resources \
  --stack-name eksctl-${CLUSTER_NAME}-nodegroup-ng-default \
  --query 'StackResources[?ResourceType==`AWS::IAM::Role`].PhysicalResourceId' \
  --output text)
```

### 2. 安装 EBS CSI Addon 并并行启动 EFS 文件系统创建

EBS CSI 驱动以托管 addon 安装，支持自动应用 Pod Identity Association 配置权限，无需手动建 Role。等待 addon ACTIVE 的同时，提前触发 EFS CSI addon 安装和 EFS 文件系统创建（约需 5 分钟），两者并行进行节省总耗时。

```bash
# 启动 EBS CSI addon 安装
aws eks create-addon \
  --cluster-name ${CLUSTER_NAME} \
  --addon-name aws-ebs-csi-driver \
  --resolve-conflicts OVERWRITE \
  --region ${AWS_REGION}

# 同时启动 EFS CSI addon 安装（并行，不等待）
aws eks create-addon \
  --cluster-name ${CLUSTER_NAME} \
  --addon-name aws-efs-csi-driver \
  --resolve-conflicts OVERWRITE \
  --region ${AWS_REGION}

# 同时开始创建 EFS 文件系统（并行，不等待）
CLUSTER_VPC=$(aws eks describe-cluster \
  --name ${CLUSTER_NAME} \
  --query 'cluster.resourcesVpcConfig.vpcId' --output text)

EFS_SG_ID=$(aws ec2 create-security-group \
  --group-name eks-efs-sg \
  --description "EFS security group for EKS" \
  --vpc-id ${CLUSTER_VPC} \
  --query 'GroupId' --output text)

NODE_SG=$(aws eks describe-cluster \
  --name ${CLUSTER_NAME} \
  --query 'cluster.resourcesVpcConfig.clusterSecurityGroupId' --output text)

aws ec2 authorize-security-group-ingress \
  --group-id ${EFS_SG_ID} \
  --protocol tcp \
  --port 2049 \
  --source-group ${NODE_SG}

EFS_ID=$(aws efs create-file-system \
  --performance-mode generalPurpose \
  --throughput-mode bursting \
  --encrypted \
  --tags Key=Name,Value=${EFS_NAME} \
  --query 'FileSystemId' --output text)

echo "EFS 文件系统 ID: ${EFS_ID}（后台创建中）"

# 现在再等待 EBS addon 就绪（EFS 创建在后台并行进行）
# EBS CSI Addon 支持 --auto-apply-pod-identity-associations，自动配置权限
aws eks wait addon-active \
  --cluster-name ${CLUSTER_NAME} \
  --addon-name aws-ebs-csi-driver \
  --region ${AWS_REGION}

echo "EBS CSI addon 已激活"
```

**预期输出**：打印"EBS CSI addon 已激活"，EFS 文件系统 ID 已输出

### 3. 测试 EBS 动态卷

创建 gp2 StorageClass 的 PVC，CSI 驱动会动态供给一块 EBS 卷并 Bound。EBS 是 ReadWriteOnce，只能被单个节点上的 Pod 挂载，适合数据库等独占型有状态应用。

```bash
kubectl create namespace ebs-demo --dry-run=client -o yaml | kubectl apply -f -

cat > /tmp/ebs-test.yaml <<'EOF'
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: ebs-pvc
  namespace: ebs-demo
spec:
  accessModes:
    - ReadWriteOnce
  storageClassName: gp2
  resources:
    requests:
      storage: 5Gi
---
apiVersion: v1
kind: Pod
metadata:
  name: ebs-writer
  namespace: ebs-demo
spec:
  containers:
    - name: writer
      image: 048912060910.dkr.ecr.cn-northwest-1.amazonaws.com.cn/ecrpublic/docker/library/busybox:1.36
      command: ["sh", "-c", "echo 'EBS persistent data' > /data/test.txt && cat /data/test.txt && sleep 3600"]
      volumeMounts:
        - name: ebs-vol
          mountPath: /data
      resources:
        limits:
          cpu: "100m"
          memory: "64Mi"
  volumes:
    - name: ebs-vol
      persistentVolumeClaim:
        claimName: ebs-pvc
EOF

kubectl apply -f /tmp/ebs-test.yaml
kubectl wait pod ebs-writer -n ebs-demo --for=condition=Ready --timeout=120s
kubectl logs ebs-writer -n ebs-demo
```

**预期输出**：输出 `EBS persistent data`，PVC 状态 `Bound`

### 4. 等待 EFS CSI Addon 就绪

EFS CSI addon 已在步骤2中并行启动，此处等待其 ACTIVE。

```bash
aws eks wait addon-active \
  --cluster-name ${CLUSTER_NAME} \
  --addon-name aws-efs-csi-driver \
  --region ${AWS_REGION}

echo "EFS CSI addon 已激活"
```

**预期输出**：打印"EFS CSI addon 已激活"

### 5. 等待 EFS 文件系统就绪并创建 Mount Target

EFS 文件系统已在步骤2中并行创建（节省约 5 分钟等待），此处轮询至 available 后为每个子网创建 Mount Target，放行节点 2049/NFS 访问。

```bash
# EFS_ID 和 EFS_SG_ID 已在步骤2中创建，此处直接使用
# 轮询等待 EFS available
until [ "$(aws efs describe-file-systems \
  --file-system-id ${EFS_ID} \
  --query 'FileSystems[0].LifeCycleState' --output text)" = "available" ]; do
  echo "等待 EFS..."; sleep 10
done

echo "EFS 文件系统 ID: ${EFS_ID}"
```

**预期输出**：EFS 文件系统 ID 输出

### 6. 测试 EFS ReadWriteMany 卷

用静态 PV 绑定 EFS 文件系统，多个 Pod 通过 ReadWriteMany 同时读写同一目录。先轮询 EFS 至 available 再部署，验证两个 Pod 能共享写入同一日志文件——这是 EBS 做不到的。

```bash
kubectl create namespace efs-demo --dry-run=client -o yaml | kubectl apply -f -

# EFS 已在步骤5中等待就绪，此处直接使用 EFS_ID
# 为每个子网创建 Mount Target（若步骤5已完成 EFS 等待则直接执行）
SUBNET_IDS=$(aws ec2 describe-subnets \
  --filters "Name=vpc-id,Values=${CLUSTER_VPC}" \
  --query 'Subnets[*].SubnetId' --output text)

for SUBNET_ID in ${SUBNET_IDS}; do
  aws efs create-mount-target \
    --file-system-id ${EFS_ID} \
    --subnet-id ${SUBNET_ID} \
    --security-groups ${EFS_SG_ID} || true
done

cat > /tmp/efs-test.yaml <<EOF
apiVersion: v1
kind: PersistentVolume
metadata:
  name: efs-pv
spec:
  capacity:
    storage: 5Gi
  volumeMode: Filesystem
  accessModes:
    - ReadWriteMany
  persistentVolumeReclaimPolicy: Retain
  storageClassName: efs-sc
  csi:
    driver: efs.csi.aws.com
    volumeHandle: ${EFS_ID}
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: efs-pvc
  namespace: efs-demo
spec:
  accessModes:
    - ReadWriteMany
  storageClassName: efs-sc
  resources:
    requests:
      storage: 5Gi
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: efs-app
  namespace: efs-demo
spec:
  replicas: 2
  selector:
    matchLabels:
      app: efs-app
  template:
    metadata:
      labels:
        app: efs-app
    spec:
      containers:
        - name: app
          image: 048912060910.dkr.ecr.cn-northwest-1.amazonaws.com.cn/ecrpublic/amazonlinux/amazonlinux:2
          command: ["sh", "-c", "while true; do date >> /data/shared.log; sleep 5; done"]
          volumeMounts:
            - name: efs-vol
              mountPath: /data
          resources:
            limits:
              cpu: "100m"
              memory: "64Mi"
      volumes:
        - name: efs-vol
          persistentVolumeClaim:
            claimName: efs-pvc
EOF

kubectl apply -f /tmp/efs-test.yaml
kubectl -n efs-demo rollout status deployment/efs-app

sleep 15
POD1=$(kubectl get pod -n efs-demo -l app=efs-app -o jsonpath='{.items[0].metadata.name}')
kubectl exec -n efs-demo ${POD1} -- cat /data/shared.log
```


> ⚠️ **首次拉取 048912060910 镜像若出现 ImagePullBackOff（403）**：这是节点 ECR credential provider 冷启动的短暂问题，无需修改任何权限配置。等待约 30 秒后执行 `kubectl rollout restart deployment/<name> -n <ns>` 即可恢复。节点一旦缓存镜像后，同节点后续 Pod 直接走本地缓存不再触发。

**预期输出**：两个 Pod 共享写入 `/data/shared.log`，日志文件持续增长

### 7. 安装 S3 Mountpoint CSI Addon

S3 CSI 驱动将 S3 桶挂载为文件系统。它使用节点实例 Role 授权（而非 Pod Identity），因此需先给节点 Role 附加 S3 权限，再安装 addon。注意中国区建桶要带 LocationConstraint，托管策略 ARN 用 `aws-cn` 分区。

```bash
# S3 CSI 使用节点实例 Role（而非 Pod Identity）
aws s3api create-bucket \
  --bucket ${S3_MOUNTPOINT_BUCKET} \
  --region ${AWS_REGION} \
  --create-bucket-configuration LocationConstraint=${AWS_REGION} || true

# 将 S3 访问权限附加到节点 Role
aws iam attach-role-policy \
  --role-name ${NODE_ROLE} \
  --policy-arn arn:${AWS_PARTITION}:iam::aws:policy/AmazonS3FullAccess

aws eks create-addon \
  --cluster-name ${CLUSTER_NAME} \
  --addon-name aws-mountpoint-s3-csi-driver \
  --resolve-conflicts OVERWRITE \
  --region ${AWS_REGION}

aws eks wait addon-active \
  --cluster-name ${CLUSTER_NAME} \
  --addon-name aws-mountpoint-s3-csi-driver \
  --region ${AWS_REGION}

echo "S3 CSI addon 已激活"
```

**预期输出**：打印"S3 CSI addon 已激活"

---

## 验收标准

完成本实验后，你应当能够：
- [ ] EBS、EFS、S3 三个 CSI Addon 均处于 ACTIVE 状态
- [ ] 动态创建的 EBS PVC 状态为 Bound 且 Pod 能持久化写入
- [ ] EFS 文件系统状态为 available 且多个 Pod 能共享读写同一卷
- [ ] 理解 EBS（RWO）、EFS（RWX）、S3 三种存储的差异与授权方式

---

## 验证检查点

| # | 检查命令 | 期望精确输出 |
|---|---------|-------------|
| 1 | `aws eks describe-addon --cluster-name demo --addon-name aws-ebs-csi-driver --region cn-northwest-1 --query 'addon.status' --output text` | `ACTIVE` |
| 2 | `aws eks describe-addon --cluster-name demo --addon-name aws-efs-csi-driver --region cn-northwest-1 --query 'addon.status' --output text` | `ACTIVE` |
| 3 | `kubectl get pvc ebs-pvc -n ebs-demo -o jsonpath='{.status.phase}'` | `Bound` |
| 4 | `aws efs describe-file-systems --creation-token eks-demo-efs --region cn-northwest-1 --query 'FileSystems[0].LifeCycleState' --output text` | `available` |

---

## 实验总结

本实验通过 EBS、EFS、S3 三种 CSI 驱动为 EKS 提供了块存储、共享文件系统和对象存储三类持久化能力，并理解了它们在访问模式（RWO vs RWX）和授权方式（Pod Identity vs 节点 Role）上的差异。中国区的 NFS 安全组、Mount Target、建桶约束是落地要点。下一个 Demo（Demo12）将聚焦身份与访问控制（Pod Identity、Access Entry、Secrets Store CSI）。

---

## 清理

```bash
source /tmp/demo-eks.env

kubectl delete namespace ebs-demo efs-demo 2>/dev/null || true
kubectl delete pv efs-pv 2>/dev/null || true

aws eks delete-addon --cluster-name ${CLUSTER_NAME} --addon-name aws-ebs-csi-driver --region ${AWS_REGION} || true
aws eks delete-addon --cluster-name ${CLUSTER_NAME} --addon-name aws-efs-csi-driver --region ${AWS_REGION} || true
aws eks delete-addon --cluster-name ${CLUSTER_NAME} --addon-name aws-mountpoint-s3-csi-driver --region ${AWS_REGION} || true

EFS_ID=$(aws efs describe-file-systems \
  --filters Name=tag:Name,Values=${EFS_NAME} \
  --query 'FileSystems[0].FileSystemId' --output text 2>/dev/null)

if [ -n "${EFS_ID}" ] && [ "${EFS_ID}" != "None" ]; then
  for MT_ID in $(aws efs describe-mount-targets \
    --file-system-id ${EFS_ID} \
    --query 'MountTargets[].MountTargetId' --output text); do
    aws efs delete-mount-target --mount-target-id ${MT_ID}
  done
  sleep 30
  aws efs delete-file-system --file-system-id ${EFS_ID}
fi

EFS_SG_ID=$(aws ec2 describe-security-groups \
  --filters "Name=group-name,Values=eks-efs-sg" \
  --query 'SecurityGroups[0].GroupId' --output text 2>/dev/null)
[ -n "${EFS_SG_ID}" ] && aws ec2 delete-security-group --group-id ${EFS_SG_ID} || true

aws iam detach-role-policy \
  --role-name ${NODE_ROLE} \
  --policy-arn arn:${AWS_PARTITION}:iam::aws:policy/AmazonS3FullAccess || true

aws s3 rm s3://${S3_MOUNTPOINT_BUCKET} --recursive || true
aws s3api delete-bucket --bucket ${S3_MOUNTPOINT_BUCKET} --region ${AWS_REGION} || true
```
